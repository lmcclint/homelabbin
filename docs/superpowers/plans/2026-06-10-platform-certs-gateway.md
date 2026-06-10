# Platform: Certs + Gateway API Implementation Plan (Plan 2b)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **For the operator:** Continues Plan 2a on the live k3s cluster. Mostly
> Helm/manifests applied via Ansible from the workstation; you supply the Porkbun
> API credentials. Commit after each task; pushing is fine (private repo).

**Goal:** Stand up cluster TLS (Let's Encrypt wildcard `*.lab.2bit.name` via Porkbun DNS-01) and a Gateway-API ingress (Traefik), proven by serving the Longhorn UI over a browser-valid HTTPS cert.

**Architecture:** cert-manager + the Talinx Porkbun DNS-01 webhook issue a wildcard `*.lab.2bit.name` Certificate (no public ports; works for RFC1918 services). Traefik runs as the Gateway API controller on a MetalLB VIP (`10.20.20.200`); a single `Gateway` terminates TLS with the wildcard cert, and `HTTPRoute`s attach per service. Everything is Helm-through-Ansible + committed manifests, consistent with Plan 2a.

**Tech Stack:** cert-manager v1.20.2, Talinx cert-manager-webhook-porkbun 1.0.0, Gateway API CRDs v1.5.1, Traefik chart 40.3.0 (Gateway provider), Longhorn UI (the test backend). `kubectl`/`helm`/`dig`/`openssl` + the existing Ansible setup.

**Source spec:** `docs/superpowers/specs/2026-06-09-k3s-cluster-dns-design.md`

**Builds on:** Plan 2a (`docs/superpowers/plans/2026-06-09-k3s-cluster-foundation.md`) — working k3s + MetalLB + Longhorn.

**Out of scope (Plan 2c):** Pi-hole, Technitium, client-DNS cutover.

---

## Conventions

- **Run from the repo root / `ansible/`** as in Plan 2a; `export KUBECONFIG=$PWD/kubeconfig` for `kubectl`.
- Versions live in `ansible/group_vars/all/versions.yml`; values below are current stable (2026-06-10) — bump there if needed.
- **"Expected" output describes success** — stop and resolve if you don't see it.
- New Ansible roles are appended to `ansible/site.yml` so a full run is reproducible.

---

## File Structure

| File | Responsibility |
|---|---|
| `ansible/group_vars/all/versions.yml` | Add cert-manager / webhook / traefik / gateway-api versions |
| `ansible/group_vars/all/vault.yml` | Add Porkbun API key + secret (ansible-vault) |
| `ansible/roles/cert-manager/tasks/main.yml` | Helm cert-manager + Talinx webhook; render porkbun-secret; apply ClusterIssuer + wildcard Certificate |
| `ansible/roles/cert-manager/templates/porkbun-secret.yaml.j2` | Secret rendered from vault vars |
| `ansible/roles/traefik/tasks/main.yml` | Apply Gateway API CRDs; Helm Traefik (Gateway provider, VIP .200); apply Gateway + HTTPRoute |
| `kubernetes/cert-manager/porkbun/cluster-issuer.yaml` | **Modify** — match Talinx groupName/secret |
| `kubernetes/cert-manager/porkbun/wildcard-certificate.yaml` | `*.lab.2bit.name` Certificate |
| `kubernetes/gateway/gateway.yaml` | `Gateway` (HTTPS listener, wildcard cert) |
| `kubernetes/gateway/httproute-longhorn.yaml` | `HTTPRoute` for the Longhorn UI (the test) |
| `docs/runbooks/k3s-cluster-asbuilt.md` | **Modify** — record platform layer |

Namespaces: cert-manager → `cert-manager`; Traefik + Gateway + wildcard cert → `traefik`.

---

## Task 0: Versions, Porkbun credentials, scaffolding

**Files:**
- Modify: `ansible/group_vars/all/versions.yml`, `ansible/group_vars/all/vault.yml`
- Create: `ansible/roles/cert-manager/templates/porkbun-secret.yaml.j2`

- [ ] **Step 1: Add platform versions**

Append to `ansible/group_vars/all/versions.yml`:
```yaml
cert_manager_chart_version: "v1.20.2"
porkbun_webhook_chart_version: "1.0.0"
gateway_api_version: "v1.5.1"
traefik_chart_version: "40.3.0"
```

- [ ] **Step 2: Add your Porkbun API credentials to the vault**

Get a Porkbun API key + secret key (porkbun.com → Account → API Access; enable
API access on the `2bit.name` domain). Add them to the encrypted vault. Run from
`ansible/`:
```bash
ansible-vault edit group_vars/all/vault.yml
```
Add these two lines (with your real values), save, and close:
```yaml
porkbun_api_key: "pk1_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
porkbun_secret_api_key: "sk1_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```
Verify they're present (values shown only inside the decrypted view):
```bash
ansible-vault view group_vars/all/vault.yml | grep -c porkbun
```
Expected: `2`.

- [ ] **Step 3: Create the Porkbun secret template**

`ansible/roles/cert-manager/templates/porkbun-secret.yaml.j2`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: porkbun-secret
  namespace: cert-manager
type: Opaque
stringData:
  PORKBUN_API_KEY: "{{ porkbun_api_key }}"
  PORKBUN_SECRET_API_KEY: "{{ porkbun_secret_api_key }}"
```

- [ ] **Step 4: Commit (no secrets — vault stays encrypted)**

```bash
cd /home/lee/git/homelabbin
git add ansible/group_vars/all/versions.yml ansible/group_vars/all/vault.yml ansible/roles/cert-manager/templates/
git commit -m "feat(platform): add cert-manager/traefik versions + porkbun vault creds"
```

---

## Task 1: cert-manager + Porkbun webhook + ClusterIssuer

**Files:**
- Create: `ansible/roles/cert-manager/tasks/main.yml`
- Modify: `kubernetes/cert-manager/porkbun/cluster-issuer.yaml`, `ansible/site.yml`

- [ ] **Step 1: Update the ClusterIssuer to match the Talinx webhook**

Replace `kubernetes/cert-manager/porkbun/cluster-issuer.yaml` with:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: le-dns-porkbun
spec:
  acme:
    email: lee@2bit.name
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: le-account-key
    solvers:
      - dns01:
          webhook:
            groupName: porkbun.talinx.dev
            solverName: porkbun
            config:
              apiKey:
                name: porkbun-secret
                key: PORKBUN_API_KEY
              secretApiKey:
                name: porkbun-secret
                key: PORKBUN_SECRET_API_KEY
```

- [ ] **Step 2: Write the cert-manager role**

`ansible/roles/cert-manager/tasks/main.yml`:
```yaml
---
- name: Install cert-manager via Helm (with CRDs)
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.helm:
    name: cert-manager
    chart_ref: cert-manager
    chart_repo_url: https://charts.jetstack.io
    chart_version: "{{ cert_manager_chart_version }}"
    release_namespace: cert-manager
    create_namespace: true
    wait: true
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"
    values:
      crds:
        enabled: true

- name: Install the Talinx Porkbun DNS-01 webhook via Helm
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.helm:
    name: cert-manager-webhook-porkbun
    chart_ref: cert-manager-webhook-porkbun
    chart_repo_url: https://talinx.github.io/cert-manager-webhook-porkbun
    chart_version: "{{ porkbun_webhook_chart_version }}"
    release_namespace: cert-manager
    wait: true
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"

- name: Render the Porkbun API secret into the cluster
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.k8s:
    state: present
    definition: "{{ lookup('template', 'porkbun-secret.yaml.j2') | from_yaml }}"
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"

- name: Apply the Porkbun ClusterIssuer
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.k8s:
    state: present
    src: "{{ playbook_dir }}/../kubernetes/cert-manager/porkbun/cluster-issuer.yaml"
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"
```

- [ ] **Step 3: Enable the role in `site.yml`**

Add `cert-manager` after `longhorn`:
```yaml
  roles:
    - node-prep
    - k3s
    - metallb
    - longhorn
    - cert-manager
    # - traefik    # enabled in Task 3
```

- [ ] **Step 4: Run it**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `failed=0`; cert-manager + webhook installed.

- [ ] **Step 5: Verify the issuer and webhook are healthy**

```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl -n cert-manager get pods
kubectl get clusterissuer le-dns-porkbun -o jsonpath='{.status.conditions[0].type}={.status.conditions[0].status}{"\n"}'
```
Expected: cert-manager, cainjector, webhook, and `cert-manager-webhook-porkbun`
pods `Running`; ClusterIssuer condition `Ready=True` (ACME account registered).

- [ ] **Step 6: Commit**

```bash
git add ansible/roles/cert-manager/tasks/ kubernetes/cert-manager/porkbun/cluster-issuer.yaml ansible/site.yml
git commit -m "feat(platform): cert-manager + Talinx Porkbun DNS-01 ClusterIssuer"
```

---

## Task 2: Wildcard `*.lab.2bit.name` certificate

**Files:**
- Create: `kubernetes/cert-manager/porkbun/wildcard-certificate.yaml`
- Modify: `ansible/roles/cert-manager/tasks/main.yml`

- [ ] **Step 1: Create the wildcard Certificate manifest**

`kubernetes/cert-manager/porkbun/wildcard-certificate.yaml` (issued into the
`traefik` namespace where the Gateway will consume it):
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: traefik
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: lab-2bit-wildcard
  namespace: traefik
spec:
  secretName: lab-2bit-wildcard-tls
  issuerRef:
    name: le-dns-porkbun
    kind: ClusterIssuer
  commonName: "*.lab.2bit.name"
  dnsNames:
    - "*.lab.2bit.name"
    - "lab.2bit.name"
```

- [ ] **Step 2: Apply it from the cert-manager role**

Append to `ansible/roles/cert-manager/tasks/main.yml`:
```yaml
- name: Apply the wildcard certificate (issued into the traefik namespace)
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.k8s:
    state: present
    src: "{{ playbook_dir }}/../kubernetes/cert-manager/porkbun/wildcard-certificate.yaml"
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"
```

- [ ] **Step 3: Run it**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `failed=0`.

- [ ] **Step 4: Verify the certificate is issued (DNS-01 round-trip)**

DNS-01 issuance creates a TXT record on Porkbun and waits for propagation — this
can take 1–3 minutes. Watch it:
```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl -n traefik get certificate lab-2bit-wildcard -w   # Ctrl-C when READY=True
```
If it stalls, inspect the order/challenge:
```bash
kubectl -n traefik describe certificate lab-2bit-wildcard | tail -20
kubectl -n traefik get challenges -o wide
```
Expected: `lab-2bit-wildcard` reaches `READY=True`; secret `lab-2bit-wildcard-tls`
exists. Confirm it's a real Let's Encrypt cert:
```bash
kubectl -n traefik get secret lab-2bit-wildcard-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer -subject -ext subjectAltName
```
Expected: issuer mentions **Let's Encrypt**, SAN includes `*.lab.2bit.name`.

- [ ] **Step 5: Commit**

```bash
git add kubernetes/cert-manager/porkbun/wildcard-certificate.yaml ansible/roles/cert-manager/tasks/
git commit -m "feat(platform): issue wildcard *.lab.2bit.name certificate"
```

---

## Task 3: Gateway API CRDs + Traefik (Gateway provider)

**Files:**
- Create: `ansible/roles/traefik/tasks/main.yml`
- Modify: `ansible/site.yml`

- [ ] **Step 1: Write the Traefik role (CRDs + Helm)**

`ansible/roles/traefik/tasks/main.yml`:
```yaml
---
- name: Install the standard Gateway API CRDs
  delegate_to: localhost
  become: false
  run_once: true
  ansible.builtin.command: >-
    kubectl --kubeconfig {{ playbook_dir }}/../kubeconfig apply -f
    https://github.com/kubernetes-sigs/gateway-api/releases/download/{{ gateway_api_version }}/standard-install.yaml
  register: gwapi_crds
  changed_when: "'created' in gwapi_crds.stdout or 'configured' in gwapi_crds.stdout"

- name: Install Traefik as the Gateway API controller via Helm
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.helm:
    name: traefik
    chart_ref: traefik
    chart_repo_url: https://traefik.github.io/charts
    chart_version: "{{ traefik_chart_version }}"
    release_namespace: traefik
    create_namespace: true
    wait: true
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"
    values:
      providers:
        kubernetesGateway:
          enabled: true
        kubernetesIngress:
          enabled: false
      gateway:
        enabled: false          # we manage our own Gateway resource
      service:
        type: LoadBalancer
        annotations:
          metallb.io/loadBalancerIPs: "10.20.20.200"
      ports:
        web:
          port: 8000
          expose:
            default: true
          exposedPort: 80
        websecure:
          port: 8443
          expose:
            default: true
          exposedPort: 443

- name: Wait for Traefik to be ready
  delegate_to: localhost
  become: false
  run_once: true
  ansible.builtin.command: >-
    kubectl --kubeconfig {{ playbook_dir }}/../kubeconfig
    -n traefik rollout status deploy/traefik --timeout=180s
  changed_when: false
```

- [ ] **Step 2: Enable the role in `site.yml`**

```yaml
  roles:
    - node-prep
    - k3s
    - metallb
    - longhorn
    - cert-manager
    - traefik
```

- [ ] **Step 3: Run it**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `failed=0`.

- [ ] **Step 4: Verify Traefik VIP + GatewayClass**

```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
kubectl get gatewayclass
```
Expected: Traefik Service `EXTERNAL-IP = 10.20.20.200`; a GatewayClass (controller
`traefik.io/gateway-controller`) with `ACCEPTED=True`.

- [ ] **Step 5: Commit**

```bash
git add ansible/roles/traefik/ ansible/site.yml
git commit -m "feat(platform): Gateway API CRDs + Traefik gateway controller on .200"
```

---

## Task 4: Gateway + Longhorn HTTPRoute (prove valid HTTPS)

**Files:**
- Create: `kubernetes/gateway/gateway.yaml`, `kubernetes/gateway/httproute-longhorn.yaml`
- Modify: `ansible/roles/traefik/tasks/main.yml`

- [ ] **Step 1: Create the Gateway**

`kubernetes/gateway/gateway.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: lab-gateway
  namespace: traefik
spec:
  gatewayClassName: traefik
  listeners:
    - name: websecure
      protocol: HTTPS
      port: 443
      hostname: "*.lab.2bit.name"
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: lab-2bit-wildcard-tls
      allowedRoutes:
        namespaces:
          from: All
```

- [ ] **Step 2: Create the Longhorn HTTPRoute**

`kubernetes/gateway/httproute-longhorn.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: longhorn-ui
  namespace: longhorn-system
spec:
  parentRefs:
    - name: lab-gateway
      namespace: traefik
  hostnames:
    - "longhorn.lab.2bit.name"
  rules:
    - backendRefs:
        - name: longhorn-frontend
          port: 80
```

- [ ] **Step 3: Apply both from the Traefik role**

Append to `ansible/roles/traefik/tasks/main.yml`:
```yaml
- name: Apply the lab Gateway and HTTPRoutes
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.k8s:
    state: present
    src: "{{ item }}"
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"
  loop:
    - "{{ playbook_dir }}/../kubernetes/gateway/gateway.yaml"
    - "{{ playbook_dir }}/../kubernetes/gateway/httproute-longhorn.yaml"
```

- [ ] **Step 4: Run it**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `failed=0`.

- [ ] **Step 5: Verify Gateway programmed + route attached**

```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl -n traefik get gateway lab-gateway -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}{"\n"}'
kubectl -n longhorn-system get httproute longhorn-ui -o jsonpath='{.status.parents[0].conditions[?(@.type=="Accepted")].status}{"\n"}'
```
Expected: Gateway `Programmed=True`; HTTPRoute `Accepted=True`.

- [ ] **Step 6: Verify browser-valid HTTPS end-to-end**

`longhorn.lab.2bit.name` isn't in DNS yet (that's Plan 2c), so resolve it manually
to the Gateway VIP for this test:
```bash
curl -sv --resolve longhorn.lab.2bit.name:443:10.20.20.200 \
  https://longhorn.lab.2bit.name/ 2>&1 | grep -iE 'SSL certificate verify|issuer:|subject:|HTTP/' | head
