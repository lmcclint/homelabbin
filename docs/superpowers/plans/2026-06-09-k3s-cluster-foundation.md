# k3s Cluster Foundation Implementation Plan (Plan 2a)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **For the operator:** This plan is **semi-automated** — most work is Ansible run
> from your workstation against `fed1/2/3`. Some steps need you (SSH access, the
> ansible-vault password). Work top-to-bottom; each task ends with a verification
> you must see pass and a commit. **Commit after each task; do not push** unless you
> intend to (repo is private now, so pushing is fine).

**Goal:** Stand up a 3-node HA k3s cluster on the existing Fedora Beelinks with MetalLB load-balancing and Longhorn replicated storage, fully provisioned by an idempotent Ansible playbook.

**Architecture:** Ansible (run from the workstation) preps each node (swap off, dedicated disks, SELinux, firewalld) then installs k3s with embedded-etcd HA (all three nodes are servers). MetalLB (L2) and Longhorn are then installed via Helm-through-Ansible. The result is a self-contained core cluster with stable VIPs and replicated PVCs.

**Tech Stack:** Ansible (`kubernetes.core`, `community.general`, `ansible.posix`), k3s (embedded etcd), Helm, MetalLB (L2/ARP), Longhorn. Fedora hosts. `kubectl`/`helm`/`dig` for verification.

**Source spec:** `docs/superpowers/specs/2026-06-09-k3s-cluster-dns-design.md`

**Out of scope (Plan 2b):** cert-manager, Traefik/Gateway API, Pi-hole, Technitium, client-DNS cutover.

---

## Conventions

- **Run Ansible from your workstation** (the WSL host), not on the nodes. Nodes are
  reached over SSH as user `lee` with the existing `~/.ssh/id_ed25519` key.
- **Node IPs (from Plan 1):** `fed1=10.20.20.11`, `fed2=10.20.20.12`, `fed3=10.20.20.13`.
- **"Expected" output describes success** — if you don't see it, stop and resolve
  before continuing.
- **Versions are pinned as variables** in `group_vars/all/versions.yml`. The values
  below are concrete known-good versions; confirm they're still current stable before
  the first run and bump the variable if needed (one place to change).

---

## File Structure

| File | Responsibility |
|---|---|
| `ansible/ansible.cfg` | Inventory path, SSH user/key, become settings |
| `ansible/inventory/hosts.yml` | The three nodes + connection vars |
| `ansible/group_vars/all/main.yml` | Non-secret cluster vars (subnets, disk sizes, pools) |
| `ansible/group_vars/all/versions.yml` | Pinned k3s/chart versions |
| `ansible/group_vars/all/vault.yml` | **ansible-vault-encrypted** secrets (k3s token) |
| `ansible/roles/node-prep/tasks/main.yml` | Swap off, disks, SELinux, firewalld, node DNS |
| `ansible/roles/k3s/tasks/main.yml` | cluster-init on fed1, join fed2/3 as servers, fetch kubeconfig |
| `ansible/roles/metallb/tasks/main.yml` | Helm install MetalLB + apply pools |
| `ansible/roles/metallb/files/pools.yaml` | `IPAddressPool` + `L2Advertisement` |
| `ansible/roles/longhorn/tasks/main.yml` | Helm install Longhorn + set default StorageClass |
| `ansible/site.yml` | Orchestrates the roles in order |
| `.gitignore` | Ensure kubeconfig + vault password excluded |

---

## Task 0: Repo scaffolding, inventory, and ansible-vault

**Files:**
- Create: `ansible/ansible.cfg`, `ansible/inventory/hosts.yml`,
  `ansible/group_vars/all/main.yml`, `ansible/group_vars/all/versions.yml`,
  `ansible/site.yml`
- Modify: `.gitignore`

- [ ] **Step 1: Confirm Ansible + collections are installed**

Run:
```bash
ansible --version
ansible-galaxy collection install kubernetes.core community.general ansible.posix
```
Expected: Ansible ≥ 2.15 reported; collections install without error. (If `ansible`
is missing: `sudo dnf install ansible` / `pipx install ansible` / `sudo apt install ansible`.)

- [ ] **Step 2: Confirm passwordless SSH + sudo to all three nodes**

