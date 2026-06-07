# Lab Network — As-Built Record

_Last updated: 2026-06-07 (Task 5 — Plan 1 complete)_

## Networks (UDM SE)

DHCP scope `.100–.199` on lab VLANs; `.11–.99` reserved for static hosts;
`.53–.54` + `.200–.254` reserved for MetalLB VIPs. Domain `lab.2bit.name` set on
lab VLANs (10/20/50/60); home + bypass left without a lab suffix.

| VLAN | Name | Subnet | Gateway | DHCP | Domain | Notes |
|---|---|---|---|---|---|---|
| 1 | Default | 192.168.1.0/24 | .1 | server (existing) | — | Home/family devices; UNAS Pro (.189) |
| 10 | lab-mgmt | 10.20.10.0/24 | .1 | .100–.199 | lab.2bit.name | iDRAC, DSM mgmt, switch mgmt |
| 20 | lab-core | 10.20.20.0/24 | .1 | .100–.199 | lab.2bit.name | Beelink nodes + MetalLB VIPs |
| 50 | lab-stor | 10.20.50.0/24 | .1 | server | lab.2bit.name | Left as-is; future storage net |
| 60 | lab-cntr | 10.20.60.0/24 | .1 | .100–.199 | lab.2bit.name | Synology macvlan services |

## Firewall (inter-VLAN)
- **Flat by design (starter phase).** No custom block/allow rules; UniFi's default
  inter-VLAN routing is left permissive, so home VLAN 1 reaches lab VLANs directly.
  Deliberate convenience choice — revisit/segment later if needed.

## SSIDs
- **Emergency-bypass SSID: deferred (not built).** Household fallback is the ATT
  router's own guest WiFi — a fully separate router/uplink, so it bypasses the
  UDM and lab entirely. Revisit a UDM-hosted bypass SSID only if that proves
  insufficient.

## Local DNS records (UDM)
Bootstrap A records served by the always-on UDM so critical names resolve even
when the cluster (Pi-hole/Technitium) is down. Answered on every UDM interface
(verified via both `@192.168.1.1` and `@10.20.20.1`).

- `syno.lab.2bit.name` → `10.20.10.20` (Synology DSM/mgmt; lab-only NAS, single
  trunked port tagging VLANs 10 + 60, also `10.20.60.20` on VLAN 60 — add
  `syno-svc` later if a VLAN-60 name is needed)
- `unas.lab.2bit.name` → `192.168.1.189` (UNAS Pro, UDM client fixed-IP +
  Local DNS Record). **Household-shared box** (Apple Time Machine + family file
  sharing) so it lives on home VLAN 1; lab reaches it over the flat routed
  network. _Deferred enhancement:_ dual-home its 10G SFP+ onto VLAN 60
  (`10.20.60.21`) for 10G Synology↔UNAS backups when the storage config is done
  — at which point decide the multi-homing route (default GW on 1G/home side;
  static `10.20.0.0/16 → 10.20.60.1` on the 10G/lab side if UNAS OS allows).
- `fed1.lab.2bit.name` → `10.20.20.11`  (k3s node, static)
- `fed2.lab.2bit.name` → `10.20.20.12`  (k3s node, static)
- `fed3.lab.2bit.name` → `10.20.20.13`  (k3s node, static)
- `registry` / `git` / `k8s-api` → deferred until those services/cluster exist

## Beelink nodes (k3s — VLAN 20 lab-core)
Migrated off the legacy VLAN 10 placement to static VLAN 20 IPs in the `.11–.99`
range. Reachable + DNS-resolvable; OS already running, ready to image for Plan 2.

- `fed1` → `10.20.20.11`
- `fed2` → `10.20.20.12`
- `fed3` → `10.20.20.13`

Legacy VLAN 3 `3Link` retired (empty, removed from UDM).
