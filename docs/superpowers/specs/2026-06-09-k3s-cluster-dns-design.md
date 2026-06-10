# k3s Core Cluster + DNS Design (Plan 2)

**Date:** 2026-06-09
**Status:** Approved design — pending implementation plan
**Builds on:** [`2026-06-05-network-foundation-design.md`](./2026-06-05-network-foundation-design.md)
(the network foundation / Plan 1, now complete and verified)

## Purpose

Stand up the core Beelink cluster and the lab's DNS so the rest of the homelab
builds on stable, self-contained ground. The cluster is treated as **boring,
stable core infrastructure** — once up, it is rarely touched (tinkering happens on
separate RHEL/OpenShift-virt/ESXi environments). It is local-only and power-outage
resilient: no dependency on the Synology or any external service to boot and serve
DNS.

This is **Plan 2**. Plan 1 (network foundation) delivered the VLANs, flat firewall,
UDM Local DNS bootstrap records, and the three nodes (`fed1/2/3`) static on
VLAN 20 at `10.20.20.11–13`. Plan 2's gated cutover formally closes **Task 6** of
Plan 1 (point client VLANs at the Pi-hole VIP).

## Scope

In scope:
- 3-node HA k3s cluster on the existing Fedora installs (`fed1/2/3`)
- MetalLB (L2), Longhorn (replicated storage), cert-manager + Porkbun DNS-01,
  Traefik running the Gateway API
- Pi-hole (filtering) + Technitium (authoritative + recursive) DNS
- Staged client-DNS cutover + verification + rollback (Plan 1 Task 6)

Out of scope (separate specs):
- **Plan 3 — fleet-wide certs** for non-k8s consumers (Cockpit on the Fedora
  hosts, Synology DSM, iDRAC). Same LE-wildcard-via-DNS-01 strategy; the open
  question is distribution (cert-manager + an Ansible/AAP push job, or
  cert-manager → private Synology git repo → AAP fan-out). cert-manager on the
  cluster here is the issuing engine that Plan 3 reuses.
- **Plex** on the cluster (N100 iGPU QuickSync transcoding feasibility) — its own
  later design.
- **Argo CD / GitOps** — deferred; adopted later where it earns its keep (OCP/ACM),
  which is also where the hands-on GitOps learning happens.

## Decisions At A Glance

| Topic | Decision |
|---|---|
| Nodes | Keep existing **Fedora + Cockpit** on `fed1/2/3`; install k3s on top (no reflash) |
| Host hardening | `firewalld` **disabled** (flat trusted lab); **SELinux enforcing** + `k3s-selinux` policy; **zram swap disabled** |
| Cluster | k3s **HA, embedded etcd**; all 3 nodes are **servers** (control-plane + etcd + schedulable); `--disable=traefik,servicelb` |
| Install method | **Ansible** (idempotent node-prep + install; sets up the AAP direction for Plan 3) |
| Disk layout | NVMe VG free space (~460G) → new LV mounted at **`/var/lib/rancher`** (k3s data-dir); **`sda` 954G → Longhorn** disk; Fedora stays on the 15G root |
| Load balancer | **MetalLB L2** (not ServiceLB — need fixed VIPs decoupled from node IPs and two services on port 53) |
| Storage | **Longhorn**, replica count 2, `StorageClass longhorn` default; durable PVCs follow pods between nodes |
| Config management | **Helm via Ansible** (`kubernetes.core.helm`), values in git; **DNS config as code** in git on top of Longhorn |
| Ingress | **Traefik running Gateway API** (ingress-nginx is retired EOL March 2026; Gateway API is the future-proof, cert-manager-friendly, OCP-relevant choice) |
| TLS | cert-manager + Porkbun DNS-01 → **one wildcard `*.lab.2bit.name`** cert, referenced by the Gateway listener |
| DNS topology | **Pi-hole** `.53` (filter + forward) → single upstream **Technitium** `.54` (authoritative `lab.2bit.name` + recursive) |
| DNS HA | Single-replica Deployments + Longhorn + MetalLB VIP failover; brief blip on node loss accepted |
| Secrets | **Ansible Vault** encrypted vars committed; vault password + kubeconfig never committed |

## Cluster Foundation

3-node HA k3s on the existing Fedora installs, stood up by an idempotent Ansible
playbook so a rebuilt node re-preps and rejoins.

**Node prep (Ansible role, all three):**
- Disable **zram swap** (kubelet wants swap off).
- Carve the **NVMe VG free space** (~460G) into a new LV, format xfs, mount at
  `/var/lib/rancher` **before** install — keeps containerd/images off the 15G root.
- Prepare **`sda` (954G)** as the Longhorn disk (raw or mounted at
  `/var/lib/longhorn`).
- `firewalld` **disabled**; **SELinux enforcing** + install `k3s-selinux`.
- Baseline packages, time sync, `/etc/hosts` sanity.

