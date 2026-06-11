# Core Cluster — Operations Quick Reference

**Read this first** when modifying the Beelink (`fed1/2/3`) k3s cluster or adding
services. It's the fast orientation; the authoritative detail lives in
[`k3s-cluster-asbuilt.md`](./k3s-cluster-asbuilt.md) and
[`network-foundation-asbuilt.md`](./network-foundation-asbuilt.md).

_Last updated: 2026-06-11 (Plans 1, 2a, 2b, 2c complete; Plan 3 Plex added)._

## What this cluster is

3-node HA k3s on Fedora Beelinks (`fed1/2/3` = `10.20.20.11/12/13`), VLAN 20.
**Stable core infra — tinker elsewhere.** Everything is provisioned by one
idempotent Ansible run; nothing is hand-applied that isn't in git.

| Layer | Component | Version | Where it lives |
|---|---|---|---|
| Cluster | k3s (HA, embedded etcd) | v1.36.1+k3s1 | `ansible/roles/k3s` |
| LB | MetalLB (L2) | 0.16.1 | `ansible/roles/metallb` + `roles/metallb/files/pools.yaml` |
| Storage | Longhorn (replica 2, default SC) | 1.12.0 | `ansible/roles/longhorn` |
| TLS | cert-manager + Talinx Porkbun webhook | v1.20.2 / 1.0.0 | `ansible/roles/cert-manager`, `kubernetes/cert-manager/porkbun/` |
| Ingress | Traefik + Gateway API | chart 40.3.0 / CRD v1.5.1 | `ansible/roles/traefik`, `kubernetes/gateway/` |
| DNS | Technitium (auth+recursive) | 15.2.0 | `kubernetes/dns/technitium.yaml`, `ansible/roles/dns` |
| DNS | Pi-hole (filter+forward) | 2026.05.0 | `kubernetes/dns/pihole.yaml`, `ansible/roles/dns` |
| Media | Plex Media Server (QuickSync) | 1.43.2 (linuxserver) | `kubernetes/plex/*.yaml`, `ansible/roles/plex` |

## Addresses & access

| Thing | Value |
|---|---|
| Cluster API / kubeconfig | `https://10.20.20.11:6443` / repo-root `./kubeconfig` (gitignored) |
| Pi-hole VIP / UI | `10.20.20.53` / `https://pihole.core.lab.2bit.name/admin` |
| Technitium VIP / UI | `10.20.20.54` / `https://dns.core.lab.2bit.name` |
| Gateway (Traefik) VIP | `10.20.20.200` (serves `*.core.lab.2bit.name` over LE wildcard) |
| Plex VIP / UI | `10.20.20.201:32400` / `http://plex.lab.2bit.name:32400/web` (own VIP, not the gateway) |
| MetalLB pools | dns `.53–.54` (by annotation), general `.200–.254` (auto) |
| Admin creds / secrets | `ansible/group_vars/all/vault.yml` (ansible-vault) |

```bash
export KUBECONFIG=/home/lee/git/homelabbin/kubeconfig   # for kubectl
cd /home/lee/git/homelabbin/ansible                      # for ansible/vault
```

## The one rule

**Everything is config-as-code.** Change the file, then re-run the whole stack:
```bash
cd ansible && ansible-playbook site.yml      # idempotent; safe to re-run anytime
```
`site.yml` runs roles in order: `node-prep → k3s → metallb → longhorn → cert-manager → traefik → dns → plex`. Workstation needs `helm` + python `kubernetes` (already installed).

## Common operations

### Add an internal DNS record (`something.lab.2bit.name`)
Edit the seed loop in `ansible/roles/dns/tasks/main.yml`, add
`- { name: "foo.lab.2bit.name", ip: "10.20.x.y" }`, re-run `ansible-playbook site.yml`.
(`overwrite=true` = idempotent.) — *Not* `*.core` names; those already resolve to the gateway.
The `dns` role ends with a `pihole reloaddns`, so the record resolves via `.53` immediately
(no stale-NXDOMAIN wait, no manual flush). Wildcards work too (e.g. `*.apps.ocp1.lab.2bit.name`).

### Add a new web service on the cluster (`foo.core.lab.2bit.name`)
1. Deploy it (Deployment + Service), e.g. under `kubernetes/<app>/`.
2. Add an `HTTPRoute` in `kubernetes/gateway/httproute-<app>.yaml` (copy
   `httproute-longhorn.yaml`), hostname `foo.core.lab.2bit.name`, backendRef → your Service.