```
Expected: TLS verifies against a public CA (`issuer: ...Let's Encrypt...`),
`subject` matches `*.lab.2bit.name`, and an `HTTP/` status line (200/302) from the
Longhorn UI — **no cert warning**. This proves the whole DNS-01 → wildcard →
Gateway → backend path on real infra.

- [ ] **Step 7: Commit**

```bash
git add kubernetes/gateway/ ansible/roles/traefik/tasks/
git commit -m "feat(platform): lab Gateway (wildcard TLS) + Longhorn HTTPRoute"
```

---

## Task 5: Milestone — verification + docs

**Files:**
- Modify: `docs/runbooks/k3s-cluster-asbuilt.md`

- [ ] **Step 1: Idempotency check**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `failed=0`; only the benign always-churn tasks change (kubeconfig
refresh, helm idempotency note). cert-manager/traefik tasks report `ok`.

- [ ] **Step 2: Record the platform layer in the as-built**

Append to `docs/runbooks/k3s-cluster-asbuilt.md`:
```markdown

## TLS & Ingress (Plan 2b)
- cert-manager v1.20.2 (`cert-manager` ns) + Talinx Porkbun DNS-01 webhook
  (groupName `porkbun.talinx.dev`). Porkbun creds in ansible-vault, rendered to
  Secret `porkbun-secret`.
- ClusterIssuer `le-dns-porkbun` (Let's Encrypt prod). Wildcard Certificate
  `lab-2bit-wildcard` -> secret `lab-2bit-wildcard-tls` in ns `traefik`, auto-renewed.
- Gateway API CRDs v1.5.1. Traefik chart 40.3.0 as the Gateway controller, LB VIP
  `10.20.20.200`, kubernetesIngress disabled.
- `Gateway/lab-gateway` (ns traefik): HTTPS listener `*.lab.2bit.name`, terminates
  with the wildcard cert, allows routes from all namespaces.
- HTTPRoutes: `longhorn-ui` -> longhorn.lab.2bit.name (proven over valid HTTPS).
- Add per-service HTTPRoutes here as services land (pihole/dns in Plan 2c).
```

- [ ] **Step 3: Commit**

```bash
git add docs/runbooks/k3s-cluster-asbuilt.md
git commit -m "docs(runbook): record platform certs + gateway (Plan 2b complete)"
```

---

## Done criteria

- ClusterIssuer `le-dns-porkbun` is `Ready`; wildcard `*.lab.2bit.name` cert issued
  by Let's Encrypt and auto-renewing.
- Traefik runs the Gateway API on VIP `10.20.20.200`; `lab-gateway` is `Programmed`.
- `https://longhorn.lab.2bit.name` (via `--resolve` to `.200`) serves the Longhorn
  UI with a browser-valid public cert — no warnings.
- `ansible-playbook site.yml` reproduces the whole stack idempotently.
- As-built updated; committed. Plan 2c (Pi-hole + Technitium + cutover) builds next,
  adding their HTTPRoutes to this Gateway and making the names resolve.
```
