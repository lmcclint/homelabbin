# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a comprehensive homelab infrastructure-as-code project for testing, development, and demonstrating enterprise technologies. The focus is on multi-cluster OpenShift management with ACM, hybrid cloud scenarios, and enterprise integration testing (Active Directory, SQL Server, SIEM).

**Status**: Planning & Initial Development (Phase 1-2)

## Project Architecture

### Hardware Layout

- **Synology DS1621+**: Container services on VLAN 60 via Docker Compose (Nexus, Gitea, Vault, Splunk, Caddy)
- **3x Beelink S12 Pro**: k0s cluster for core infrastructure services (VLAN 20) - DNS, IPAM, monitoring
- **5x Lenovo M710q**: Split into two OpenShift clusters
  - 3-node compact cluster (VLAN 30) - ACM hub
  - SNO + worker (VLAN 40) - ACM spoke
- **Dell T640**: RHEL KVM hypervisor (VLAN 70) for Windows VMs (AD, SQL Server) and Podman containers

### Network Architecture

**IP Addressing**: 10.20.0.0/16 lab network with per-VLAN /24 subnets using VLAN ID as third octet (e.g., VLAN 60 → 10.20.60.0/24).

**Key VLANs**:
- VLAN 10: Management (iDRAC, Cockpit, IPMI)
- VLAN 20: Core services cluster (k0s, DNS, monitoring)
- VLAN 30: OpenShift compact cluster
- VLAN 40: OpenShift SNO + worker
- VLAN 50: Storage network (Layer 2 only, no gateway, MTU 9000)
- VLAN 60: Container services (Nexus, Gitea, Vault, Splunk, Caddy)
- VLAN 70: KVM virtual machines
- VLAN 80: DMZ/external services
- VLAN 90: Guest/isolated
- VLAN 100: Disconnected/air-gapped (optional)

**DNS Strategy**:
- Domain: `lab.2bit.name` (subdomain of owned domain `2bit.name`)
- Split-horizon DNS with Technitium (authoritative + recursive) on k0s
- Valid SSL certificates via Let's Encrypt + Porkbun DNS-01 ACME
- No .local domains (avoids mDNS conflicts and certificate issues)

### Service Architecture

#### Synology Container Services (VLAN 60)

All services use **macvlan networking** with static IPs on VLAN 60. Key architectural decisions:

1. **Split Deployment Strategy**: Services are deployed in separate stacks for modularity
   - Main stack: `deployment/synology/docker-compose.yml` (Nexus, Splunk, Vault, optional Caddy)
   - Gitea stack: `deployment/synology/gitea/docker-compose.yml`
   - Caddy stack: `deployment/synology/caddy/docker-compose.yml`

2. **Macvlan Networking Pattern**:
   - Each service container gets a static IP on VLAN 60
   - Services also connect to internal bridge networks for inter-container communication
   - **Important**: Synology host cannot directly reach macvlan containers (use another LAN device or reverse proxy)

3. **Service IPs**:
   - Caddy: 10.20.60.10
   - Nexus: 10.20.60.11
   - Gitea: 10.20.60.12
   - Vault: 10.20.60.14
   - Splunk: 10.20.60.15

#### Reverse Proxy Architecture (Caddy)

Caddy provides centralized TLS termination and routing:
- **ACME DNS-01 via Porkbun**: Wildcard certificates for `*.lab.2bit.name` without exposing ports
- **Custom build**: Uses `xcaddy` with Porkbun DNS plugin (see `deployment/synology/caddy/Dockerfile`)
- **Wildcard vhost**: Single `*.lab.2bit.name` block with host-based routing
- **Backend routing**: Routes to services on VLAN 60 (Gitea on 10.20.60.11:3000) or internal bridge network (whoami test service)

#### Secrets Management (Vault)

- **Placement**: Synology Docker, single-node Raft storage
- **TLS**: Always enabled (self-signed or internal CA)
- **Integration plan**: External Secrets Operator (ESO) for k0s cluster
- **Auth methods**: Kubernetes (for ESO), userpass, approle
- **Paths**: `secret/gitea/db`, `secret/nexus/admin`, etc.

