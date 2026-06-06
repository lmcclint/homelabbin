# Network Foundation Design (Starter)

**Date:** 2026-06-05
**Status:** Approved design — pending implementation plan
**Supersedes (on implementation):** the networking portions of `docs/01-overview.md` and `docs/02-networking.md`

## Purpose

Establish a solid, verified network and addressing foundation for the homelab so
later phases (k8s services, OpenShift, KVM) build on stable ground. The guiding
principle this round: **build the full VLAN/addressing scheme on the UDM SE
properly, but deploy a small footprint on top of it.** Network on paper is
complete; deployment starts minimal.

This document is the authoritative source for what we are building first. The
older `01-overview.md` / `02-networking.md` are aspirational and contain
contradictions (e.g. VLAN 70 listed as both KVM and k8s nodes; storage shown as
VLAN 40 in one place and VLAN 50 in another). Those will be reconciled against
this doc during implementation.

## Scope

In scope:
- VLAN and IP addressing plan (validated)
- DNS architecture (Pi-hole + Technitium on the core cluster)
- Inter-VLAN access policy for the starter phase

UDM configuration is done **by hand in the UniFi UI** for now — the setup is simple
enough that config-as-code (Ansible) isn't worth the added learning curve yet. See
"Deferred / Optional" for the automation path if/when it's wanted later.

Out of scope (reserved on paper, deployed later):
- OpenShift cluster VLANs (30, 40), KVM (70), DMZ (80), guest (90), disconnected (100)
- **Storage VLAN L2/jumbo-frame setup** — deferred until multi-homed compute exists
  (see Storage VLAN section)
- Config-as-code automation for the UDM (Ansible) — deferred

## Decisions At A Glance

| Topic | Decision |
|---|---|
| Supernet | `10.20.0.0/16`, VLAN ID = third octet |
| Home network | VLAN 1 `Default` `192.168.1.0/24` — unchanged |
| Legacy VLAN 3 `3Link` | Retire after Beelinks migrate to VLAN 20 |
| Core cluster (Beelinks) | VLAN 20 only, single NIC, consume NAS storage over routed network |
| Git (Gitea) | Stays on Synology macvlan, VLAN 60 (data gravity) |
| Storage VLAN 50 | **Leave as-is for now.** Defer L2-only + MTU 9000 until multi-homed compute (M710q/T640) exists. Beelinks use the NAS over the routed network at MTU 1500. |
| MetalLB | L2/ARP mode; dns pool `10.20.20.53-54` + general pool `.200-.254` |
| DNS | Pi-hole (client-facing) + Technitium (authoritative/recursive) on cluster, MetalLB VIPs |
| DNS resilience | Single HA VIP (no non-filtering secondary). UniFi Local DNS records for bootstrap. Emergency bypass SSID. |
| Home → lab access | Allowed directly (no network hopping) |
| Config management | Manual UniFi UI for now; Ansible deferred |

## VLAN & Addressing Plan

### Active in starter phase

| VLAN | Name | Subnet | Gateway | DHCP scope | Static range | VIP pool | Notes |
|---|---|---|---|---|---|---|---|
| 1 | Default (home) | 192.168.1.0/24 | .1 | existing | — | — | Family/home devices. Unchanged. |
| 10 | lab-mgmt | 10.20.10.0/24 | .1 | .100–.199 (reservations) | .10–.99 | — | iDRAC, DSM mgmt, Cockpit, switch mgmt |
| 20 | lab-core | 10.20.20.0/24 | .1 | .100–.199 | .11–.52, .55–.99 | .53–.54, .200–.254 | Beelink nodes + MetalLB VIPs; .53/.54 carved out for DNS VIPs |
| 50 | lab-stor | 10.20.50.0/24 | (unchanged for now) | (unchanged) | — | — | Reserved for a future dedicated storage network. Left as-is until multi-homed compute exists; see Storage VLAN section. |
| 60 | lab-cntr | 10.20.60.0/24 | .1 | .100–.199 | .10–.99 | .200–.254 | Synology macvlan services (Gitea, Nexus, Vault, Caddy) |

### Reserved on paper (not deployed yet)

| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 30 | ocp-compact | 10.20.30.0/24 | OpenShift compact cluster (ACM hub) |
| 40 | ocp-sno | 10.20.40.0/24 | OpenShift SNO + worker (ACM spoke) |
| 70 | kvm | 10.20.70.0/24 | KVM/libvirt VMs on Dell T640 |
| 80 | dmz | 10.20.80.0/24 | External-facing / reverse proxy |
| 90 | guest | 10.20.90.0/24 | Guest / isolated |
| 100 | disconnected | 10.20.100.0/24 | Air-gapped (optional) |

### Addressing policy

