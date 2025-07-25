# Networking Review & Next Steps

## 🔍 Current State Analysis

### What's Working
- ✅ **UDM SE**: Operational as primary network management
- ✅ **VLAN 3**: Existing Beelink k0s cluster (192.168.1.0/24)
- ✅ **Basic connectivity**: All hardware networked and accessible

### What Needs Implementation
- 🚧 **VLAN segmentation**: 9 new VLANs need configuration
- 🚧 **Static IP assignments**: Management and service IPs
- 🚧 **Inter-VLAN routing**: Firewall rules and policies
- 🚧 **DNS infrastructure**: CoreDNS deployment and zone configuration

## 📋 VLAN Architecture Summary

| VLAN | Subnet | Purpose | Priority | Status |
|------|--------|---------|----------|---------|
| **3** | 192.168.1.0/24 | Current Beelink cluster | Current | ✅ Active |
| **10** | 192.168.10.0/24 | Management (iDRAC, IPMI) | High | 🚧 Plan |
| **20** | 192.168.20.0/24 | Core services (DNS, monitoring) | High | 🚧 Plan |
| **30** | 192.168.30.0/24 | OpenShift compact cluster | Medium | 🚧 Plan |
| **40** | 192.168.40.0/24 | OpenShift SNO + worker | Medium | 🚧 Plan |
| **50** | 192.168.50.0/24 | Storage network (10Gbps) | High | 🚧 Plan |
| **60** | 192.168.60.0/24 | Container services | Medium | 🚧 Plan |
| **70** | 192.168.70.0/24 | KVM virtual machines | Low | 🚧 Plan |
| **80** | 192.168.80.0/24 | DMZ/external services | Low | 🚧 Plan |
| **90** | 192.168.90.0/24 | Guest/isolated | Low | 🚧 Plan |

## 🎯 Implementation Phases

### Phase 1: Foundation VLANs (Week 1-2)
**Priority: Critical - Must complete first**

#### VLAN 10 - Management Network
```
Subnet: 192.168.10.0/24
Gateway: 192.168.10.1 (UDM SE)

Static IP Assignments:
- Dell T640 iDRAC: 192.168.10.10
- Dell T640 RHEL mgmt: 192.168.10.11
- M710q IPMI: 192.168.10.12-16
- Synology mgmt: 192.168.10.20
- UNAS Pro mgmt: 192.168.10.21
- Cockpit/libvirt: 192.168.10.100

DHCP Pool: 192.168.10.150-199 (temp access)
```

#### VLAN 50 - Storage Network
```
Subnet: 192.168.50.0/24
Gateway: 192.168.50.1 (UDM SE)

Static IP Assignments:
- Synology storage: 192.168.50.20
- UNAS Pro storage: 192.168.50.21
- Democratic-CSI: 192.168.50.30-39
- KVM storage: 192.168.50.100-109

Features: Jumbo frames (9000 MTU), high QoS priority
```

#### VLAN 20 - Core Services
```
Subnet: 192.168.20.0/24
Gateway: 192.168.20.1 (UDM SE)

Migration Plan:
1. Create VLAN 20
2. Schedule maintenance window
3. Migrate Beelink cluster from VLAN 3 → VLAN 20
4. Update democratic-csi storage network
5. Deploy CoreDNS with .lab.local zones

Static IP Assignments:
- Beelink k0s cluster: 192.168.20.11-13
- CoreDNS servers: 192.168.20.53-54
- IPAM (NetBox): 192.168.20.80
- Monitoring: 192.168.20.90-99
- MetalLB pool: 192.168.20.200-220
```

### Phase 2: OpenShift Networks (Week 3-4)
**Priority: High - After core services stable**