**Security**: TLS certs and vault.hcl are NEVER committed. Use `docker-compose.override.yml` (git-ignored) to mount host paths with actual secrets.

## Common Operations

### Synology Container Services

#### Deploy Main Services
```bash
cd deployment/synology
cp .env.example .env
# Edit .env with secure passwords and network settings
docker compose up -d
```

#### Deploy Gitea (Split Stack)
```bash
cd deployment/synology/gitea
cp .env.example .env
# Edit .env: passwords, TZ, GITEA_IP, ROOT_URL, DOMAIN, SSH_DOMAIN
docker compose up -d
# Access initially at http://10.20.60.11:3000 (or configured GITEA_IP)
```

#### Deploy Caddy Reverse Proxy (Split Stack)
```bash
cd deployment/synology/caddy
cp .env.example .env
# Edit .env: TZ, CADDY_IP, ACME_EMAIL, PORKBUN_API_KEY, PORKBUN_API_SECRET_KEY, upstream addresses
docker compose up -d --build
# Test at https://whoami.lab.2bit.name (requires DNS A record)
```

#### Enable Caddy in Main Stack (Alternative)
```bash
cd deployment/synology
docker compose --profile edge up -d
```

#### View Logs
```bash
docker compose logs -f [service_name]
```

#### Restart Services
```bash
docker compose restart [service_name]
```

### Vault Operations

#### Initialize Vault (First Time)
```bash
docker exec vault vault operator init -key-shares=3 -key-threshold=2
# Store unseal keys and root token OFFLINE (password manager + printed copy)
```

#### Unseal Vault
```bash
docker exec vault vault operator unseal
# Enter unseal key (repeat 2 times for threshold of 2)
```

#### Create Raft Snapshot (Backup)
```bash
docker exec vault vault operator raft snapshot save /tmp/vault-raft.snap
docker cp vault:/tmp/vault-raft.snap ./backups/vault-raft-$(date +%Y%m%d%H%M).snap
```

## Important Patterns & Conventions

### Environment Variables

Each deployment has a `.env.example` template. The actual `.env` file is git-ignored. Common variables:

- **Network**: `SYNOLOGY_INTERFACE`, `VLAN60_SUBNET`, `VLAN60_GATEWAY`
- **Service IPs**: `CADDY_IP`, `GITEA_IP`, `VAULT_IP`, etc.
- **Passwords**: `GITEA_DB_PASSWORD`, `SPLUNK_ADMIN_PASSWORD`, etc.
- **ACME/DNS**: `ACME_EMAIL`, `PORKBUN_API_KEY`, `PORKBUN_API_SECRET_KEY`
- **Gitea URLs**: `GITEA_ROOT_URL`, `GITEA_DOMAIN`, `GITEA_SSH_DOMAIN`

### DNS Configuration

For services fronted by Caddy, create two DNS A records:
- HTTPS: `git.lab.2bit.name` → `CADDY_IP` (e.g., 10.20.60.10)
- SSH: `git-ssh.lab.2bit.name` → `GITEA_IP` (e.g., 10.20.60.12)

This split allows Caddy to handle HTTPS while Gitea receives SSH directly.

### Macvlan Networking

When adding new services to VLAN 60:

1. Define both `vlan60` (macvlan) and an internal bridge network:
```yaml
networks:
  vlan60:
    ipv4_address: 10.20.60.XX
  internal-net:
```

2. Only multi-homed hosts get default gateway. Services on macvlan should route through VLAN 60 gateway (10.20.60.1).

3. Storage VLAN 50 has NO gateway—layer 2 only with jumbo frames (MTU 9000).

### Security Practices

The `.gitignore` enforces security by excluding:
- `*.key`, `*.pem`, `*.p12`, `*.pfx` (certificates/keys)
- `.env`, `*.env.local`, `*.env.prod` (environment files)
- `docker-compose.override.yml` (host-specific configs with secrets)
- `secrets/`, `vault-keys.txt`, `passwords.txt`
- Terraform state, kubeconfig, Ansible vault passwords

**Never commit secrets**. Use:
- `.env.example` templates for documentation
- `docker-compose.override.yml` for host-specific secret mounts
- Vault for runtime secrets
- Password manager + offline storage for root credentials

