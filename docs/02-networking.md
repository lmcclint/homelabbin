# Homelab Networking Plan

> **Authoritative sources.** The validated design lives in
> [`docs/superpowers/specs/2026-06-05-network-foundation-design.md`](./superpowers/specs/2026-06-05-network-foundation-design.md);
> the **as-built** (what is actually configured on the UDM right now) lives in
> [`docs/runbooks/network-foundation-asbuilt.md`](./runbooks/network-foundation-asbuilt.md).
> This document is the human-readable architecture overview reconciled against
> those two. Where they disagree, the spec/as-built win.

## Guiding Principle

**Build the full VLAN/addressing scheme on the UDM SE properly, but deploy a
small footprint on top of it.** The network on paper is complete; deployment
starts minimal and grows. Reserved VLANs are documented but not created until
there is something to put on them.

## IP Addressing

- **Supernet:** `10.20.0.0/16` for the lab; home stays `192.168.1.0/24`.
- **Per-VLAN /24** using the VLAN ID as the third octet (VLAN 20 → `10.20.20.0/24`).
- **Domain:** `lab.2bit.name` (split-horizon), a subdomain of the owned `2bit.name`.

Rationale for `10.x`: avoids overlap with common `192.168.x` networks, scales
cleanly with a /24 per VLAN, and clearly separates lab from the home network.

## VLAN Architecture

### Active (deployed in the starter phase)

| VLAN | Name | Subnet | Gateway | DHCP scope | Static range | VIP pool | Purpose |
|---|---|---|---|---|---|---|---|
| 1 | Default (home) | 192.168.1.0/24 | .1 | existing | — | — | Family/home devices; UNAS Pro (`.189`). Unchanged. |
| 10 | lab-mgmt | 10.20.10.0/24 | .1 | .100–.199 | .11–.99 | — | iDRAC, DSM/UNAS mgmt, Cockpit, switch mgmt |
| 20 | lab-core | 10.20.20.0/24 | .1 | .100–.199 | .11–.52, .55–.99 | .53–.54, .200–.254 | k3s Beelink nodes (`fed1/2/3 = .11–.13`) + MetalLB VIPs |
| 50 | lab-stor | 10.20.50.0/24 | .1 (for now) | unchanged | — | — | Reserved future storage net. Left as a normal routed VLAN until multi-homed compute exists (see Storage VLAN). |
| 60 | lab-cntr | 10.20.60.0/24 | .1 | .100–.199 | .10–.99 | .200–.254 | Synology macvlan services (Gitea, Nexus, Vault, Caddy) |

### Reserved on paper (not deployed yet)

| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 30 | ocp-compact | 10.20.30.0/24 | OpenShift compact 3-node cluster (ACM hub) |
| 40 | ocp-sno | 10.20.40.0/24 | OpenShift SNO + worker (ACM spoke) |
| 70 | kvm | 10.20.70.0/24 | KVM/libvirt VMs on Dell T640 (AD, SQL Server, etc.) |
| 80 | dmz | 10.20.80.0/24 | External-facing / reverse proxy |
| 90 | guest | 10.20.90.0/24 | Guest / isolated |
| 100 | disconnected | 10.20.100.0/24 | Air-gapped / disconnected (optional) |

> The legacy **VLAN 3 `3Link`** (`192.168.3.0/24`, old Beelink home) has been
> **retired** — the Beelinks now live on VLAN 20.

### Addressing policy

- **DHCP** `.100–.199` on lab VLANs; **static hosts** `.11–.99`; the tail
  `.200–.254` is reserved for load-balancer (MetalLB) VIPs and never overlaps DHCP.
- VLAN 20 additionally carves `.53–.54` out of the static range for the DNS VIPs.
- One default gateway per multi-homed host (hosts via their primary VLAN;
  Synology services via VLAN 60).
- IPAM (phpIPAM/NetBox) is deferred; the tables here + the as-built record are the
  interim source of truth.

## DNS Architecture

Both DNS services run on the core k3s cluster behind MetalLB VIPs. The cluster is
treated as stable core infrastructure (tinkering happens on separate
RHEL/OpenShift-virt/ESXi environments), so DNS-on-cluster is acceptable.

- **Pi-hole** — client-facing resolver + ad/tracker filtering. VIP `10.20.20.53`.
- **Technitium** — authoritative split-horizon `lab.2bit.name` + recursive
  resolver. VIP `10.20.20.54`. Pi-hole forwards to it.

### Query flow

1. Client → **Pi-hole** (`.53`).
2. Blocked ad/tracker domain → Pi-hole returns `NXDOMAIN`/`0.0.0.0` (a *successful*
   answer, so the client does not fail over to any secondary).
3. `*.lab.2bit.name` → Pi-hole conditional-forwards to **Technitium** (`.54`) →
   internal IP.
4. Normal external domain → Pi-hole → upstream (Technitium recursive, or public).

### Resilience (single HA VIP)

- Clients are handed **only** the Pi-hole VIP — **no non-filtering secondary**.
  Stub resolvers (notably Windows) don't reliably do "primary, then secondary only
  on failure"; a non-filtering secondary (e.g. the UDM) would cause intermittent
  ad leakage. The MetalLB VIP already provides HA across the 3 nodes.
