# Homelab Plan

## Overview
This homelab supports testing and development needs, focusing on OpenShift, Kubernetes, container technologies, and enterprise integration scenarios.

## Hardware Allocation

### Storage Infrastructure
- **Synology DS1621+** (10Gbps, SSD array)
  - Container registries and artifact repositories
  - Fast storage for kubernetes persistent volumes via democratic-csi
  - File services (NFS/SMB)
  
- **Ubiquiti UNAS Pro** (10Gbps, spinning rust)
  - Long-term storage and backups
  - Archive storage

### Compute Infrastructure
- **3x Beelink S12 Pro** (16GB RAM, 4 CPU each)
  - k3s cluster (3-node HA, embedded etcd) for core infrastructure services
  - High availability for critical services
  
- **5x Lenovo M710q** - Split into two OpenShift clusters:
  - **Compact 3-node cluster** (stable/production-like)
    - ACM control plane
    - All nodes as control plane + worker (schedulable masters)
  - **Single Node OpenShift + 1 worker** (testing/experimental)
    - SNO for edge computing scenarios
    - Additional worker for workload distribution testing
  
- **Dell T640** (56 cores, 512GB RAM, 40TB NVMe)
  - RHEL-based hypervisor with KVM/libvirt + Podman
  - Low-power idle consumption
  - On-demand test environments (VMs + containers)
  - Windows workloads (AD, SQL Server) via KVM
  - Native Red Hat virtualization stack

## Service Distribution

### Core Services Cluster (k3s on Beelinks — VLAN 20)
- DNS: Pi-hole (client-facing filtering, VIP `10.20.20.53`) + Technitium
  (authoritative split-horizon + recursive, VIP `10.20.20.54`)
- Load balancing (MetalLB, L2/ARP mode)
- Certificate management (cert-manager)
- Monitoring (Prometheus + Grafana)
- IPAM (phpIPAM or NetBox) — later
- Plex (if N100 iGPU QuickSync transcoding pans out)

### Storage Services (Synology)
- Container registry (Nexus)
- Artifact repository (Nexus/Artifactory)
- Git repository (Gitea/GitLab CE)
- Democratic-CSI storage provisioner
- Secrets store (HashiCorp Vault CE)
- **Splunk** (initial placement - monitor CPU usage)

### OpenShift Clusters (M710q nodes)

#### Compact 3-Node Cluster (Production-like)
- OpenShift 4.14+
- Advanced Cluster Management (ACM) - **Primary hub cluster**
- OpenShift GitOps (ArgoCD)
- OpenShift Pipelines (Tekton)
- Quay operator
- Red Hat OpenShift Data Foundation (if needed)

#### Single Node OpenShift + Worker (Testing)
- OpenShift 4.14+ (SNO deployment)
- ACM managed cluster (spoke)
- Edge computing use case testing
- Resource-constrained scenario testing
- Workload migration testing between SNO and worker

### RHEL Hypervisor Platform (Dell T640)
- **KVM/libvirt VMs**:
  - Windows Server 2022 (Active Directory)
  - SQL Server 2022 (multiple editions/versions)
  - Additional k8s test clusters
  - Red Hat Satellite
  - Client testing VMs
  - Nested ESXi testing (when needed)
- **Podman containers**:
  - Resource-intensive services
  - Development environments
  - CI/CD runners

## Additional Services to Consider

### Identity & Security
- Red Hat SSO/Keycloak
- HashiCorp Vault
- Red Hat Advanced Cluster Security

### DevOps & Observability
- Jenkins (traditional CI/CD)
- SonarQube
- Jaeger (distributed tracing)
- Elastic Stack
- **Splunk Enterprise** (log aggregation, SIEM, enterprise integration)

### Testing Infrastructure
- Red Hat CodeReady Workspaces
- Selenium Grid
- Test databases (PostgreSQL, MySQL, MongoDB)

## Network Design

> **Detailed networking plan**: See [02-networking.md](./02-networking.md) and the
> authoritative design spec + as-built record it references.

### VLAN Summary