Run:
```bash
for n in 11 12 13; do ssh -o BatchMode=yes lee@10.20.20.$n 'hostname; sudo -n true && echo SUDO_OK'; done
```
Expected: each prints its hostname (`fed1`/`fed2`/`fed3`) and `SUDO_OK`. If sudo
prompts for a password, either configure NOPASSWD sudo for `lee` on the nodes, or
plan to pass `--ask-become-pass` to every `ansible-playbook` call below.

- [ ] **Step 3: Create `ansible/ansible.cfg`**

```ini
[defaults]
inventory = inventory/hosts.yml
remote_user = lee
private_key_file = ~/.ssh/id_ed25519
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml

[privilege_escalation]
become = True
become_method = sudo
```

- [ ] **Step 4: Create `ansible/inventory/hosts.yml`**

```yaml
all:
  children:
    k3s_cluster:
      hosts:
        fed1:
          ansible_host: 10.20.20.11
          k3s_role: server-init
        fed2:
          ansible_host: 10.20.20.12
          k3s_role: server
        fed3:
          ansible_host: 10.20.20.13
          k3s_role: server
```

- [ ] **Step 5: Create `ansible/group_vars/all/main.yml`**

```yaml
# Cluster networking
lab_core_subnet: "10.20.20.0/24"
udm_gateway: "10.20.20.1"          # nodes resolve here (no circular DNS dep)
k3s_api_endpoint: "10.20.20.11"    # fed1, the cluster-init server

# MetalLB address pools (VLAN 20)
metallb_dns_pool: "10.20.20.53-10.20.20.54"
metallb_general_pool: "10.20.20.200-10.20.20.254"

# Disk layout
k3s_data_vg: "fedora"              # existing VG on the NVMe
k3s_data_lv: "k3s-data"
k3s_data_lv_size: "200g"           # carved from the ~460G VG free space
k3s_data_mount: "/var/lib/rancher"
longhorn_disk_dev: "/dev/sda"      # the empty 954G SSD
longhorn_mount: "/var/lib/longhorn"
```

- [ ] **Step 6: Create `ansible/group_vars/all/versions.yml`**

```yaml
# Pinned versions — confirm current stable before first run, bump here if needed.
k3s_version: "v1.31.5+k3s1"
metallb_chart_version: "0.14.9"
longhorn_chart_version: "1.7.2"
```

- [ ] **Step 7: Create the ansible-vault secrets file**

Generate a k3s join token and store it encrypted. Run:
```bash
cd ansible
echo "k3s_token: '$(openssl rand -hex 32)'" > /tmp/vault.yml
ansible-vault encrypt /tmp/vault.yml --output group_vars/all/vault.yml
rm -f /tmp/vault.yml
```
Expected: prompts for a New Vault password (store it in your password manager —
**never commit it**); `group_vars/all/vault.yml` is created and begins with
`$ANSIBLE_VAULT;1.1;AES256`.

- [ ] **Step 8: Create a minimal `ansible/site.yml` (roles added per task)**

```yaml
---
- name: k3s cluster foundation
  hosts: k3s_cluster
  gather_facts: true
  roles:
    - node-prep
    # - k3s        # enabled in Task 2
    # - metallb    # enabled in Task 3
    # - longhorn   # enabled in Task 4
```

- [ ] **Step 9: Ensure secrets are gitignored**

Confirm `.gitignore` excludes kubeconfig and any vault password file. Append if missing:
```
# k3s / ansible
kubeconfig
*.kubeconfig
.vault-pass
ansible/.vault-pass
```
(The encrypted `vault.yml` is safe to commit; the **password** is not.)

- [ ] **Step 10: Commit**

```bash
git add ansible/ .gitignore
git commit -m "feat(ansible): scaffold k3s cluster inventory, vars, and vault"
```

---

## Task 1: Node-prep role (swap, disks, SELinux, firewalld, DNS)

Idempotent host preparation so a rebuilt node re-preps identically.

**Files:**
- Create: `ansible/roles/node-prep/tasks/main.yml`

- [ ] **Step 1: Write the node-prep tasks**