- The only uncovered failure is a **full-cluster cold boot** (power outage), during
  which DNS is briefly unavailable. Mitigations:
  - **UniFi Local DNS records** on the always-on UDM for bootstrap-critical names
    (`syno`, `unas`, `fed1/2/3`, later `registry`/`k8s-api`).
  - **Emergency bypass** — the ATT router's own guest WiFi (a separate
    router/uplink) gets the household online during a lab outage. A UDM-hosted
    bypass SSID was considered and deferred.
- **Cluster nodes' host resolvers point at the UDM**, not the cluster VIP, to avoid
  a circular dependency. Inside the cluster, CoreDNS forwards `lab.2bit.name` to the
  Technitium VIP.

### Naming

- Base domain `2bit.name` (owned, public). Lab subdomain `lab.2bit.name` (internal).
- Valid SSL via Let's Encrypt + Porkbun DNS-01 ACME — no `.local`, no mDNS/cert pain.
- Future per-cluster zones when those clusters exist: `ocp-compact.lab.2bit.name`,
  `ocp-sno.lab.2bit.name`, `kvm.lab.2bit.name`, etc.

## Load Balancing (MetalLB)

L2/ARP mode (no BGP — buys nothing at this scale). A VIP is highly available across
the 3 nodes; if a node dies, MetalLB re-ARPs the VIP to a live node.

| Pool | Range | Use |
|---|---|---|
| VLAN 20 dns | `10.20.20.53–54` | Pi-hole (`.53`), Technitium (`.54`) |
| VLAN 20 general | `10.20.20.200–254` | Plex and other core-cluster services |
| VLAN 60 | `10.20.60.200–254` | Container-services VIPs (if/when needed) |
| VLAN 30 (reserved) | `10.20.30.200–220` | OpenShift compact |
| VLAN 40 (reserved) | `10.20.40.200–220` | OpenShift SNO |

## Inter-VLAN Access (Starter Phase)

Deliberately **flat / convenience-leaning** for a home lab; tighten later if needed.
This matches the as-built: no custom block rules, UniFi's default inter-VLAN routing
left permissive.

- **Home VLAN 1 → lab VLANs (10/20/60):** allowed (verified). Home devices reach lab
  services directly without hopping onto a lab VLAN/SSID.
- **lab-mgmt VLAN 10 → all:** allowed (administrative).
- **lab-core VLAN 20 → VLAN 50/60:** allowed.
- **VLAN 60 services → VLAN 50:** allowed.
- **Guest / emergency bypass:** internet only, no lab access.

> This is intentionally looser than an enterprise deny-by-default posture. Revisit
> segmentation (and a proper deny-by-default with explicit allows) once the lab
> hosts anything sensitive.

## Storage VLAN (Deferred — Reference)

VLAN 50 `lab-stor` is reserved for a future **dedicated storage network** but is
**not built out yet**. It is left as a normal routed VLAN until there is compute
that can actually use it.

- **L2-only (no gateway):** a flat switched segment — keeps high-volume storage
  traffic on the switch fabric, off the router. *Cost:* every consumer needs a
  second NIC ("leg") on VLAN 50 with a static IP (multi-homing).
- **Jumbo frames (MTU 9000):** more throughput / less CPU on big sequential
  transfers. *Footgun:* MTU must match on every device, switch port, and NIC in the
  path; a single mismatch silently breaks large transfers. Only safe on a dedicated,
  end-to-end-controlled path.
- **Why deferred:** the single-NIC Beelinks get no storage leg, so there's nothing
  to multi-home and nothing for jumbo frames to optimize today — it would be a fast
  lane with no traffic. It earns its keep when the M710q/T640 nodes (with dedicated
  storage NICs) arrive, which is also when end-to-end jumbo is safe to configure.
- **Until then:** Beelinks reach the NAS over the routed network at MTU 1500 —
  adequate for small services + Plex on 1GbE.

## Configuration Approach

UDM configuration is done **by hand in the UniFi Network UI** for this phase — the
change set is small and one-time, so config-as-code (Ansible
`kenmoini.unifi_network`) isn't worth the added learning curve yet. Every manual
change is recorded in the as-built runbook so it's reproducible and reviewable even
without automation. Automation remains an option if manual upkeep becomes painful.

## Disconnected / Air-Gapped Networks (Future Reference)

VLAN 100 is reserved for disconnected workflows — a planned future addition for
testing **"remote" / disconnected software installs** (air-gapped product installs
that must pull from a local mirror with no internet egress): OpenShift disconnected
installs, RHEL Satellite content, compliance/air-gap demos, and certification prep.
Nothing is built for it yet; this is the placeholder so the intent isn't lost.
Two implementation paths when the time comes:

- **Physical VLAN 100** — true isolation, multi-node bare-metal testing, realistic
  customer simulation. More setup.
- **KVM NAT networks on the T640** — quick to create/destroy, snapshot-able, VM-only.

Likely a hybrid: start with KVM NAT for development, add physical VLAN 100 for demos
and bare-metal scenarios.

---
*Reconciled 2026-06-07 against the network-foundation design spec and as-built
record. Earlier revisions of this file described an older, contradictory VLAN
scheme (e.g. VLAN 70 as k8s nodes, storage as VLAN 40, deny-by-default firewall) and
are superseded.*
