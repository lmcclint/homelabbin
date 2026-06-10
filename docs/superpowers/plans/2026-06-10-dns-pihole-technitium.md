# DNS: Pi-hole + Technitium + Cutover Implementation Plan (Plan 2c)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **For the operator:** Continues on the live cluster. DNS apps are deployed via
> Ansible/manifests; the **final cutover (Task 5) is manual UniFi UI work** you
> perform (no agent can click your UDM). Commit after each task; pushing is fine.

**Goal:** Deploy Pi-hole (filtering) + Technitium (authoritative `lab.2bit.name` + recursive) on the cluster behind MetalLB VIPs, serve their UIs over the gateway, then cut the household over to filtered split-horizon DNS — closing Plan 1's gated Task 6.

**Architecture:** Two single-replica Deployments on Longhorn PVCs. Technitium (`10.20.20.54`) is authoritative for `lab.2bit.name` (seeded via its HTTP API, including a `*.core.lab.2bit.name → 10.20.20.200` wildcard) and recursive for the internet. Pi-hole (`10.20.20.53`) filters ads then forwards everything to Technitium — it's the only resolver clients ever get. UIs are exposed as `pihole.core` / `dns.core` `.lab.2bit.name` on the existing Gateway. Cutover points each client VLAN's DHCP DNS at `.53`.

**Tech Stack:** Technitium DNS 15.2.0, Pi-hole v6 (2026.05.0), Longhorn PVCs, MetalLB VIPs, Gateway API HTTPRoutes, Ansible (`kubernetes.core`, `ansible.builtin.uri`). UniFi UI for cutover.

**Source spec:** `docs/superpowers/specs/2026-06-09-k3s-cluster-dns-design.md`

**Builds on:** Plan 2a (cluster) + Plan 2b (gateway/TLS at `*.core.lab.2bit.name`).

---

## Conventions

- Run from `ansible/` (vars/vault loaded) or repo root with `export KUBECONFIG=$PWD/kubeconfig`.
- DNS apps live in namespace **`dns`**. Image tags are pinned in `versions.yml`.
- The cluster owns **`*.core.lab.2bit.name`** (Plan 2b). Service UIs are
  `pihole.core.lab.2bit.name`, `dns.core.lab.2bit.name` — both resolve to the
  gateway VIP `10.20.20.200` via the wildcard zone record seeded in Task 2.
- **"Expected" output describes success** — stop and resolve if you don't see it.

---

## File Structure

| File | Responsibility |
|---|---|
| `ansible/group_vars/all/vault.yml` | Add `technitium_admin_password`, `pihole_admin_password` |
| `kubernetes/dns/namespace.yaml` | `dns` namespace |
| `kubernetes/dns/technitium.yaml` | Technitium PVC + Deployment + LoadBalancer Service (.54) |
| `kubernetes/dns/pihole.yaml` | Pi-hole PVC + Deployment + LoadBalancer Service (.53) |
| `kubernetes/gateway/httproute-pihole.yaml` | `pihole.core.lab.2bit.name` → pihole web |
| `kubernetes/gateway/httproute-technitium.yaml` | `dns.core.lab.2bit.name` → technitium web |
| `ansible/roles/dns/templates/technitium-admin-secret.yaml.j2` | admin pw Secret from vault |
| `ansible/roles/dns/templates/pihole-admin-secret.yaml.j2` | admin pw Secret from vault |
| `ansible/roles/dns/tasks/main.yml` | render secrets, apply manifests, seed Technitium zone via API |
| `docs/runbooks/network-foundation-asbuilt.md` | **Modify** — record cutover (Task 5) |
| `docs/runbooks/k3s-cluster-asbuilt.md` | **Modify** — record DNS services |

---

## Task 0: Admin passwords, namespace, scaffolding

> Image tags are pinned **directly in the manifests** (Tasks 1 & 3:
> `technitium/dns-server:15.2.0`, `pihole/pihole:2026.05.0`) and applied via `src:`,
> consistent with the other raw manifests in `kubernetes/`.

**Files:**
- Modify: `ansible/group_vars/all/vault.yml`
- Create: `kubernetes/dns/namespace.yaml`, both admin-secret templates