**k3s install (HA, embedded etcd):**
- `fed1` = first server (`--cluster-init`); `fed2`/`fed3` join as **servers**
  (all three are control-plane + etcd + schedulable; odd quorum survives one node).
- `--disable=traefik,servicelb` (we run our own Traefik + MetalLB).
- `--data-dir` default (now backed by the `/var/lib/rancher` LV).
- **Node host resolvers point at the UDM (`10.20.20.1`)**, never the cluster VIP —
  avoids the circular dependency of the cluster needing the DNS it hosts.
- Kubeconfig fetched to the workstation (gitignored); nodes labeled.

## Platform Layer

Deployed via Helm-through-Ansible in dependency order.

**MetalLB (L2/ARP):** two `IPAddressPool`s on VLAN 20 + one `L2Advertisement`:
- **dns pool** `10.20.20.53–54` — assigned explicitly by service annotation
  (Pi-hole → `.53`, Technitium → `.54`).
- **general pool** `10.20.20.200–254` — auto-assigned (Gateway VIP, future services).

Rationale over ServiceLB: ServiceLB binds host ports on node IPs — it can't give a
fixed VIP decoupled from node IPs and can't run two services on port 53. MetalLB L2
gives stable VIPs with node-failover (failover, not load-spreading — correct for DNS
and admin UIs).

**Longhorn:** replicated block storage on the `sda` SSDs; **replica count 2**
(survives one node loss, lighter than 3×); `StorageClass longhorn` set default; UI
exposed via the Gateway. Plain `local-path` was rejected because its PVs are
node-pinned (a dead node strands the pod) — Longhorn lets pods reschedule with their
state, which is the actual HA requirement.

**cert-manager + Porkbun DNS-01:** cert-manager + the Porkbun webhook (Helm) + the
existing `le-dns-porkbun` ClusterIssuer. Porkbun API creds come from an
Ansible-Vault-sourced Secret (never in git plaintext). Issue **one wildcard
`*.lab.2bit.name` Certificate** → secret `lab-2bit-wildcard-tls`, auto-renewed. This
proves the DNS-01 path end-to-end and is the template Plan 3 reuses.

