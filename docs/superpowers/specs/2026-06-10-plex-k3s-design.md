# Plex Media Server on the Core k3s Cluster — Design

_Date: 2026-06-10_
_Status: Design — pending implementation plan_

## Goal

Run Plex Media Server on the existing 3-node core k3s cluster (`fed1/2/3`, VLAN 20),
config-as-code via Ansible, with:

- Media served read-only from the UNAS Pro over NFS.
- Plex config/metadata/database on Longhorn (replicated).
- Hardware (QuickSync) transcoding on the Beelink N100 iGPU (lifetime Plex Pass present).
- LAN-only access to start. Remote access (Cloudflare) explicitly deferred.

## Context

The cluster is stable and fully config-as-code (`ansible/site.yml`); see
`docs/runbooks/cluster-operations.md`. Plex follows the established single-replica
pattern used by the DNS apps (one RWO config volume, `Recreate` strategy). This is the
first workload in the new `plex` namespace and the first NFS consumer on the cluster.

## Decisions (settled)

| Topic | Decision |
|---|---|
| Namespace | `plex` |
| Workload | Deployment, `replicas: 1`, `strategy: Recreate`, schedulable on any node |
| Image | `lscr.io/linuxserver/plex` (LinuxServer; bundles Intel media drivers, PUID/PGID, render group) |
| Media storage | Static NFS PV → `192.168.1.189:/var/nfs/shared/media`, mounted **read-only** at `/media` |
| Config storage | Longhorn PVC `plex-config` (RWO), mounted at `/config` |
| HW transcode | Host `/dev/dri` mounted into pod + node `render` GID as supplementary group |
| Access | MetalLB `LoadBalancer` VIP on VLAN 20, port 32400, `externalTrafficPolicy: Local` |
| DNS | `plex.lab.2bit.name` → the VIP, added via the Ansible DNS seed loop |
| Identity | `PLEX_CLAIM` token on first boot, stored in ansible-vault |
| Node prep | None — `nfs-utils` already installed by `node-prep` (Longhorn prereq) |

## Architecture

### Workload & lifecycle
- Single Plex replica. `strategy: Recreate` because `/config` is a single RWO Longhorn
  volume and there is one Plex server identity. A node loss causes a brief blip while the
  pod reschedules and the volume reattaches — same expectation as the DNS apps.
- No node pinning: all three Beelinks have the same N100 iGPU, so `/dev/dri` exists on any
  node Plex lands on.

### Storage
- **Media (NFS, read-only).** Static `PersistentVolume` with an `nfs:` source
  (`server: 192.168.1.189`, `path: /var/nfs/shared/media`), `accessModes: ReadOnlyMany`,
  `mountOptions: [ro, nfsvers=3, hard]`, bound to a PVC in `plex`. Mounted at `/media`
  read-only — Plex never writes to the library. **NFSv3 is required**: the UNAS rejects
  NFSv4 on the friendly path (`No such file or directory` — v4 wants Ubiquiti's long
  internal pseudo-path); v3 mounts the friendly path cleanly. Verified live on `fed1`.
  - The NFS **client is the kubelet on the node**, not the pod, so the UNAS export must
    allow the node IPs. Use subnet `10.20.20.0/24` (survives reschedule across `fed1/2/3`).
  - Inter-VLAN routing (UDM) carries `10.20.20.x → 192.168.1.189` without NAT, so the UNAS
    sees the real node IP.
  - **The UNAS Pro uses `all_squash`** (built-in default, not configurable). Confirmed by
    live test on `fed1`: a real `0660` file owned `988:988` was read successfully while
    impersonating six different uid/gid combos, including `5000:5000` (matching neither
    owner nor group). The server discards the client-asserted UID/GID and remaps every
    request to its own internal identity that owns the files. **Therefore Plex reads the
    media regardless of its `PUID`/`PGID`** — no matching user, no custom UNAS user, and no
    Kerberos is needed.
  - **Security boundary = the IP allow-list.** NFSv3/`AUTH_SYS` performs no authentication;
    any host on an allowed IP mounts with zero credentials. The `media` export allows all
    three node IPs (`10.20.20.11/12/13`), so Plex can mount from whichever node it lands on.
    A node rebuilt on a new IP would need adding to the export on the UNAS.
- **Config (Longhorn, RWO).** PVC `plex-config` mounted at `/config` (database, metadata,
  thumbnails). Small; benefits from Longhorn replication so library state survives node loss.