- [ ] **Step 1: Generate + store the two admin passwords in the vault**

Run from `ansible/`:
```bash
TPW=$(openssl rand -base64 24); PPW=$(openssl rand -base64 24)
ansible-vault decrypt group_vars/all/vault.yml --output /tmp/v.yml
printf 'technitium_admin_password: "%s"\npihole_admin_password: "%s"\n' "$TPW" "$PPW" >> /tmp/v.yml
ansible-vault encrypt /tmp/v.yml --output group_vars/all/vault.yml
rm -f /tmp/v.yml
echo "Technitium admin: $TPW"; echo "Pi-hole admin: $PPW"
```
Save both printed passwords to your password manager. Verify:
```bash
ansible-vault view group_vars/all/vault.yml | grep -c admin_password
```
Expected: `2`.

- [ ] **Step 2: Create the namespace manifest**

`kubernetes/dns/namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dns
```

- [ ] **Step 3: Create the admin-secret templates**

`ansible/roles/dns/templates/technitium-admin-secret.yaml.j2`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: technitium-admin
  namespace: dns
type: Opaque
stringData:
  password: "{{ technitium_admin_password }}"
```

`ansible/roles/dns/templates/pihole-admin-secret.yaml.j2`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pihole-admin
  namespace: dns
type: Opaque
stringData:
  password: "{{ pihole_admin_password }}"
```

- [ ] **Step 4: Commit**

```bash
cd /home/lee/git/homelabbin
git add ansible/group_vars/all/vault.yml kubernetes/dns/namespace.yaml ansible/roles/dns/templates/
git commit -m "feat(dns): admin passwords in vault + dns namespace scaffolding"
```

---

## Task 1: Technitium deployment (authoritative + recursive)

**Files:**
- Create: `kubernetes/dns/technitium.yaml`, `ansible/roles/dns/tasks/main.yml`
- Modify: `ansible/site.yml`

- [ ] **Step 1: Create the Technitium manifest**

`kubernetes/dns/technitium.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: technitium-config
  namespace: dns
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: technitium
  namespace: dns
spec:
  replicas: 1
  strategy:
    type: Recreate          # single RWO volume — don't run two pods at once
  selector:
    matchLabels:
      app: technitium
  template:
    metadata:
      labels:
        app: technitium
    spec:
      containers:
        - name: technitium
          image: technitium/dns-server:15.2.0
          env:
            - name: DNS_SERVER_DOMAIN
              value: dns.core.lab.2bit.name
            - name: DNS_SERVER_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: technitium-admin
                  key: password
            - name: DNS_SERVER_RECURSION
              value: AllowOnlyForPrivateNetworks
          ports:
            - { containerPort: 53, protocol: UDP }
            - { containerPort: 53, protocol: TCP }
            - { containerPort: 5380, protocol: TCP }
          volumeMounts:
            - name: config
              mountPath: /etc/dns
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: technitium-config
---
apiVersion: v1
kind: Service
metadata:
  name: technitium-dns
  namespace: dns
  annotations:
    metallb.io/loadBalancerIPs: "10.20.20.54"
spec:
  type: LoadBalancer
  selector:
    app: technitium
  ports:
    - { name: dns-udp, port: 53, targetPort: 53, protocol: UDP }
    - { name: dns-tcp, port: 53, targetPort: 53, protocol: TCP }
    - { name: web, port: 5380, targetPort: 5380, protocol: TCP }
```

> `DNS_SERVER_RECURSION=AllowOnlyForPrivateNetworks` lets the lab networks + the
> Pi-hole pod (all RFC1918) recurse, while refusing the public internet. No
> forwarders = full recursion via root hints (DNSSEC validated by default).

- [ ] **Step 2: Write the dns role (secrets + apply Technitium)**

