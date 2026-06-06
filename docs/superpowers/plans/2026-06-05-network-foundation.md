# Network Foundation (Starter Phase) Implementation Plan

> **For the operator:** This plan is a **manual runbook** — most steps are performed
> in the UniFi Network UI and verified from a shell on a LAN host. It is *not*
> agent-executable (no subagent can click your UDM). Work top-to-bottom; each task
> ends with a verification you must see pass before moving on, and a commit of the
> as-built record. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring the lab network to the validated foundation state — correct lab VLANs, home→lab access, an emergency-bypass SSID, bootstrap DNS records, and the Beelinks migrated to VLAN 20 with the legacy VLAN retired — all captured in a committed as-built record.

**Architecture:** All changes are made on the Ubiquiti UDM SE via the UniFi Network UI. The repo artifact is `docs/runbooks/network-foundation-asbuilt.md`, an as-built record updated and committed after each task so the network's desired/actual state is version-controlled even without config-as-code.

**Tech Stack:** Ubiquiti UDM SE (UniFi Network), `dig`/`nslookup`, `ping`, `ip`/`ipconfig`. No code.

**Source spec:** `docs/superpowers/specs/2026-06-05-network-foundation-design.md`

**Out of scope (later plans):** k0s/k3s cluster install, MetalLB, Pi-hole + Technitium (Plan 2); Plex (Plan 3); storage VLAN 50 L2/jumbo build-out; config-as-code (Ansible).

**Commit policy:** Commit after each task. **Do not push** (repo goes private first).

---

## File Structure

| File | Responsibility |
|---|---|
| `docs/runbooks/network-foundation-asbuilt.md` | Living as-built record of the lab network: per-VLAN settings, firewall rule, SSID, Local DNS records, Beelink IPs. Created in Task 0, updated each task. |

There is no application code in this plan. The as-built doc is the single tracked artifact.

---

## Conventions

- **Addressing source of truth:** the table in the design spec. Active lab VLANs this phase: 10 `lab-mgmt`, 20 `lab-core`, 60 `lab-cntr`. VLAN 50 `lab-stor` is left untouched. Reserved VLANs (30/40/70/80/90/100) are not created.
- **Run verification commands from a LAN host** on the relevant VLAN. On Linux/macOS use `dig`/`ping`/`ip addr`; on Windows use `nslookup`/`ping`/`ipconfig`.
- "Expected" output describes what success looks like — if you don't see it, stop and resolve before continuing.

---

## Task 0: Create the as-built record

**Files:**
- Create: `docs/runbooks/network-foundation-asbuilt.md`

- [ ] **Step 1: Capture the current UDM network list**

In the UniFi UI: **Settings → Networks**. Note each network's Name, VLAN ID, Subnet, Gateway, and DHCP mode. (You already have a screenshot baseline: VLANs 1, 3, 10, 20, 50, 60.)

- [ ] **Step 2: Write the initial as-built file**

Create `docs/runbooks/network-foundation-asbuilt.md` with this content:

```markdown
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
```

- [ ] **Step 3: Commit**

```bash
git add docs/runbooks/network-foundation-asbuilt.md
git commit -m "docs(runbook): network foundation as-built baseline"
```

---

## Task 1: Reconcile lab VLANs 10 / 20 / 60

Confirm the three active lab VLANs match the addressing plan. Most should already be correct from your earlier setup; this task verifies and corrects DHCP scopes and reserves the static/VIP ranges.

**Files:**
- Modify: `docs/runbooks/network-foundation-asbuilt.md`

- [ ] **Step 1: Set VLAN 20 `lab-core` DHCP scope and reservations**

UniFi UI → **Settings → Networks → lab-core**:
- Gateway/Subnet: `10.20.20.1/24` (confirm).
- DHCP Range: **start `10.20.20.100`, stop `10.20.20.199`**.
- This leaves `.11–.99` for static hosts (Beelink nodes will be `.11–.13`) and `.53–.54` + `.200–.254` for MetalLB VIPs (assigned later by MetalLB, not the UDM). No UDM action needed for the VIP ranges beyond keeping DHCP off them.