`ansible/roles/node-prep/tasks/main.yml`:
```yaml
---
- name: Disable firewalld (flat trusted lab)
  ansible.builtin.systemd:
    name: firewalld
    state: stopped
    enabled: false
  failed_when: false

- name: Ensure SELinux is enforcing (targeted)
  ansible.posix.selinux:
    policy: targeted
    state: enforcing

- name: Remove zram swap config (kubelet wants swap off)
  ansible.builtin.dnf:
    name: zram-generator-defaults
    state: absent

- name: Turn off all swap now
  ansible.builtin.command: swapoff -a
  changed_when: false

- name: Install Longhorn node prerequisites (iscsi + nfs client)
  ansible.builtin.dnf:
    name:
      - iscsi-initiator-utils
      - nfs-utils
    state: present

- name: Enable and start iscsid (required by Longhorn)
  ansible.builtin.systemd:
    name: iscsid
    state: started
    enabled: true

- name: Create k3s-data logical volume on the NVMe VG
  community.general.lvol:
    vg: "{{ k3s_data_vg }}"
    lv: "{{ k3s_data_lv }}"
    size: "{{ k3s_data_lv_size }}"
    state: present

- name: Format k3s-data as xfs
  community.general.filesystem:
    fstype: xfs
    dev: "/dev/{{ k3s_data_vg }}/{{ k3s_data_lv }}"

- name: Mount k3s data dir ({{ k3s_data_mount }})
  ansible.posix.mount:
    path: "{{ k3s_data_mount }}"
    src: "/dev/{{ k3s_data_vg }}/{{ k3s_data_lv }}"
    fstype: xfs
    state: mounted

- name: Format the dedicated SSD ({{ longhorn_disk_dev }}) as xfs for Longhorn
  community.general.filesystem:
    fstype: xfs
    dev: "{{ longhorn_disk_dev }}"
    # force defaults to false -> will NOT reformat if a filesystem already exists

- name: Mount Longhorn data dir ({{ longhorn_mount }})
  ansible.posix.mount:
    path: "{{ longhorn_mount }}"
    src: "{{ longhorn_disk_dev }}"
    fstype: xfs
    state: mounted

- name: Discover the primary NetworkManager connection
  ansible.builtin.command: "nmcli -t -g GENERAL.CONNECTION device show {{ ansible_default_ipv4.interface }}"
  register: nm_conn
  changed_when: false

- name: Pin node DNS to the UDM and ignore DHCP-provided DNS
  ansible.builtin.command: >-
    nmcli connection modify "{{ nm_conn.stdout }}"
    ipv4.ignore-auto-dns yes ipv4.dns "{{ udm_gateway }}"
  register: nm_dns
  changed_when: true

- name: Reapply the connection so DNS takes effect
  ansible.builtin.command: 'nmcli connection up "{{ nm_conn.stdout }}"'
  when: nm_dns is changed
  changed_when: false
```

> Why pin node DNS explicitly: after Plan 2b's cutover, VLAN 20's DHCP hands out the
> Pi-hole VIP (`.53`). If nodes used that, the cluster would depend on the DNS it
> hosts (circular). Forcing node resolvers to the UDM breaks that loop.

- [ ] **Step 2: Run node-prep (check mode first)**

Run from `ansible/`:
```bash
ansible-playbook site.yml --check --diff
```
Expected: a dry-run diff of the prep changes, no errors. (Add `--ask-become-pass` if
sudo needs a password.)

- [ ] **Step 3: Apply node-prep for real**

Run:
```bash
ansible-playbook site.yml
```
Expected: `PLAY RECAP` shows `fed1/fed2/fed3` with `failed=0`.

- [ ] **Step 4: Verify prep on every node**

Run:
```bash
ansible k3s_cluster -a 'bash -lc "swapon --show; findmnt /var/lib/rancher; findmnt /var/lib/longhorn; getenforce; systemctl is-active firewalld; cat /etc/resolv.conf"'
```
Expected per node: **no swap** listed; `/var/lib/rancher` mounted on the LV;
`/var/lib/longhorn` mounted on `/dev/sda`; `Enforcing`; firewalld `inactive`;
`resolv.conf` shows `nameserver 10.20.20.1`.

- [ ] **Step 5: Commit**

```bash
git add ansible/roles/node-prep/
git commit -m "feat(ansible): node-prep role (swap, disks, selinux, firewalld, dns)"
```