`ansible/roles/dns/tasks/main.yml`:
```yaml
---
- name: Ensure the dns namespace exists
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.k8s:
    state: present
    src: "{{ playbook_dir }}/../kubernetes/dns/namespace.yaml"
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"

- name: Render the Technitium admin secret
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.k8s:
    state: present
    definition: "{{ lookup('template', 'technitium-admin-secret.yaml.j2') | from_yaml }}"
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"

- name: Deploy Technitium
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.k8s:
    state: present
    src: "{{ playbook_dir }}/../kubernetes/dns/technitium.yaml"
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"

- name: Wait for Technitium to be ready
  delegate_to: localhost
  become: false
  run_once: true
  ansible.builtin.command: >-
    kubectl --kubeconfig {{ playbook_dir }}/../kubeconfig
    -n dns rollout status deploy/technitium --timeout=180s
  changed_when: false
```

- [ ] **Step 3: Enable the dns role in `site.yml`**

```yaml
  roles:
    - node-prep
    - k3s
    - metallb
    - longhorn
    - cert-manager
    - traefik
    - dns
```

- [ ] **Step 4: Run it**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `failed=0`; Technitium rollout complete.

- [ ] **Step 5: Verify Technitium answers DNS + web is up**

```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl -n dns get svc technitium-dns -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'   # 10.20.20.54
dig +short example.com @10.20.20.54          # recursion works -> an IP
curl -s -o /dev/null -w "web=%{http_code}\n" http://10.20.20.54:5380/   # web up (200)
```
Expected: VIP `10.20.20.54`; `example.com` resolves (recursion); web returns `200`.

- [ ] **Step 6: Commit**

```bash
git add kubernetes/dns/technitium.yaml ansible/roles/dns/tasks/ ansible/site.yml
git commit -m "feat(dns): deploy Technitium (authoritative + recursive) on .54"
```

---

## Task 2: Seed the `lab.2bit.name` zone via the Technitium API

**Files:**
- Modify: `ansible/roles/dns/tasks/main.yml`

- [ ] **Step 1: Add the zone-seed tasks**

Append to `ansible/roles/dns/tasks/main.yml` (uses the API at the `.54` web port):
```yaml
- name: Log in to the Technitium API
  delegate_to: localhost
  become: false
  run_once: true
  ansible.builtin.uri:
    url: "http://10.20.20.54:5380/api/user/login?user=admin&pass={{ technitium_admin_password | urlencode }}&includeInfo=true"
    return_content: true
  register: tech_login
  retries: 10
  delay: 5
  until: tech_login.json is defined and tech_login.json.status == "ok"

- name: Set the API token fact
  delegate_to: localhost
  become: false
  run_once: true
  ansible.builtin.set_fact:
    tech_token: "{{ tech_login.json.token }}"

- name: Create the lab.2bit.name primary zone (idempotent)
  delegate_to: localhost
  become: false
  run_once: true
  ansible.builtin.uri:
    url: "http://10.20.20.54:5380/api/zones/create?token={{ tech_token }}&zone=lab.2bit.name&type=Primary"
    return_content: true
  register: zone_create
  failed_when: >-
    zone_create.json.status != "ok" and
    'already exists' not in (zone_create.json.errorMessage | default(''))

- name: Seed lab.2bit.name A records (overwrite = idempotent)
  delegate_to: localhost
  become: false
  run_once: true
  ansible.builtin.uri:
    url: >-
      http://10.20.20.54:5380/api/zones/records/add?token={{ tech_token }}&zone=lab.2bit.name&domain={{ item.name }}&type=A&ipAddress={{ item.ip }}&ttl=300&overwrite=true
    return_content: true
  register: rec
  failed_when: rec.json.status != "ok"
  loop:
    - { name: "*.core.lab.2bit.name", ip: "10.20.20.200" }   # gateway wildcard
    - { name: "syno.lab.2bit.name",   ip: "10.20.10.20" }
    - { name: "unas.lab.2bit.name",   ip: "192.168.1.189" }
    - { name: "fed1.lab.2bit.name",   ip: "10.20.20.11" }
    - { name: "fed2.lab.2bit.name",   ip: "10.20.20.12" }
    - { name: "fed3.lab.2bit.name",   ip: "10.20.20.13" }
```

- [ ] **Step 2: Run it**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `failed=0`; login + zone + 6 records succeed.

- [ ] **Step 3: Verify authoritative answers (incl. the wildcard)**

