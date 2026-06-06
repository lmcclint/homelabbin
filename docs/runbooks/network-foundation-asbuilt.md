# Lab Network — As-Built Record

_Last updated: 2026-06-05 (Task 0 baseline)_

## Networks (UDM SE)

| VLAN | Name | Subnet | Gateway | DHCP | Notes |
|---|---|---|---|---|---|
| 1 | Default | 192.168.1.0/24 | .1 | server | Home/family devices |
| 3 | 3Link | 192.168.3.0/24 | .1 | server | Legacy Beelink home — to retire |
| 10 | lab-mgmt | 10.20.10.0/24 | .1 | server | iDRAC, DSM mgmt, switch mgmt |
| 20 | lab-core | 10.20.20.0/24 | .1 | server | Beelink nodes + MetalLB VIPs |
| 50 | lab-stor | 10.20.50.0/24 | .1 | server | Left as-is; future storage net |
| 60 | lab-cntr | 10.20.60.0/24 | .1 | server | Synology macvlan services |

## Firewall (inter-VLAN)
- (none recorded yet)

## SSIDs
- (none recorded yet)

## Local DNS records (UDM)
- (none recorded yet)

## Beelink nodes
- (not yet migrated to VLAN 20)
