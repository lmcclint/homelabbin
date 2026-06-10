# k3s Cluster — As-Built Record

_Last updated: 2026-06-10 (Plan 2a complete)_

## Cluster
- k3s **v1.36.1+k3s1**, HA with embedded etcd. All 3 nodes are servers
  (control-plane + etcd + schedulable) — quorum survives one node down.
- Nodes: `fed1` 10.20.20.11, `fed2` 10.20.20.12, `fed3` 10.20.20.13
  (Fedora 42 Server, kernel 6.19.x).
- Bundled `traefik` + `servicelb` disabled (`--disable=traefik,servicelb`).
- kubeconfig fetched to repo root `./kubeconfig` (gitignored); API at
  `https://10.20.20.11:6443` (tls-san set).

## DNS on nodes
- Nodes run **systemd-resolved** over the tagged VLAN link `enp1s0.20`, which had
  **no upstream DNS**. node-prep pins it to the **UDM (10.20.20.1)** and ignores
  DHCP DNS — avoids depending on the cluster's own DNS (no circular dep). Verify
  with `resolvectl status enp1s0.20`, not `/etc/resolv.conf` (it's the 127.0.0.53
  stub).

## Disks (per node)
- OS volume-group name differs per node (`fed1`/`fed2` = `fedora`,
  `fed3` = `fedora_fed3`) — node-prep auto-detects it.
- NVMe VG: LV `k3s-data` (200G, xfs) -> `/var/lib/rancher` (k3s data-dir; keeps
  images off the 15G root).
- `/dev/sda` (954G, xfs) -> `/var/lib/longhorn` (Longhorn data path).
- zram swap disabled; SELinux enforcing (+ `python3-libselinux`); firewalld off.

## Load balancing — MetalLB (L2)
- `metallb` chart 0.16.1, `metallb-system` namespace.
- Pools: `dns-pool` 10.20.20.53-54 (autoAssign **false**, by annotation only);
  `general-pool` 10.20.20.200-254 (autoAssign true). One `L2Advertisement` (l2-all).
- Speakers on all 3 nodes. Note: first contact to a freshly-announced VIP can drop
  while the UDM ARP-resolves it (normal L2 behavior); clients retry.

## Storage — Longhorn
- `longhorn` chart 1.12.0, `longhorn-system` namespace.
- Default StorageClass `longhorn`, **replica count 2**, data on `/var/lib/longhorn`.
- k3s `local-path` kept available but **unset as default** (longhorn is sole default).

## Install
- Ansible: `ansible/site.yml` (roles: node-prep, k3s, metallb, longhorn).
  Idempotent (re-run is a no-op aside from a benign kubeconfig refresh).
- Secrets: ansible-vault `group_vars/all/vault.yml` (k3s token). Vault password in
  `ansible/.vault-pass` (gitignored) + password manager.
- Workstation deps: `helm`, python `kubernetes` lib (for `kubernetes.core`).

## TLS & Ingress (Plan 2b)
- cert-manager v1.20.2 (`cert-manager` ns) + Talinx Porkbun DNS-01 webhook
  (groupName `porkbun.talinx.dev`). Porkbun creds in ansible-vault, rendered to
  Secret `porkbun-secret`.
- ClusterIssuer `le-dns-porkbun` (Let's Encrypt prod). Wildcard Certificate
  `core-lab-wildcard` -> secret `core-lab-wildcard-tls` in ns `traefik`, auto-renewed
  (`CN=*.core.lab.2bit.name`, SAN `*.core.lab.2bit.name` + `core.lab.2bit.name`).
- **Subdomain scoping:** this cluster owns `*.core.lab.2bit.name` (its own
  per-cluster wildcard), NOT flat `*.lab.2bit.name` — the flat namespace stays free
  for the Synology/Caddy plane (which serves `git.lab.2bit.name` etc. with its own
  Porkbun wildcard). Avoids two ingress planes fighting over one wildcard + lets us
  use a single `*.core.lab.2bit.name -> 10.20.20.200` DNS record cleanly.
- Gateway API CRDs v1.5.1. Traefik chart 40.3.0 as the Gateway controller, LB VIP
  `10.20.20.200`, kubernetesIngress disabled. EntryPoints bind real 80/443 via
  `NET_BIND_SERVICE` so Gateway `:443` listeners match.
- `Gateway/lab-gateway` (ns traefik): HTTPS listener `*.core.lab.2bit.name`,
  terminates with the wildcard cert, allows routes from all namespaces. Programmed=True.
- HTTPRoutes: `longhorn-ui` -> longhorn.core.lab.2bit.name (proven over browser-valid
  HTTPS, `ssl_verify_result=0`). Add per-service HTTPRoutes here as services land.

## Next
- Plan 2c: Pi-hole (.53) + Technitium (.54) DNS, their HTTPRoutes on lab-gateway,
  staged client-DNS cutover (closes Plan 1 Task 6).