- Tail of each /24 (`.200–.254`) reserved for load-balancer VIPs; never overlaps DHCP.
- VLAN 50 (storage): when eventually built out — static only, no gateway, jumbo
  MTU 9000 end-to-end. Deferred this phase (left as a normal routed VLAN).
- One default gateway per multi-homed host (hosts via their primary VLAN; Synology
  services via VLAN 60).
- IPAM (phpIPAM/NetBox) deferred; this table is the interim source of truth.

### Cleanup items

- **VLAN 50 `lab-stor`**: currently a normal routed VLAN with a DHCP server. **Leave
  it as-is for now** — the L2-only + jumbo-frame conversion is deferred (see Storage
  VLAN section). No action needed this phase.
- **VLAN 3 `3Link`** (192.168.3.0/24) is the old Beelink home, now empty. Retire it
  after the Beelinks are migrated to VLAN 20.

## Core Cluster Networking (Beelinks)

- 3x Beelink S12 Pro, **single NIC each** → live on **VLAN 20 only**.
- Node static IPs: `10.20.20.11–13`.
- No dedicated storage-VLAN leg in the starter phase. The single NIC means a VLAN 50
  tag would share the same physical link, so jumbo frames buy little and add
  MTU-mismatch risk. Beelinks consume NFS/iSCSI from the NAS over the routed network
  at standard MTU 1500 — fine for a few small services + Plex on 1GbE. The dedicated
  storage network is reserved for the M710q/T640 nodes later (see Storage VLAN below).
- MetalLB in **L2/ARP mode**. A LoadBalancer VIP is highly available across the 3
  nodes: if a node dies, MetalLB re-ARPs the VIP to a live node. No BGP — it buys
  nothing at this scale. Two address pools on VLAN 20:
  - **dns pool** `10.20.20.53-54` — dedicated, mnemonic (port 53) addresses for the
    DNS services. Carved out of the static host range and excluded from DHCP.
  - **general pool** `10.20.20.200-254` — everything else (Plex, future services).

## DNS Architecture

### Topology

Both DNS services run on the core Beelink cluster behind MetalLB VIPs. The cluster is
treated as stable core infrastructure (tinkering happens on separate RHEL/OpenShift-virt/ESXi
environments), so DNS-on-cluster is acceptable.

- **Pi-hole** — client-facing resolver + ad/tracker filtering. VIP `10.20.20.53`.
- **Technitium** — authoritative split-horizon `lab.2bit.name` + recursive resolver.
  VIP `10.20.20.54`. Pi-hole forwards to it.

### Query flow

1. Client → **Pi-hole** (`.53`).
2. Blocked ad/tracker domain → Pi-hole returns `NXDOMAIN`/`0.0.0.0` (a *successful*
   answer, so the client does not consult any secondary).
3. `*.lab.2bit.name` → Pi-hole conditional-forwards to **Technitium** (`.54`) →
   internal IP.
4. Normal external domain → Pi-hole → upstream (Technitium recursive, or public) → answer.

### Resilience model (Option 1: single HA VIP)

- Clients are handed **only** the Pi-hole VIP — **no non-filtering secondary**.
  Reason: stub resolvers (notably Windows) don't reliably do "primary then secondary
  on failure only"; some query both servers and cache the fastest. A non-filtering
  secondary (e.g. the UDM) would therefore cause intermittent, hard-to-debug ad
  leakage. The MetalLB VIP already provides HA across the 3 nodes, so a
  client-side secondary is unnecessary for single-node failures.
- The only uncovered failure is a **full-cluster cold boot** (e.g. after a power
  outage), during which DNS is briefly unavailable while the cluster comes up.
  Mitigations:
  - **UniFi Local DNS records** on the UDM for a handful of bootstrap-critical names
    (NAS, registry, cluster API VIP), so essentials resolve even with the lab dark.
  - **Emergency bypass SSID** (see below) to get the household online fast.
- **Cluster nodes' host resolvers** point at the **UDM**, not the cluster VIP, to
  avoid a circular dependency (cluster needing DNS the cluster provides). Inside the
  cluster, CoreDNS forwards `lab.2bit.name` to the Technitium VIP.

### Whole-house filtering + emergency bypass

- All client VLANs (including home VLAN 1) are handed the Pi-hole VIP as DNS, so the
  **whole house gets ad-blocking** and can resolve `lab.2bit.name` names directly.
- **Emergency bypass SSID** (e.g. `2bit-bypass`): a separate SSID/VLAN whose DHCP
  hands out a **public resolver (1.1.1.1) or the UDM's own resolver** — never the
  cluster VIP. Served entirely by the UDM, independent of the cluster, so it works
  even with the entire lab down. Family hops to it during an outage and is online in
  seconds (no filtering, no internal names — acceptable for "just get us back online").
  The thing that fails in this scenario is the *cluster*, not the *UDM*, so the bypass
  does not need to route around the UDM or reach the ISP router directly.

