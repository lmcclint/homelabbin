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

## Next
- Plan 2b: cert-manager + Porkbun wildcard, Traefik running Gateway API,
  Pi-hole (.53) + Technitium (.54), staged client-DNS cutover (closes Plan 1 Task 6).