3. Apply it (add to a role's apply loop, or `kubectl apply -f`).
**No DNS or cert work needed** — the `*.core` wildcard (Technitium) + wildcard cert (gateway) already cover any `*.core.lab.2bit.name` name.

### Add/rotate a secret
```bash
cd ansible && ansible-vault edit group_vars/all/vault.yml   # uses .vault-pass automatically
```
Add/change the var, then re-run `ansible-playbook site.yml` (the role renders it into a k8s Secret). To just *view*: `ansible-vault view ...` — **never** `ansible-vault decrypt <file>` without `--output` (it rewrites the file in plaintext).

### Bump a version
- Helm components (k3s, metallb, longhorn, cert-manager, traefik, webhook): edit
  `ansible/group_vars/all/versions.yml`, re-run.
- DNS app images (technitium/pihole): edit the `image:` in `kubernetes/dns/*.yaml`, re-run.

### Expand the MetalLB pool
Edit `ansible/roles/metallb/files/pools.yaml` (add ranges/pools), re-run. Avoid the
`.100–.199` DHCP scope and existing static IPs.

### Rebuild / re-add a node
Fix or reinstall Fedora on the node (keep its `10.20.20.1x` IP), then
`ansible-playbook site.yml` — `node-prep` + `k3s` re-prep and rejoin it.

## Verify it's healthy
```bash
export KUBECONFIG=/home/lee/git/homelabbin/kubeconfig
kubectl get nodes                                   # 3 Ready
kubectl get pods -A | grep -v Running | grep -v Completed   # nothing unexpected
dig +short doubleclick.net @10.20.20.53             # 0.0.0.0 (Pi-hole filtering)
dig +short longhorn.core.lab.2bit.name @10.20.20.54 # 10.20.20.200 (Technitium wildcard)
kubectl -n traefik get certificate                  # wildcard Ready=True
```

## Gotchas / lessons (already handled — don't re-trip)
- **Nodes use systemd-resolved over tagged VLAN link `enp1s0.20`**, pinned to the UDM
  (`10.20.20.1`) in node-prep — nodes must NOT resolve via the cluster's own DNS (circular).
  Check node DNS with `resolvectl status`, not `/etc/resolv.conf` (it's the `127.0.0.53` stub).
- **VG name differs per node** (`fed1/2=fedora`, `fed3=fedora_fed3`) — node-prep auto-detects it.
- **Traefik entryPoints bind real 80/443** (`NET_BIND_SERVICE`) so Gateway `:443` listeners
  match — Gateway API matches listener↔entryPoint by port number.
- **`longhorn` is the sole default StorageClass** (k3s `local-path` default is unset by the
  longhorn role; local-path still exists for scratch).
- **Ingress = Traefik + Gateway API**, NOT ingress-nginx (retired EOL March 2026).
- **Cluster owns `*.core.lab.2bit.name`** only — the flat `*.lab.2bit.name` is reserved for the
  Synology/Caddy plane. Put cluster web apps under `*.core`.
- **Single DNS VIP, no non-filtering secondary** — clients only get `.53` (avoids ad leakage);
  HA comes from MetalLB VIP failover + Longhorn, not a backup resolver.
- DNS apps are single-replica with `Recreate` strategy (single RWO Longhorn volume) — a node
  loss causes a brief blip while the pod reschedules + volume reattaches. Expected.
- **Plex media is NFSv3 read-only from the UNAS** (`192.168.1.189:/var/nfs/shared/media`); the
  UNAS `all_squash`es to `unifi-drive-nfs`, so any `PUID`/`PGID` can read it. Export is pinned
  to the node IPs `10.20.20.11/12/13`.
- **Plex QuickSync needs `privileged: true`** — a bare `/dev/dri` hostPath mount fails with
  `EPERM` (device cgroup, *not* SELinux — verified). Least-privilege alternative is the Intel
  GPU device plugin.
- **Plex post-deploy settings live in `/config` (Longhorn), not git** — if `/config` is ever
  rebuilt, re-apply in Settings → Network: Custom server access URLs
  `http://10.20.20.201:32400,http://plex.lab.2bit.name:32400` (Plex auto-detects only its
  unreachable pod IP otherwise → relay), LAN Networks `10.20.20.0/24,192.168.0.0/16`, and
  Settings → Transcoder: enable HDR tone mapping (else DV/HDR files transcode to HEVC HDR and
  show black in the web app). Re-claim with `-e plex_claim_token=claim-XXXX` on `site.yml`.