---

## Task 2: k3s HA install (embedded etcd, all servers)

**Files:**
- Create: `ansible/roles/k3s/tasks/main.yml`
- Modify: `ansible/site.yml`

- [ ] **Step 1: Write the k3s install tasks**

`ansible/roles/k3s/tasks/main.yml`:
```yaml
---
- name: Initialise the first server (fed1)
  when: k3s_role == "server-init"
  ansible.builtin.shell: |
    curl -sfL https://get.k3s.io | \
      INSTALL_K3S_VERSION="{{ k3s_version }}" \
      K3S_TOKEN="{{ k3s_token }}" \
      sh -s - server --cluster-init \
        --disable=traefik,servicelb \
        --write-kubeconfig-mode=644 \
        --tls-san={{ k3s_api_endpoint }}
  args:
    creates: /etc/rancher/k3s/k3s.yaml

- name: Wait for the API server on fed1 to be ready
  when: k3s_role == "server-init"
  ansible.builtin.command: k3s kubectl get --raw='/readyz'
  register: readyz
  until: readyz.rc == 0
  retries: 30
  delay: 10
  changed_when: false

- name: Join the remaining servers (fed2, fed3)
  when: k3s_role == "server"
  ansible.builtin.shell: |
    curl -sfL https://get.k3s.io | \
      INSTALL_K3S_VERSION="{{ k3s_version }}" \
      K3S_TOKEN="{{ k3s_token }}" \
      sh -s - server --server https://{{ k3s_api_endpoint }}:6443 \
        --disable=traefik,servicelb \
        --write-kubeconfig-mode=644
  args:
    creates: /etc/rancher/k3s/k3s.yaml

- name: Fetch kubeconfig from fed1 to the workstation
  when: k3s_role == "server-init"
  ansible.builtin.fetch:
    src: /etc/rancher/k3s/k3s.yaml
    dest: "{{ playbook_dir }}/../kubeconfig"
    flat: true

- name: Rewrite kubeconfig server address to fed1's IP
  when: k3s_role == "server-init"
  delegate_to: localhost
  become: false
  ansible.builtin.replace:
    path: "{{ playbook_dir }}/../kubeconfig"
    regexp: 'https://127\.0\.0\.1:6443'
    replace: "https://{{ k3s_api_endpoint }}:6443"
```

- [ ] **Step 2: Enable the k3s role in `site.yml`**

Edit `ansible/site.yml` — uncomment the k3s role:
```yaml
  roles:
    - node-prep
    - k3s
    # - metallb    # enabled in Task 3
    # - longhorn   # enabled in Task 4
```

- [ ] **Step 3: Run the install**

Run from `ansible/`:
```bash
ansible-playbook site.yml
```
Expected: `failed=0`; a `kubeconfig` file appears at the repo root
(`../kubeconfig`).

- [ ] **Step 4: Point kubectl at the new cluster and verify HA**

Run from the repo root:
```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl get nodes -o wide
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" etcd="}{.metadata.labels.node-role\.kubernetes\.io/etcd}{"\n"}{end}'
```
Expected: three nodes `fed1/fed2/fed3` all `Ready`, all with `etcd=true` (all three
are etcd members = HA quorum).

- [ ] **Step 5: Confirm bundled Traefik/ServiceLB are absent**

Run:
```bash
kubectl get pods -A | grep -Ei 'traefik|svclb' || echo "none (expected)"
```
Expected: `none (expected)` — neither runs (we disabled them).

- [ ] **Step 6: Commit**

```bash
git add ansible/roles/k3s/ ansible/site.yml
git commit -m "feat(ansible): k3s HA install (embedded etcd, 3 servers)"
```

---

## Task 3: MetalLB (L2) with DNS + general pools

**Files:**
- Create: `ansible/roles/metallb/tasks/main.yml`, `ansible/roles/metallb/files/pools.yaml`
- Modify: `ansible/site.yml`

- [ ] **Step 1: Write the MetalLB pools manifest**

`ansible/roles/metallb/files/pools.yaml`:
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: dns-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.20.20.53-10.20.20.54
  autoAssign: false        # .53/.54 assigned only by explicit annotation
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: general-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.20.20.200-10.20.20.254
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-all
  namespace: metallb-system
