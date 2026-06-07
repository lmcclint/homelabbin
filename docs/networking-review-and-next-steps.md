# Networking Review & Next Steps — SUPERSEDED

> **This document is superseded and intentionally trimmed.**
>
> It described an earlier, contradictory VLAN scheme (VLAN 20 as Users/Admin,
> VLAN 40 as storage, VLAN 70/71 as k8s nodes/VIPs, VLAN 98/99, a deny-by-default
> firewall, and the legacy `192.168.x` lab segments). All of that has been
> replaced.
>
> The current, authoritative networking material lives in:
>
> - **Design (validated):** [`superpowers/specs/2026-06-05-network-foundation-design.md`](./superpowers/specs/2026-06-05-network-foundation-design.md)
> - **Implementation runbook (Plan 1):** [`superpowers/plans/2026-06-05-network-foundation.md`](./superpowers/plans/2026-06-05-network-foundation.md)
> - **As-built (actual UDM state):** [`runbooks/network-foundation-asbuilt.md`](./runbooks/network-foundation-asbuilt.md)
> - **Architecture overview:** [`02-networking.md`](./02-networking.md)
>
> The full original content remains in git history if needed for reference.