- [ ] **Step 2: Set VLAN 10 `lab-mgmt` and VLAN 60 `lab-cntr` DHCP scopes**

For each (Settings → Networks → *network*):
- `lab-mgmt`: Gateway `10.20.10.1/24`, DHCP range `10.20.10.100–10.20.10.199`.
- `lab-cntr`: Gateway `10.20.60.1/24`, DHCP range `10.20.60.100–10.20.60.199`.

- [ ] **Step 3: Verify DHCP and routing on VLAN 20**

From a laptop/device placed on VLAN 20 (wired port profiled to `lab-core`, or a test SSID on VLAN 20):

```bash
ip addr show        # expect an address in 10.20.20.100–199
ping -c2 10.20.20.1 # expect replies from the VLAN 20 gateway
ping -c2 1.1.1.1     # expect replies (internet via routed VLAN)
```

Expected: an address inside the DHCP scope, gateway reachable, internet reachable.

- [ ] **Step 4: Update the as-built record**

In `docs/runbooks/network-foundation-asbuilt.md`, update the Networks table DHCP column for VLANs 10/20/60 to read `100–199 (static .11–.99; VIPs .53–54,.200–254)` and bump the `_Last updated_` line to `(Task 1)`.

- [ ] **Step 5: Commit**

```bash
git add docs/runbooks/network-foundation-asbuilt.md
git commit -m "docs(runbook): reconcile lab VLAN 10/20/60 DHCP scopes"
```

---

## Task 2: Home → lab inter-VLAN access rule

Allow personal/home devices on VLAN 1 to reach lab services directly (no network hopping), per the spec's convenience-leaning policy.

**Files:**
- Modify: `docs/runbooks/network-foundation-asbuilt.md`

- [ ] **Step 1: Create the allow rule**

UniFi UI → **Settings → Security → (Traffic & Firewall) Rules** → create a rule:
- Action: **Allow**
- Source: **Network = Default (VLAN 1)**
- Destination: **Networks = lab-mgmt (10), lab-core (20), lab-cntr (60)**
- (Leave the default inter-VLAN block in place for other VLANs; this rule sits above it for VLAN 1 → lab.)

> Note: on newer UniFi OS the equivalent lives under **Settings → Routing & Firewall → Firewall Rules** (LAN IN) or the **Traffic Rules** UI. Use whichever your controller exposes; the intent is "VLAN 1 may initiate to VLANs 10/20/60."

- [ ] **Step 2: Verify from a VLAN 1 host**

From a normal home device on VLAN 1:

```bash
ping -c2 10.20.20.1   # expect replies (can reach lab-core gateway)
ping -c2 10.20.60.1   # expect replies (can reach lab-cntr gateway)
```

Expected: replies from both lab gateways. (Reaching an actual service IP comes once services exist; gateway reachability confirms the route + rule.)

- [ ] **Step 3: Update the as-built record**

Under `## Firewall (inter-VLAN)` replace the placeholder with:

```markdown
- ALLOW: VLAN 1 (Default) → VLAN 10/20/60 (home devices reach lab directly)
```

Bump `_Last updated_` to `(Task 2)`.

- [ ] **Step 4: Commit**

```bash
git add docs/runbooks/network-foundation-asbuilt.md
git commit -m "docs(runbook): allow home VLAN1 -> lab VLAN 10/20/60"
```

---

## Task 3: Emergency-bypass SSID

A standalone wireless network that hands out a public resolver (not the cluster), served entirely by the UDM, so the household can get online even with the whole lab down.

**Files:**
- Modify: `docs/runbooks/network-foundation-asbuilt.md`

- [ ] **Step 1: Create a bypass VLAN (or reuse guest)**

UniFi UI → **Settings → Networks** → create `lab-bypass` if you want it isolated (e.g. VLAN 90 `10.20.90.0/24`, gateway `.1`, DHCP `.100–.199`). Alternatively reuse an existing Guest network.

