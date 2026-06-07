# Lab Network — As-Built Record

_Last updated: 2026-06-07 (Task 3)_

## Networks (UDM SE)

DHCP scope `.100–.199` on lab VLANs; `.11–.99` reserved for static hosts;
`.53–.54` + `.200–.254` reserved for MetalLB VIPs. Domain `lab.2bit.name` set on
lab VLANs (10/20/50/60); home + bypass left without a lab suffix.

| VLAN | Name | Subnet | Gateway | DHCP | Domain | Notes |
|---|---|---|---|---|---|---|
| 1 | Default | 192.168.1.0/24 | .1 | server (existing) | — | Home/family devices |
| 3 | 3Link | 192.168.3.0/24 | .1 | server | — | Legacy Beelink home — to retire |
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
- (none recorded yet)

## Beelink nodes
- (not yet migrated to VLAN 20)