spec:
  ipAddressPools:
    - dns-pool
    - general-pool
```

- [ ] **Step 2: Write the MetalLB install tasks**

`ansible/roles/metallb/tasks/main.yml`:
```yaml
---
- name: Install MetalLB via Helm
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.helm:
    name: metallb
    chart_ref: metallb
    chart_repo_url: https://metallb.github.io/metallb
    chart_version: "{{ metallb_chart_version }}"
    release_namespace: metallb-system
    create_namespace: true
    wait: true
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"

- name: Wait for the MetalLB webhook (controller) to be ready
  delegate_to: localhost
  become: false
  run_once: true
  ansible.builtin.command: >-
    kubectl --kubeconfig {{ playbook_dir }}/../kubeconfig
    -n metallb-system rollout status deploy/metallb-controller --timeout=120s
  changed_when: false

- name: Apply MetalLB address pools and L2 advertisement
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.k8s:
    state: present
    src: "{{ role_path }}/files/pools.yaml"
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"
```

- [ ] **Step 3: Enable the metallb role in `site.yml`**

```yaml
  roles:
    - node-prep
    - k3s
    - metallb
    # - longhorn   # enabled in Task 4
```

- [ ] **Step 4: Run it**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `failed=0`; MetalLB pods running.

- [ ] **Step 5: Verify pools and a real VIP assignment**

Run from repo root (`KUBECONFIG` still exported):
```bash
kubectl -n metallb-system get ipaddresspools
kubectl create deploy vip-test --image=traefik/whoami
kubectl expose deploy vip-test --type=LoadBalancer --port=80
kubectl get svc vip-test -w   # Ctrl-C once EXTERNAL-IP is populated
```
Expected: two pools listed; `vip-test` gets an `EXTERNAL-IP` from the **general**
pool (`10.20.20.200-254`). Then confirm it answers and clean up:
```bash
IP=$(kubectl get svc vip-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s --max-time 5 http://$IP/ | grep -i hostname && echo "VIP OK"
kubectl delete svc vip-test && kubectl delete deploy vip-test
```
Expected: `VIP OK` (the VIP is reachable from your workstation via the routed lab
network), then clean deletion.

- [ ] **Step 6: Commit**

```bash
git add ansible/roles/metallb/ ansible/site.yml
git commit -m "feat(ansible): MetalLB L2 with dns and general pools"
```

---

## Task 4: Longhorn replicated storage

**Files:**
- Create: `ansible/roles/longhorn/tasks/main.yml`
- Modify: `ansible/site.yml`

- [ ] **Step 1: Verify Longhorn's node prerequisites (installed by node-prep)**

The `iscsi`/`nfs` client tooling is installed by the node-prep role (Task 1). Confirm:
```bash
ansible k3s_cluster -a 'bash -lc "rpm -q iscsi-initiator-utils nfs-utils; systemctl is-active iscsid"'
```
Expected: both packages report a version (not "not installed"), and `iscsid` is
`active` on all three nodes.

- [ ] **Step 2: Write the Longhorn install tasks**

`ansible/roles/longhorn/tasks/main.yml`:
```yaml
---
- name: Install Longhorn via Helm
  delegate_to: localhost
  become: false
  run_once: true
  kubernetes.core.helm:
    name: longhorn
    chart_ref: longhorn
    chart_repo_url: https://charts.longhorn.io
    chart_version: "{{ longhorn_chart_version }}"
    release_namespace: longhorn-system
    create_namespace: true
    wait: true
    timeout: 10m0s
    kubeconfig: "{{ playbook_dir }}/../kubeconfig"
    values:
      defaultSettings:
        defaultDataPath: "{{ longhorn_mount }}"   # the /dev/sda mount
        defaultReplicaCount: 2
      persistence:
        defaultClass: true
        defaultClassReplicaCount: 2

- name: Wait for Longhorn manager to be ready on all nodes
  delegate_to: localhost
  become: false
  run_once: true
  ansible.builtin.command: >-
    kubectl --kubeconfig {{ playbook_dir }}/../kubeconfig
    -n longhorn-system rollout status daemonset/longhorn-manager --timeout=300s
  changed_when: false
```

- [ ] **Step 3: Enable the longhorn role in `site.yml`**

```yaml
  roles:
    - node-prep
    - k3s
    - metallb
    - longhorn
```

- [ ] **Step 4: Run it**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `failed=0`; Longhorn install completes (can take several minutes).

- [ ] **Step 5: Verify default StorageClass and a bound, replicated PVC**

Run from repo root:
```bash
kubectl get storageclass
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lh-test
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 1Gi } }
EOF
kubectl get pvc lh-test -w   # Ctrl-C once STATUS is Bound
```
Expected: `longhorn (default)` StorageClass present; `lh-test` reaches `Bound`.
Then confirm replica count = 2 and clean up:
```bash
VOL=$(kubectl get pvc lh-test -o jsonpath='{.spec.volumeName}')
kubectl -n longhorn-system get replicas.longhorn.io -l longhornvolume=$VOL --no-headers | wc -l
kubectl delete pvc lh-test
```
Expected: replica count prints `2`; PVC deletes cleanly.

- [ ] **Step 6: Commit**

```bash
git add ansible/roles/longhorn/ ansible/site.yml
git commit -m "feat(ansible): Longhorn replicated storage (replica 2, default SC)"
```

---

## Task 5: Foundation milestone — full verification + docs

**Files:**
- Create: `docs/runbooks/k3s-cluster-asbuilt.md`

- [ ] **Step 1: Run the whole playbook once more to prove idempotency**

```bash
cd ansible && ansible-playbook site.yml && cd ..
```
Expected: `PLAY RECAP` shows `changed=0` (or near-zero) — re-running is a no-op,
confirming the playbook is idempotent.

- [ ] **Step 2: Capture the cluster as-built record**

Create `docs/runbooks/k3s-cluster-asbuilt.md`:
```markdown
# k3s Cluster — As-Built Record