#### VLAN 30 - Compact OpenShift Cluster
```
Subnet: 192.168.30.0/24
Static IPs only (no DHCP)

M710q Assignment:
- Node 1 (master+worker): 192.168.30.11
- Node 2 (master+worker): 192.168.30.12  
- Node 3 (master+worker): 192.168.30.13
- API VIP: 192.168.30.100
- Apps Ingress: 192.168.30.101
- ACM Console: 192.168.30.110
- MetalLB pool: 192.168.30.200-220

DNS: *.apps.ocp-compact.lab.local → 192.168.30.101
```

#### VLAN 40 - SNO + Worker Cluster
```
Subnet: 192.168.40.0/24
Static IPs only (no DHCP)

M710q Assignment:
- SNO node: 192.168.40.11
- Worker node: 192.168.40.12
- API VIP: 192.168.40.100
- Apps Ingress: 192.168.40.101
- MetalLB pool: 192.168.40.200-220

DNS: *.apps.ocp-sno.lab.local → 192.168.40.101
```

### Phase 3: Application Services (Week 5-6)
**Priority: Medium - After OpenShift operational**

#### VLAN 60 - Container Services
```
Subnet: 192.168.60.0/24

Synology Container Deployment:
- Harbor registry: 192.168.60.10
- Nexus repository: 192.168.60.11
- Gitea: 192.168.60.12
- Splunk: 192.168.60.15
- Quay registry: 192.168.60.20
- MetalLB pool: 192.168.60.200-220

DNS Zones:
- harbor.lab.local → 192.168.60.10
- nexus.lab.local → 192.168.60.11
- git.lab.local → 192.168.60.12
- splunk.lab.local → 192.168.60.15
```

### Phase 4: Virtualization & DMZ (Week 7-8)
**Priority: Low - After core infrastructure complete**

#### VLAN 70 - KVM Virtual Machines
```
Subnet: 192.168.70.0/24

Planned VMs on T640:
- Windows Server AD: 192.168.70.10
- SQL Server VMs: 192.168.70.20-29
- RHEL VMs: 192.168.70.30-39
- Test k8s clusters: 192.168.70.40-49
- DHCP pool: 192.168.70.100-199
- Nested ESXi: 192.168.70.200-209
```

#### VLAN 80 - DMZ Services
```
Subnet: 192.168.80.0/24

Future Services:
- Reverse proxy: 192.168.80.10
- External DNS: 192.168.80.53
- VPN endpoints: 192.168.80.100-109
- Public services: 192.168.80.200-220
```

## 🔧 UDM SE Configuration Tasks

### 1. VLAN Creation
```bash
# Create VLANs in UDM SE UI
VLAN 10: Management (192.168.10.0/24)
VLAN 20: Core Services (192.168.20.0/24) 
VLAN 30: OCP Compact (192.168.30.0/24)
VLAN 40: OCP SNO (192.168.40.0/24)
VLAN 50: Storage (192.168.50.0/24)
VLAN 60: Containers (192.168.60.0/24)
VLAN 70: KVM VMs (192.168.70.0/24)
VLAN 80: DMZ (192.168.80.0/24)
VLAN 90: Guest (192.168.90.0/24)
```

### 2. DHCP Configuration
- **Static reservations** for all infrastructure devices
- **DHCP pools** only where needed (management, VMs, guest)
- **DNS forwarding** to CoreDNS on VLAN 20

### 3. Firewall Rules (Security-First Approach)
```
Default: DENY ALL inter-VLAN traffic

Allow Rules:
- VLAN 10 (Management) → ALL (admin access)
- VLAN 20 (Core) → ALL (DNS, monitoring)
- VLAN 30,40 (OpenShift) → VLAN 50 (storage), VLAN 60 (containers)
- VLAN 70 (KVM) → VLAN 50 (storage), VLAN 20 (DNS)
- VLAN 60 (Containers) → VLAN 50 (storage)
- VLAN 80 (DMZ) → Specific services only
- VLAN 90 (Guest) → Internet only

Block Rules:
- Guest → Internal networks
- OpenShift clusters → Direct communication
- Storage VLAN → Initiate connections
```