```bash
export KUBECONFIG=$PWD/kubeconfig
dig +short syno.lab.2bit.name           @10.20.20.54   # 10.20.10.20
dig +short fed2.lab.2bit.name           @10.20.20.54   # 10.20.20.12
dig +short longhorn.core.lab.2bit.name  @10.20.20.54   # 10.20.20.200 (via wildcard)
dig +short anything.core.lab.2bit.name  @10.20.20.54   # 10.20.20.200 (wildcard catches all)
```
Expected: each returns the IP shown. The wildcard means every `*.core` name
resolves to the gateway VIP.

- [ ] **Step 4: Commit**

```bash
git add ansible/roles/dns/tasks/main.yml
git commit -m "feat(dns): seed lab.2bit.name zone (records + *.core wildcard) via API"
```

---

## Task 3: Pi-hole deployment (filter + forward to Technitium)

**Files:**
- Create: `kubernetes/dns/pihole.yaml`
- Modify: `ansible/roles/dns/tasks/main.yml`

- [ ] **Step 1: Create the Pi-hole manifest**

`kubernetes/dns/pihole.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pihole-config
  namespace: dns
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pihole
  namespace: dns
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: pihole
  template:
    metadata:
      labels:
        app: pihole
    spec:
      containers:
        - name: pihole
          image: pihole/pihole:2026.05.0
          env:
            - name: TZ
              value: America/New_York
            - name: FTLCONF_dns_listeningMode
              value: "ALL"
            - name: FTLCONF_dns_upstreams
              value: "10.20.20.54"          # Technitium VIP (single upstream)
            - name: FTLCONF_webserver_api_password
              valueFrom:
                secretKeyRef:
                  name: pihole-admin
                  key: password
          ports:
            - { containerPort: 53, protocol: UDP }
            - { containerPort: 53, protocol: TCP }
            - { containerPort: 80, protocol: TCP }
          volumeMounts:
            - name: config
              mountPath: /etc/pihole
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: pihole-config
---
apiVersion: v1
kind: Service
metadata:
  name: pihole-dns
  namespace: dns
  annotations:
    metallb.io/loadBalancerIPs: "10.20.20.53"
spec:
  type: LoadBalancer
  selector:
    app: pihole
  ports:
    - { name: dns-udp, port: 53, targetPort: 53, protocol: UDP }
    - { name: dns-tcp, port: 53, targetPort: 53, protocol: TCP }
    - { name: web, port: 80, targetPort: 80, protocol: TCP }
```

- [ ] **Step 2: Apply Pi-hole from the dns role**

Append to `ansible/roles/dns/tasks/main.yml`:
```yaml
- name: Render the Pi-hole admin secret
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.k8s:
    state: present
    definition: "{{ lookup('template', 'pihole-admin-secret.yaml.j2') | from_yaml }}"
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"

- name: Deploy Pi-hole
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.k8s:
    state: present
    src: "{{ playbook_dir }}/../kubernetes/dns/pihole.yaml"
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"

- name: Wait for Pi-hole to be ready
  delegate_to: localhost
  become: false
  run_once: true
  ansible.builtin.command: >-
    kubectl --kubeconfig {{ playbook_dir }}/../kubeconfig
    -n dns rollout status deploy/pihole --timeout=180s
  changed_when: false
```

- [ ] **Step 3: Run it**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `failed=0`; Pi-hole rollout complete.

- [ ] **Step 4: Verify the full query chain through `.53`**

```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl -n dns get svc pihole-dns -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'  # 10.20.20.53
dig +short example.com            @10.20.20.53   # external resolves (Pi-hole->Technitium->recurse)
dig +short syno.lab.2bit.name     @10.20.20.53   # internal resolves (forwarded to Technitium) -> 10.20.10.20
dig +short longhorn.core.lab.2bit.name @10.20.20.53  # wildcard -> 10.20.20.200
dig +short doubleclick.net        @10.20.20.53   # BLOCKED -> 0.0.0.0 (Pi-hole default blocklist)
```
Expected: external + internal names resolve through `.53`; a known ad domain
returns `0.0.0.0` (or NXDOMAIN) — filtering works.

- [ ] **Step 5: Commit**