_Last updated: 2026-06-09 (Plan 2a complete)_

## Cluster
- k3s HA, embedded etcd. All 3 nodes are servers (control-plane + etcd + schedulable).
- Nodes: fed1 10.20.20.11, fed2 10.20.20.12, fed3 10.20.20.13.
- Bundled traefik + servicelb disabled. Node resolvers pinned to UDM 10.20.20.1.
- kubeconfig at repo root (gitignored); API at https://10.20.20.11:6443.

## Disks (per node)
- NVMe VG `fedora` LV `k3s-data` (200G, xfs) -> /var/lib/rancher (k3s data-dir).
- /dev/sda (954G, xfs) -> /var/lib/longhorn (Longhorn data path).

## Load balancing — MetalLB (L2)
- dns-pool 10.20.20.53-54 (autoAssign false; by annotation only).
- general-pool 10.20.20.200-254 (autoAssign true).

## Storage — Longhorn
- Default StorageClass `longhorn`, replica count 2, data on /var/lib/longhorn.

## Install
- Ansible: ansible/site.yml (roles: node-prep, k3s, metallb, longhorn).
- Secrets: ansible-vault group_vars/all/vault.yml (k3s token). Vault pass not committed.
```

- [ ] **Step 3: Final health snapshot**

```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl get nodes
kubectl get pods -A
kubectl -n metallb-system get ipaddresspools
kubectl get storageclass
```
Expected: 3 `Ready` nodes; all system pods `Running`; both pools present; `longhorn`
default StorageClass. This is the green light for Plan 2b.

- [ ] **Step 4: Commit**

```bash
git add docs/runbooks/k3s-cluster-asbuilt.md
git commit -m "docs(runbook): k3s cluster foundation as-built (Plan 2a complete)"
```

---

## Done criteria

- `ansible-playbook site.yml` builds the whole foundation from scratch and is
  idempotent on re-run.
- 3 nodes `Ready`, all etcd members (HA quorum survives one node down).
- MetalLB hands out VIPs from both pools; a test LoadBalancer is reachable.
- Longhorn is the default StorageClass; a test PVC binds with 2 replicas.
- `docs/runbooks/k3s-cluster-asbuilt.md` records the result; everything committed.
- Plan 2b (cert-manager, Traefik/Gateway API, Pi-hole, Technitium, cutover) is
  written against this real cluster next.
```