- [ ] **Step 2: Set its DHCP DNS to a public resolver**

On that network's settings, set **DHCP Name Server → Manual → `1.1.1.1`, `1.0.0.1`** (or `8.8.8.8`). This is the key property: the bypass must **never** point at the cluster DNS VIP.

- [ ] **Step 3: Create the SSID**

UniFi UI → **Settings → WiFi** → create SSID `2bit-bypass` → assign it to the bypass network/VLAN. Optionally leave it disabled until needed, or keep it broadcasting as a permanent fallback.

- [ ] **Step 4: Verify**

Join `2bit-bypass` from a phone/laptop:

```bash
nslookup example.com        # resolves via 1.1.1.1
ping -c2 1.1.1.1            # internet works
ping -c2 10.20.20.1        # should FAIL/timeout (no lab access from bypass)
```

Expected: internet + public DNS work; lab is unreachable.

- [ ] **Step 5: Update the as-built record**

Under `## SSIDs` add:

```markdown
- 2bit-bypass → VLAN 90 (lab-bypass), DHCP DNS = 1.1.1.1/1.0.0.1, no lab access. Emergency household fallback.
```

Add the bypass VLAN to the Networks table. Bump `_Last updated_` to `(Task 3)`.

- [ ] **Step 6: Commit**

```bash
git add docs/runbooks/network-foundation-asbuilt.md
git commit -m "docs(runbook): add emergency-bypass SSID and VLAN"
```

---

## Task 4: UniFi Local DNS bootstrap records

Static host records on the UDM for the few names that must resolve even when the cluster (and its Pi-hole/Technitium) is down.

**Files:**
- Modify: `docs/runbooks/network-foundation-asbuilt.md`

- [ ] **Step 1: Add Local DNS records**

UniFi UI → **Settings → Routing & Firewall → (or) Network → Local DNS / DNS Records**. Add A records for bootstrap-critical names. Use the IPs that apply to your setup; suggested starting set:

```
nas.lab.2bit.name        -> 10.20.60.<synology VLAN60 IP>   (or 10.20.10.20 mgmt)
registry.lab.2bit.name   -> 10.20.60.11                      (Nexus, per existing docs)
k8s-api.lab.2bit.name    -> 10.20.20.<cluster API VIP, TBD in Plan 2>
```

> If you don't yet know the cluster API VIP, add the NAS and registry now and append the API record during Plan 2. Record whatever you add in the as-built file.

- [ ] **Step 2: Verify**