### Hardware transcoding (QuickSync)
- Mount host `/dev/dri` into the pod (device/hostPath) and add the node's `render` group
  GID as a supplementary group so the container can open `renderD128`.
- The render GID on the Fedora nodes is detected at implementation time (it varies per
  distro/version) rather than hard-coded.
- The LinuxServer image ships the Intel media driver; enabling HW transcode is a Plex
  setting once the device is present. Lifetime Plex Pass satisfies the licensing gate.

### Access (LAN-only)
- A dedicated MetalLB `LoadBalancer` Service exposes port `32400` on a VLAN 20 VIP
  (next free in `.200–.254`; target `10.20.20.201`, confirmed free at apply time).
- `externalTrafficPolicy: Local` from the start so Plex sees real client IPs and avoids
  kube-proxy SNAT (lesson already learned with Pi-hole). With MetalLB L2 this is the
  supported source-IP-preserving path.
- DNS `A` record `plex.lab.2bit.name` → the VIP, added via the seed loop in
  `ansible/roles/dns/tasks/main.yml`.
- Plex clients connect directly to the VIP:32400; Plex's own `*.plex.direct` certificate
  handles HTTPS, so no reverse proxy is required.
- **Out of scope for v1:** the optional `plex.core.lab.2bit.name` gateway HTTPRoute (a
  browser-convenience URL) and any remote access. Plex apps dislike being proxied, so the
  VIP is the canonical path; a gateway route can be layered later if wanted.

### Identity / claim
- `PLEX_CLAIM` (from https://plex.tv/claim, ~4-minute lifetime) is supplied as an env var
  on first boot to bind the server to the account. Stored in `ansible-vault`
  (`group_vars/all/vault.yml`). It is single-use; after claim the association lives in
  `/config`. A re-claim is only needed if `/config` is wiped.

## Manifests & config-as-code integration

- All manifests under `kubernetes/plex/` (namespace, NFS PV, PVC, config PVC, Deployment,
  Service). Mirrors `kubernetes/dns/` layout.
- A small `plex` Ansible role (or extension of the existing apply flow) renders the claim
  token secret and applies `kubernetes/plex/`, wired into `site.yml` after `dns` so
  `ansible-playbook site.yml` stays the single source of truth and remains idempotent.
- `node-prep` already installs `nfs-utils` (Longhorn prereq), so the kubelet can already
  mount NFS on any node — no change needed (verified by a live mount test on `fed1`).

## Secrets

| Secret | Where | Notes |
|---|---|---|
| `PLEX_CLAIM` token | `ansible-vault` → rendered to a k8s Secret in `plex` | Short-lived; refresh right before the first apply |

## Parameters to confirm at implementation/apply time

- **PUID/PGID** — irrelevant to NFS media access (the UNAS `all_squash` ignores client
  identity; verified). They only affect ownership of the local Longhorn `/config` volume.
  Set `PUID=1000`/`PGID=988` to mirror the media ownership Plex displays — purely cosmetic.
- **Render GID** on `fed1/2/3` (detected, then set as supplementary group).
- **VIP** — verify `10.20.20.201` (or next free) is unused before assigning.
- **NFS mount options** — `ro,nfsvers=3,hard` (v3 required; verified). Tune `rsize`/`wsize`
  only if 1 GbE throughput needs it (default 1 MB negotiated, which is fine).
- **Fresh `PLEX_CLAIM`** token captured immediately before the first deploy.

## Prerequisites (user actions on the UNAS Pro)

1. NFS already enabled on the `media` share (`/var/nfs/shared/media`) — verified.
2. All three node IPs (`10.20.20.11/12/13`) on the export allow-list — done. Add a node's
   new IP here only if it's ever rebuilt on a different address.
3. No file-permission or NFS-user action needed — live read test confirmed the nodes read
   the media regardless of identity (`all_squash`).

## Verification plan

- `kubectl -n plex get pods` → Running; `kubectl -n plex get pvc` → both Bound.
- `mount | grep media` inside the pod shows the NFS export read-only; library scan sees files.
- Web UI reachable at `https://<vip>:32400/web` and `plex.lab.2bit.name` resolves to the VIP.
- Play a file that forces transcode; Plex dashboard shows `(hw)` on the transcode session.
- `dig +short plex.lab.2bit.name @10.20.20.53` → the VIP.
- `ansible-playbook site.yml` re-run is clean/idempotent.

## Out of scope (future)

- Remote access (Cloudflare tunnel or alternative) — to be designed separately.
- Optional `plex.core.lab.2bit.name` gateway HTTPRoute.
- Companion *arr stack / writable media paths.