### 4. QoS/Traffic Prioritization
```
High Priority:
- VLAN 50 (Storage) - Low latency for NFS/iSCSI
- VLAN 10 (Management) - Critical for operations

Medium Priority:  
- VLAN 20 (Core Services) - DNS, monitoring
- VLAN 30,40 (OpenShift) - Production workloads

Standard Priority:
- VLAN 60 (Containers) - Application services
- VLAN 70 (KVM VMs) - General workloads

Low Priority:
- VLAN 90 (Guest) - Non-critical traffic
```

## 📝 Implementation Checklist

### Pre-Implementation
- [ ] **Backup current UDM SE config**
- [ ] **Document current network state**
- [ ] **Plan maintenance windows**
- [ ] **Prepare rollback procedures**

### Phase 1 Tasks
- [ ] Create VLAN 10, 20, 50 in UDM SE
- [ ] Configure static IP reservations
- [ ] Set up DHCP pools for management
- [ ] Enable jumbo frames on VLAN 50
- [ ] Test basic connectivity
- [ ] **Schedule Beelink migration window**
- [ ] Migrate Beelink cluster: VLAN 3 → VLAN 20
- [ ] Deploy CoreDNS with lab.local zones
- [ ] Update democratic-csi for VLAN 50
- [ ] Verify storage performance

### Phase 2 Tasks  
- [ ] Create VLAN 30, 40 in UDM SE
- [ ] Configure OpenShift static IPs
- [ ] Set up MetalLB IP pools
- [ ] Deploy OpenShift compact cluster (VLAN 30)
- [ ] Deploy OpenShift SNO + worker (VLAN 40)
- [ ] Configure ACM hub on compact cluster
- [ ] Add SNO cluster as ACM spoke
- [ ] Test inter-cluster connectivity

### Phase 3 Tasks
- [ ] Create VLAN 60 in UDM SE
- [ ] Deploy Synology container services
- [ ] Configure DNS zones for services
- [ ] Test container registry access
- [ ] Set up GitOps workflows
- [ ] Deploy Splunk for log aggregation

### Phase 4 Tasks
- [ ] Create VLAN 70, 80, 90
- [ ] Configure KVM networking on T640
- [ ] Deploy Windows AD VM
- [ ] Set up SQL Server VMs
- [ ] Configure DMZ services
- [ ] Test external access patterns

## 🚨 Risk Mitigation

### Critical Risks
1. **Beelink Migration**: Could lose core k0s cluster
   - **Mitigation**: Full backup, staged migration, rollback plan
   
2. **Storage Network Change**: Could break persistent volumes
   - **Mitigation**: Test democratic-csi on VLAN 50 first
   
3. **DNS Disruption**: Could break all name resolution
   - **Mitigation**: Keep UDM SE DNS active during transition

### Testing Strategy
- **Connectivity tests** after each VLAN creation
- **Performance benchmarks** for storage network
- **Service health checks** during migrations
- **Rollback procedures** documented and tested

## 📊 Success Metrics

### Phase 1 Complete
- [ ] All devices reachable on management VLAN
- [ ] Storage network achieving >1Gbps throughput
- [ ] CoreDNS resolving .lab.local domains
- [ ] Beelink cluster stable on VLAN 20

### Phase 2 Complete
- [ ] Both OpenShift clusters operational
- [ ] ACM managing SNO cluster from compact
- [ ] Cross-cluster networking functional
- [ ] GitOps workflows active

### Phase 3 Complete
- [ ] All container services accessible
- [ ] Harbor integrated with OpenShift
- [ ] Splunk collecting cluster logs
- [ ] Development workflows functional

### Final State
- [ ] **9 VLANs operational** with proper segmentation
- [ ] **Sub-5ms latency** on storage network
- [ ] **99.9% uptime** for core services
- [ ] **Zero security violations** in firewall logs

---

**Next Action**: Begin Phase 1 VLAN creation in UDM SE interface
**Estimated Timeline**: 8 weeks for complete implementation
**Risk Level**: Medium (with proper testing and rollback plans)