### Porkbun ACME DNS-01

Both Caddy and cert-manager (planned on k8s) use Porkbun DNS-01 for automatic certificate management:
- No public ports required on Synology
- Supports wildcard certificates (`*.lab.2bit.name`)
- API keys scoped to DNS operations only
- Public zone must be hosted on Porkbun for TXT record creation

## Documentation Structure

Documentation is numbered by implementation priority:

**Phase 1 - Foundation**:
- `docs/01-overview.md`: Hardware allocation and service distribution
- `docs/02-networking.md`: VLAN architecture and IP addressing (authoritative)

**Phase 2 - Container Platform**:
- `docs/15-secrets.md`: Vault CE setup and ESO integration plan
- `docs/17-object-storage-garage.md`: GarageHQ S3-compatible storage
- `docs/container-registry-decision.md`: Nexus vs Quay comparison

**Per-Service READMEs**:
- `deployment/synology/caddy/README.md`: Reverse proxy setup and DNS-01 ACME
- `deployment/synology/gitea/README.md`: Git repository with DNS/SSH split
- `deployment/synology/vault/README.md`: Bootstrap, initialization, and backup procedures

**Read the networking docs first** (`docs/02-networking.md`) when working on network-related changes—it contains the authoritative VLAN plan and IP addressing policy.

## Integration Points

### Gitea + Caddy Integration

Gitea environment must match Caddy routing:
- `GITEA_ROOT_URL=https://git.lab.2bit.name/`
- `GITEA_DOMAIN=git.lab.2bit.name`
- `GITEA_SSH_DOMAIN=git-ssh.lab.2bit.name`

Caddy Caddyfile routes `@git` matcher to `http://10.20.60.11:3000` (or `GITEA_UPSTREAM_HTTP_ADDR` from `.env`).

### Vault + ESO Integration (Planned)

When implementing External Secrets Operator on k0s:

1. Enable Kubernetes auth in Vault with k8s API address and CA
2. Create role mapping ServiceAccount to policy `eso-read`
3. Deploy ClusterSecretStore pointing to `https://vault.lab.2bit.name:8200`
4. Create ExternalSecret resources referencing `secret/data/<path>` (kv-v2)

See `docs/15-secrets.md` for detailed configuration examples.

### MetalLB Pools

Reserved IP ranges per VLAN for load balancer VIPs:
- VLAN 20 (Core): 10.20.20.200-254
- VLAN 30 (OCP Compact): 10.20.30.200-220
- VLAN 40 (OCP SNO): 10.20.40.200-220
- VLAN 60 (Container Services): 10.20.60.200-254

Avoid conflicts when assigning static IPs.

## Current State & Next Steps

**Completed**:
- ✓ Network architecture design (VLANs, IP addressing, DNS strategy)
- ✓ Nexus deployment (unified artifact + Docker registry)
- ✓ Gitea split deployment with PostgreSQL backend
- ✓ Caddy reverse proxy with Porkbun DNS-01 ACME automation
- ✓ Vault CE with Raft storage (configuration documented)
- ✓ Documentation for networking, secrets, object storage

**In Progress**:
- Splunk deployment on Synology
- Vault initialization and TLS certificate setup
- GarageHQ S3 storage deployment

**Planned**:
- k0s cluster deployment on Beelink nodes (VLAN 20)
- Technitium DNS and Pi-hole on k0s
- OpenShift compact cluster installation (VLAN 30)
- OpenShift SNO + worker cluster (VLAN 40)
- ACM hub/spoke configuration
- External Secrets Operator integration with Vault
- KVM/libvirt setup on Dell T640 with Windows VMs

## Project Phases

1. **Foundation**: Networking, core services cluster, DNS/IPAM, storage
2. **Container Platform**: Registry, Git, artifact repo, monitoring (current phase)
3. **OpenShift Clusters**: Install compact + SNO clusters, configure ACM, GitOps integration
4. **RHEL Hypervisor**: KVM/libvirt, Windows infrastructure, VM templates
5. **Advanced Services**: Monitoring, backup/DR, compliance scenarios