```bash
git add kubernetes/dns/pihole.yaml ansible/roles/dns/tasks/main.yml
git commit -m "feat(dns): deploy Pi-hole (filter + forward to Technitium) on .53"
```

---

## Task 4: Expose the DNS UIs on the gateway

**Files:**
- Create: `kubernetes/gateway/httproute-pihole.yaml`, `kubernetes/gateway/httproute-technitium.yaml`
- Modify: `ansible/roles/dns/tasks/main.yml`

- [ ] **Step 1: Create the HTTPRoutes**

`kubernetes/gateway/httproute-pihole.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: pihole-ui
  namespace: dns
spec:
  parentRefs:
    - name: lab-gateway
      namespace: traefik
  hostnames:
    - "pihole.core.lab.2bit.name"
  rules:
    - backendRefs:
        - name: pihole-dns
          port: 80
```

`kubernetes/gateway/httproute-technitium.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: technitium-ui
  namespace: dns
spec:
  parentRefs:
    - name: lab-gateway
      namespace: traefik
  hostnames:
    - "dns.core.lab.2bit.name"
  rules:
    - backendRefs:
        - name: technitium-dns
          port: 5380
```

- [ ] **Step 2: Apply them from the dns role**

Append to `ansible/roles/dns/tasks/main.yml`:
```yaml
- name: Apply the DNS UI HTTPRoutes
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.k8s:
    state: present
    src: "{{ item }}"
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"
  loop:
    - "{{ playbook_dir }}/../kubernetes/gateway/httproute-pihole.yaml"
    - "{{ playbook_dir }}/../kubernetes/gateway/httproute-technitium.yaml"
```

- [ ] **Step 3: Run it**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `failed=0`.

- [ ] **Step 4: Verify both UIs over valid HTTPS**

The `*.core` names already resolve via Technitium (`.54`), but your workstation
isn't using it yet (cutover is Task 5). Resolve to the gateway VIP for this test:
```bash
for h in pihole.core.lab.2bit.name dns.core.lab.2bit.name; do
  curl -s -o /dev/null -w "$h: code=%{http_code} verify=%{ssl_verify_result}\n" \
    --resolve $h:443:10.20.20.200 https://$h/
done
```
Expected: each returns an HTTP status (200/30x) with `verify=0` (browser-valid
wildcard cert). Log in later with the admin passwords from Task 0.

- [ ] **Step 5: Commit**

```bash
git add kubernetes/gateway/httproute-pihole.yaml kubernetes/gateway/httproute-technitium.yaml ansible/roles/dns/tasks/main.yml
git commit -m "feat(dns): expose pihole.core + dns.core UIs on the gateway"
```

---

## Task 5: Staged client-DNS cutover (closes Plan 1 Task 6)

**This is manual UniFi UI work.** Point each client VLAN's DHCP DNS at the Pi-hole
VIP, safest VLAN first, household last.

**Files:**
- Modify: `docs/runbooks/network-foundation-asbuilt.md`

- [ ] **Step 1: Pre-flight — all must pass before touching DHCP**

```bash
dig +short longhorn.core.lab.2bit.name @10.20.20.53   # internal/wildcard -> 10.20.20.200
dig +short syno.lab.2bit.name          @10.20.20.53   # internal -> 10.20.10.20
dig +short example.com                 @10.20.20.53   # external resolves
dig +short doubleclick.net             @10.20.20.53   # blocked -> 0.0.0.0/NXDOMAIN
```
Proceed only if all four behave.

- [ ] **Step 2: Cut over the lab VLANs first (20, then 10, 60)**

In the UniFi UI, for each of `lab-core (20)`, `lab-mgmt (10)`, `lab-cntr (60)`:
Settings → Networks → *network* → **DHCP Name Server → Manual → `10.20.20.53`**
(single entry — no secondary). Renew a client on that VLAN and verify:
```bash
# from a client on the VLAN, after DHCP renew
dig +short doubleclick.net      # blocked
dig +short example.com          # resolves
dig +short syno.lab.2bit.name   # 10.20.10.20
```

- [ ] **Step 3: Cut over the home VLAN (1) last**