Active (deployed): home stays on VLAN 1 (`192.168.1.0/24`); lab uses the
`10.20.0.0/16` supernet with the VLAN ID as the third octet.

- **VLAN 10** — `lab-mgmt` (`10.20.10.0/24`): iDRAC, DSM/UNAS mgmt, switch mgmt
- **VLAN 20** — `lab-core` (`10.20.20.0/24`): k3s Beelink nodes + MetalLB VIPs
- **VLAN 50** — `lab-stor` (`10.20.50.0/24`): reserved storage net, routed for now
- **VLAN 60** — `lab-cntr` (`10.20.60.0/24`): Synology macvlan services

Reserved on paper (not yet deployed): **30** ocp-compact, **40** ocp-sno,
**70** kvm, **80** dmz, **90** guest, **100** disconnected. Legacy VLAN 3 retired.

### DNS Strategy
- **Internal**: `lab.2bit.name` (split-horizon); **External**: `2bit.name`
  (no lab exposure initially)
- **Pi-hole** (VIP `10.20.20.53`): client-facing first hop, ad/tracker filtering;
  conditional-forwards `lab.2bit.name` to Technitium
- **Technitium** (VIP `10.20.20.54`): authoritative `lab.2bit.name` + recursive
- Single HA VIP (no non-filtering secondary); UDM Local DNS records for bootstrap

## Implementation Phases

1. **Foundation**: Network, core services cluster, DNS/IPAM, storage
2. **Container Platform**: Registry, Git, artifact repo, monitoring
3. **OpenShift Clusters**: 
   - Install compact 3-node cluster with ACM hub
   - Deploy SNO + worker cluster as ACM spoke
   - Configure GitOps integration
4. **RHEL Hypervisor Platform**: KVM/libvirt setup, Windows VMs, VM templates, Podman services
5. **Advanced Services**: Security scanning, advanced monitoring, automation

## Key Considerations

- Multi-cluster management with ACM
- Hybrid cloud scenarios (bare metal + virtualized + containerized)
- Customer environment replication
- Integration testing (AD, databases, legacy apps)
- Certification lab environments
- **Enterprise logging/SIEM integration** with Splunk
- **Compliance and auditing** scenarios
- **Log forwarding** from OpenShift to enterprise tools

## Splunk Placement Strategy

### Option 1: Synology (Initial)
- **Pros**: Fast storage for indexes, co-located with logs
- **Cons**: Limited CPU, may struggle with heavy indexing
- **Recommendation**: Start here, monitor CPU usage

### Option 2: KVM VM on T640 (If Synology struggles)
- **Pros**: Dedicated CPU resources, easy to scale, native RHEL integration
- **Cons**: Network hop to storage, more resource overhead
- **Use case**: Heavy indexing, multiple users, enterprise features

### Option 4: Podman on T640 (Alternative)
- **Pros**: Container efficiency, easy management, resource sharing
- **Cons**: Less isolation than VM, shared resources
- **Use case**: Moderate workloads, integrated with other containers

### Option 3: OpenShift Compact Cluster
- **Pros**: Kubernetes-native, easy scaling, integrated monitoring
- **Cons**: May impact cluster stability, resource competition
- **Use case**: Cloud-native deployment, integration with OCP logging

## Questions for Refinement

1. Specific OpenShift version preferences?
2. Preference for GitLab vs Gitea for git hosting?
3. Any specific Red Hat products to prioritize?
4. External connectivity requirements (VPN, public access)?
5. Backup strategy preferences?
6. Any compliance requirements to simulate?
7. **Expected Splunk data volume** (logs/day, retention period)?

## Related Documents

- [Networking Plan](./02-networking.md) - Detailed VLAN and network architecture
- [Core Services Setup](./03-core-services.md) *(TBD)* - k3s cluster deployment
- [Storage Configuration](./04-storage.md) *(TBD)* - Synology and democratic-csi setup
- [Container Registry Setup](./05-container-registry.md) *(TBD)* - Harbor deployment
- [Security Configuration](./15-security.md) *(TBD)* - Certificates and compliance

---
*This plan will be migrated to a git repository once the infrastructure is established.*