**Traefik + Gateway API:** Helm-install Traefik (version-pinned; the bundled copy is
disabled so there's exactly one managed instance). Traefik implements both classic
Ingress and Gateway API — start with Gateway API for the future-proof, no-rewrite
path to OCP-style routing.
- One **`Gateway`** = infra entrypoint; its Service on general-pool VIP **`.200`**;
  HTTPS listener referencing the wildcard cert secret via explicit `certificateRefs`
  (no experimental annotations).
- One **`HTTPRoute`** per UI (`pihole`, `dns`, `longhorn` `.lab.2bit.name`),
  attaching to the Gateway. Gateway is infra-owned; routes are per-app.
- **DNS port 53 does not go through the Gateway** — Pi-hole/Technitium keep their own
  L4 LoadBalancer Services at `.53`/`.54`. (Gateway API `TCPRoute`/`UDPRoute` remain
  experimental and aren't needed.)

## DNS Services

**Layout & exposure:**
- **Pi-hole** — Deployment (single replica) + Longhorn PVC. LoadBalancer Service at
  **`.53`** (udp/tcp 53). Web UI via `HTTPRoute pihole.lab.2bit.name`.
- **Technitium** — Deployment (single replica) + Longhorn PVC. LoadBalancer Service
  at **`.54`** (udp/tcp 53). Web UI via `HTTPRoute dns.lab.2bit.name`.

**Query chain:**
1. Client → **Pi-hole `.53`** (the only resolver clients are ever handed).
2. Pi-hole applies **blocklist filtering**; blocked → `0.0.0.0`/NXDOMAIN (a
   successful answer — no fallback leak).
3. Everything else → Pi-hole's **single upstream = Technitium `.54`**.
4. **Technitium** answers `lab.2bit.name` **authoritatively**, and **recurses**
   (root hints, optional DNSSEC) for the public internet.

Pi-hole stays "filter + forward"; Technitium is the one brain for internal zones +
recursion. No conditional-forward rules on Pi-hole.

**Config as code:**
- **Pi-hole v6**: `pihole.toml` + adlists + custom local DNS/CNAME records in the
  repo, applied via ConfigMap/env. Gravity DB rebuilds from adlists on start.
- **Technitium**: `lab.2bit.name` zone records + settings seeded via its HTTP API
  (an Ansible/init step) from repo files, or a restored settings backup. Longhorn
  PVC holds runtime state; git is the source of truth.

**Bootstrap / circular-dependency handling:**
- Nodes resolve via the **UDM**, never the cluster VIP.
- The **UDM Local DNS records** from Plan 1 (`syno`, `unas`, `fed1/2/3`) are the
  cold-boot floor: during a full power-cycle, essentials resolve while the cluster
  comes up. Technitium then serves the full `lab.2bit.name` zone once up (minor
  intentional overlap).

**HA behavior & tradeoff:** each service is single-replica. On node failure, k8s
reschedules the pod, **Longhorn reattaches its volume** on the new node (a replica
exists), and **MetalLB moves the VIP** — genuine single-node-failure recovery, with
a **brief blip** (tens of seconds for reschedule + reattach) on that VIP. Acceptable
for house DNS, and far better than node-pinned. Pi-hole/Technitium don't cluster
active-active cleanly, so single-replica-with-failover is the boring correct choice.

## Cutover (Plan 1 Task 6) + Verification + Rollback

**Pre-flight (all must pass before touching DHCP):**
```
dig +short git.lab.2bit.name  @10.20.20.53   # internal resolves (via Technitium)
dig +short doubleclick.net    @10.20.20.53   # blocked → 0.0.0.0/NXDOMAIN
dig +short example.com        @10.20.20.53   # external resolves
dig +short fed1.lab.2bit.name @10.20.20.54   # Technitium authoritative
```

**Staged cutover (one VLAN at a time, safest first):**
1. **lab VLANs first** (20, then 10, 60): DHCP DNS → **`10.20.20.53`** (single entry,
   no non-filtering secondary — the ad-leak guard). Renew a client, verify.
2. **Home VLAN 1 last**, only after the lab VLANs are proven.
- Nodes' host resolvers stay on the **UDM** (unchanged).
- Bypass path unchanged: **ATT guest WiFi** for emergencies.

**Per-VLAN verification (renew lease, then):**
```
# resolv.conf / ipconfig shows 10.20.20.53
dig +short doubleclick.net      # blocked
dig +short example.com          # external OK
dig +short syno.lab.2bit.name   # internal OK
```

**Rollback (fast):** revert that VLAN's DHCP DNS to **`10.20.20.1`** (UDM) or a
public resolver, renew leases. Clients only ever had `.53`, so one DHCP-field revert
fully restores resolution; UDM Local DNS records still answer critical names during
rollback.

**Record-keeping:** update `docs/runbooks/network-foundation-asbuilt.md` (per-VLAN
DNS now `.53`) and check off Plan 1 Task 6.

## Repo Structure & Secrets

**Layout:**
```
ansible/
  inventory/hosts.yml            # fed1/2/3 → .11–.13, group_vars
  site.yml                       # node-prep → k3s → platform → dns
  roles/
    node-prep/                   # zram-swap off, NVMe LV→/var/lib/rancher,
                                 #   sda→Longhorn, k3s-selinux, firewalld off
    k3s/                         # cluster-init on fed1, join fed2/3 as servers
    platform/                    # helm: metallb, longhorn, cert-manager+porkbun, traefik
    dns/                         # helm/apply: pihole, technitium + API seed step
kubernetes/                      # manifests Helm doesn't own
  metallb/                       # IPAddressPool (dns .53–54, general .200–254), L2Advertisement
  cert-manager/porkbun/          # (exists) clusterissuer + NEW wildcard Certificate
  gateway/                       # Gateway (.200, wildcard cert) + HTTPRoutes (pihole/dns/longhorn)
  dns/pihole/                    # values, pihole.toml, adlists, custom records, LB svc (.53)
  dns/technitium/                # values, zone-seed files, LB svc (.54)
```

**Secrets — Ansible Vault (committable pattern):**
- `ansible/group_vars/all/vault.yml` holds Porkbun API key/secret, Pi-hole admin
  password, k3s token — **encrypted with `ansible-vault`** (ciphertext is safe to
  commit). The **vault password is never committed** (`.gitignore` already blocks
  `.env`/`*.key`/vault-pass files).
- Ansible decrypts at deploy time and renders k8s Secrets (`porkbun-credentials`,
  etc.); plaintext never touches git or a manifest.
- **kubeconfig** stays gitignored.

**Committed:** playbooks, roles, chart values, manifests, `pihole.toml`/adlists/zone
seeds, the encrypted `vault.yml`.
**Not committed:** vault password, kubeconfig, any plaintext creds.

Result: a reproducible "clone → `ansible-playbook site.yml` (with vault pass) →
working cluster" story, config-as-code for DNS, no secrets exposed.

## Open Items / Deferred

- **Plan 3 — fleet-wide certs** (Cockpit/DSM/iDRAC): distribution mechanism TBD
  (cert-manager + AAP push job, or cert-manager → Synology git → AAP fan-out).
- **Plex** on the cluster: N100 iGPU QuickSync transcoding feasibility + Intel GPU
  device plugin — separate design.
- **Argo CD / GitOps**: adopt later (OCP/ACM), pointing it at the same chart/values
  for continuous reconciliation.
- **DNSSEC validation** on Technitium recursion: enable; confirm it doesn't break
  any internal/forwarded names.
- Exact **Pi-hole / Technitium / Longhorn / Traefik / cert-manager chart versions**
  pinned at plan-writing time.