From any LAN host, query the UDM directly (replace `10.20.20.1` with the gateway of the VLAN you're on):

```bash
dig +short nas.lab.2bit.name @10.20.20.1
dig +short registry.lab.2bit.name @10.20.20.1
```

Expected: each returns the IP you set.

- [ ] **Step 3: Update the as-built record**

Under `## Local DNS records (UDM)` list exactly what you added, e.g.:

```markdown
- nas.lab.2bit.name -> 10.20.60.13
- registry.lab.2bit.name -> 10.20.60.11
```

Bump `_Last updated_` to `(Task 4)`.

- [ ] **Step 4: Commit**

```bash
git add docs/runbooks/network-foundation-asbuilt.md
git commit -m "docs(runbook): add UDM Local DNS bootstrap records"
```

---

## Task 5: Migrate Beelinks to VLAN 20 and retire VLAN 3

Move the three Beelink nodes onto `lab-core` (VLAN 20) with static IPs, verify, then remove the now-empty legacy `3Link` VLAN.

**Files:**
- Modify: `docs/runbooks/network-foundation-asbuilt.md`

- [ ] **Step 1: Assign each Beelink a static IP on VLAN 20**

For each node, set the switch port profile (or its IP) so the three nodes land on VLAN 20 at:

```
beelink-1 -> 10.20.20.11
beelink-2 -> 10.20.20.12
beelink-3 -> 10.20.20.13
```

Use UniFi DHCP **reservations** (Client → Fixed IP) or static config on the hosts. Keep them inside `.11–.99` (outside the `.100–.199` DHCP scope).

- [ ] **Step 2: Verify each node**

From each Beelink (or by pinging from another VLAN 20 host):

```bash
ping -c2 10.20.20.11
ping -c2 10.20.20.12
ping -c2 10.20.20.13
ping -c2 10.20.20.1     # gateway reachable from a node
```

Expected: all four reachable.

- [ ] **Step 3: Confirm VLAN 3 is empty**

UniFi UI → **Insights / Clients**, filter by network `3Link`. Expected: **no active clients**. (If anything remains, move it before deleting.)

- [ ] **Step 4: Delete VLAN 3 `3Link`**

UniFi UI → **Settings → Networks → 3Link → Remove**. Confirm.

- [ ] **Step 5: Verify deletion did not disrupt anything**

```bash
ping -c2 10.20.20.11    # nodes still reachable
ping -c2 1.1.1.1        # internet still fine
```

Expected: still healthy.

- [ ] **Step 6: Update the as-built record**

Remove the VLAN 3 row from the Networks table, fill in `## Beelink nodes`:

```markdown
- beelink-1 -> 10.20.20.11
- beelink-2 -> 10.20.20.12
- beelink-3 -> 10.20.20.13
```

Bump `_Last updated_` to `(Task 5)`.

- [ ] **Step 7: Commit**

```bash
git add docs/runbooks/network-foundation-asbuilt.md
git commit -m "docs(runbook): migrate Beelinks to VLAN 20, retire VLAN 3"
```

---

## Task 6 (GATED): Point client VLANs at the Pi-hole DNS VIP

> **Blocked until Plan 2** delivers Pi-hole on the cluster at VIP `10.20.20.53`.
> Do not perform this task until that VIP answers DNS, or you will break name
> resolution on every client VLAN.

**Files:**
- Modify: `docs/runbooks/network-foundation-asbuilt.md`

- [ ] **Step 1: Pre-check the VIP answers**

```bash
dig +short git.lab.2bit.name @10.20.20.53     # expect the Gitea/Caddy IP
dig +short doubleclick.net @10.20.20.53        # expect 0.0.0.0 / NXDOMAIN (blocked)
```

Expected: internal name resolves; a known ad domain is blocked. Only proceed if both pass.

- [ ] **Step 2: Set per-VLAN DHCP DNS to the VIP**

For each **client** VLAN (1 Default, 10 lab-mgmt, 20 lab-core, 60 lab-cntr): Settings → Networks → *network* → **DHCP Name Server → Manual → `10.20.20.53`** (single entry, per the single-HA-VIP decision — do **not** add the UDM as a non-filtering secondary). Leave the bypass VLAN on `1.1.1.1`.

- [ ] **Step 3: Verify a client picks it up**

Renew DHCP on a client, then:

```bash
# Linux/macOS
cat /etc/resolv.conf            # expect nameserver 10.20.20.53
dig +short doubleclick.net      # expect blocked
# Windows
ipconfig /all                   # DNS Servers: 10.20.20.53
```

Expected: client uses `.53`; ad domain blocked; internal + external names resolve.

- [ ] **Step 4: Update the as-built record**

Note per-VLAN DNS = `10.20.20.53` (bypass = `1.1.1.1`). Bump `_Last updated_`.

- [ ] **Step 5: Commit**

```bash
git add docs/runbooks/network-foundation-asbuilt.md
git commit -m "docs(runbook): point client VLANs at Pi-hole DNS VIP"
```

---

## Done criteria

- VLANs 10/20/60 verified; DHCP scopes correct; VIP/static ranges reserved.
- Home VLAN 1 can reach lab gateways 10/20/60.
- `2bit-bypass` SSID gives internet + public DNS, no lab access.
- UDM Local DNS resolves the bootstrap names.
- Beelinks live at `10.20.20.11–13`; VLAN 3 removed.
- `docs/runbooks/network-foundation-asbuilt.md` reflects the final state, committed (not pushed).
- Task 6 remains unchecked until Plan 2 (cluster + Pi-hole) is delivered.