## Inter-VLAN Access (Starter Phase)

Deliberately convenience-leaning for a home lab; tighten later if needed.

- **Home VLAN 1 → lab VLANs (10/20/60):** **allowed.** Personal/home devices reach lab
  services directly without hopping onto a lab VLAN/SSID.
- **lab-mgmt VLAN 10 → all:** allowed (administrative).
- **lab-core VLAN 20 → VLAN 50 (storage), VLAN 60 (services):** allowed.
- **VLAN 60 services → VLAN 50 (storage):** allowed.
- **Storage VLAN 50:** receives only; initiates no connections; no gateway.
- **Guest/emergency-bypass VLAN:** internet only, no lab access.

## Storage VLAN (Deferred — Reference)

VLAN 50 is reserved for a future **dedicated storage network**, but is **not built
out in the starter phase**. It is left as a normal routed VLAN until there is compute
that can actually use it. Recorded here so the decision is understood, not lost.

What the eventual setup means, and why it's deferred:

- **L2-only (no gateway):** removing the VLAN's gateway makes it a flat switched
  segment — devices can only reach others on the same VLAN, nothing routes in/out.
  Keeps high-volume storage traffic on the switch fabric, isolated and off the router.
  *Cost:* any consumer needs a second NIC ("leg") on VLAN 50 with a static IP
  (multi-homing).
- **Jumbo frames (MTU 9000):** larger packets → more throughput / less CPU on big
  sequential transfers. *Footgun:* MTU must match on every device, switch port, and
  NIC in the path; a single mismatch causes silent failures (small pings fine, large
  transfers hang). Only safe on a dedicated path you control end-to-end.
- **Why deferred:** the single-NIC Beelinks don't get a storage leg, so there's no
  multi-homed path and nothing for jumbo frames to optimize today. Building it now
  would be a fast lane with no traffic — pure risk, no benefit. It earns its keep once
  the M710q/T640 nodes (with dedicated storage NICs) arrive and need the bandwidth,
  which is also exactly when end-to-end jumbo frames are safe to configure.
- **Until then:** Beelinks reach the NAS over the routed network at MTU 1500 — fully
  adequate for small services + Plex on 1GbE.

## Configuration Approach

UDM configuration is performed **by hand in the UniFi Network UI** for this phase.
The number of changes is small and one-time, and avoiding config-as-code keeps the
learning surface focused on the lab itself rather than on tooling.

Changes to make in the UI:
- Ensure VLANs 10 / 20 / 60 match the addressing table (subnet, gateway, DHCP scope).
- Per-VLAN DHCP: set the DNS server option to the Pi-hole VIP (`10.20.20.53`) on the
  client VLANs (including home VLAN 1) once Pi-hole is up.
- Add UniFi **Local DNS** records for bootstrap-critical names (NAS, registry,
  cluster API VIP).
- Create the inter-VLAN firewall rule allowing home VLAN 1 → lab VLANs (10/20/60).
- Create the emergency-bypass SSID/VLAN handing out a public/UDM resolver.
- Leave reserved VLANs (30/40/70/80/90/100) and VLAN 50 untouched for now.

Each manual change should be noted (a short runbook in the repo) so it's reproducible
and reviewable even without automation.

## Cutover Notes

All steps are done in the UniFi UI (no automation this phase).

1. Confirm/adjust VLANs 10/20/60 to match the addressing table. Leave VLAN 50 as-is.
2. Stand up Pi-hole + Technitium on the cluster (separate effort) and assign VIPs
   `.53`/`.54`.
3. Set per-VLAN DHCP DNS option to the Pi-hole VIP (`.53`) on client VLANs.
4. Add UniFi Local DNS bootstrap records and the emergency-bypass SSID.
5. Migrate Beelinks from VLAN 3 to VLAN 20; verify; then **retire VLAN 3**.
6. Add the home→lab inter-VLAN allow rule.

## Open Items / Deferred

- ~~k0s vs k3s platform choice~~ **Resolved 2026-06-05: k3s** (3-node HA with
  embedded etcd; chosen for community/docs depth and stable-and-boring fit; OpenShift
  covers vanilla/enterprise reps elsewhere). Gates Plan 2 (cluster + DNS).
- Plex on cluster with N100 iGPU QuickSync transcoding (feasibility + Intel GPU device
  plugin) — separate design.
- Storage VLAN 50 L2-only + jumbo-frame build-out, for M710q/T640 multi-homing (later phase).
- Config-as-code for the UDM (Ansible `kenmoini.unifi_network`) — optional, if manual
  upkeep ever becomes painful.
- phpIPAM/NetBox as eventual source of truth (replaces the interim table here).