Only after the lab VLANs are proven: Settings → Networks → **Default (VLAN 1)** →
DHCP Name Server → Manual → `10.20.20.53`. Renew a home device; confirm browsing
works and ads are filtered. (Nodes stay on the UDM — their DNS is pinned in
node-prep, unaffected.)

- [ ] **Step 4: Verify the household path end-to-end**

From a normal home device (after renew):
```bash
nslookup pihole.core.lab.2bit.name   # -> 10.20.20.200
nslookup doubleclick.net             # -> 0.0.0.0 (blocked)
```
Browse to `https://pihole.core.lab.2bit.name` — valid cert, Pi-hole dashboard shows
queries from across the house.

- [ ] **Step 5: Rollback note (keep handy, don't run unless needed)**

If DNS misbehaves: revert that VLAN's **DHCP Name Server back to `10.20.20.1`**
(UDM) and renew leases — one field restores resolution. The UDM Local DNS bootstrap
records still answer the critical internal names during a rollback. Emergency
household path remains the ATT guest WiFi.

- [ ] **Step 6: Record the cutover + close Plan 1 Task 6**

In `docs/runbooks/network-foundation-asbuilt.md`, update the per-VLAN DNS to
`10.20.20.53` (bypass/emergency excluded), and note Plan 1 Task 6 is complete. Bump
the `_Last updated_` line.

- [ ] **Step 7: Commit**

```bash
git add docs/runbooks/network-foundation-asbuilt.md
git commit -m "docs(runbook): client VLANs cut over to Pi-hole .53 (closes Plan 1 Task 6)"
```

---

## Task 6: Milestone — verification + docs

**Files:**
- Modify: `docs/runbooks/k3s-cluster-asbuilt.md`

- [ ] **Step 1: Idempotency check**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `failed=0`; DNS apply/seed tasks report `ok` (records overwrite cleanly),
only the benign kubeconfig/CRD churn shows changed.

- [ ] **Step 2: Record the DNS layer in the as-built**

Append to `docs/runbooks/k3s-cluster-asbuilt.md`:
```markdown

## DNS services (Plan 2c)
- Namespace `dns`. Single-replica Deployments on Longhorn PVCs (Recreate strategy).
- Technitium 15.2.0 @ VIP 10.20.20.54 (53 udp/tcp + 5380 web). Authoritative for
  lab.2bit.name (Primary zone), recursive (AllowOnlyForPrivateNetworks, root hints).
  Zone seeded via API: *.core.lab.2bit.name->.200 (gateway wildcard), syno, unas,
  fed1-3. UI: dns.core.lab.2bit.name.
- Pi-hole v6 (2026.05.0) @ VIP 10.20.20.53 (53 udp/tcp + 80 web). Filters then
  forwards to Technitium (.54) as sole upstream; listeningMode ALL. UI:
  pihole.core.lab.2bit.name.
- Admin passwords in ansible-vault (technitium-admin / pihole-admin secrets).
- Client VLANs (1/10/20/60) DHCP DNS = 10.20.20.53. Nodes still resolve via UDM.
- Single-HA-VIP model: no non-filtering secondary (avoids ad leakage). MetalLB VIP
  failover + Longhorn give single-node-failure recovery (brief blip on reschedule).
```

- [ ] **Step 3: Commit**

```bash
git add docs/runbooks/k3s-cluster-asbuilt.md
git commit -m "docs(runbook): record Pi-hole + Technitium DNS (Plan 2c complete)"
```

---

## Done criteria

- Technitium authoritative for `lab.2bit.name` (incl. `*.core` wildcard) + recursive,
  at `10.20.20.54`.
- Pi-hole at `10.20.20.53` filters ads and forwards to Technitium; the full chain
  (external, internal, blocked) verified.
- `pihole.core` / `dns.core` `.lab.2bit.name` UIs served over the wildcard cert.
- All client VLANs handed `10.20.20.53`; household gets filtered split-horizon DNS;
  **Plan 1 Task 6 closed**.
- `ansible-playbook site.yml` reproduces the whole stack (2a+2b+2c) idempotently.
- Both as-built runbooks updated; committed.
```
