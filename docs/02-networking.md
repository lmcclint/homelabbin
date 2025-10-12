# Homelab Networking Plan

## Overview
This document outlines the network architecture for the homelab, leveraging the Ubiquiti UDM SE for network management and segmentation.

## Updated Plan (Authoritative)
- Addressing: 10.20.0.0/16 for lab (home stays 192.168.1.0/24)
- Domain: lab.2bit.name (split-horizon)
- VLANs: 10 mgmt, 20 users, 30 servers, 40 storage, 60 lab services, 70 k8s nodes, 71 k8s LB VIPs, 80 IoT, 90 guest, 98 DMZ, 99 OOB
- DNS: Pi-hole (first hop) → Technitium (authoritative + recursive) on k8s (Technitium PVC 10Gi, default StorageClass)

## Current State
- **UDM SE**: Primary network management
- **Home Network**: VLAN 1 (192.168.1.0/24) - Default/home devices
- **Existing Lab**: VLAN 3 (192.168.3.0/24) - Beelink k0s cluster

## IP Addressing Strategy

### Current Decision: 192.168.x.x vs 10.x.x.x

**Recommendation: Continue with 192.168.x.x for consistency**

**Pros of 192.168.x.x:**
- ✅ **Consistency** with existing home network (192.168.1.0/24)
- ✅ **Simple management** - single addressing scheme
- ✅ **UDM SE familiarity** - already configured for 192.168.x.x
- ✅ **Sufficient address space** - 254 addresses per VLAN
- ✅ **Clear separation** by VLAN ID

**Cons of 192.168.x.x:**
- ⚠️ **Limited scalability** - only 254 hosts per subnet
- ⚠️ **Common conflicts** with other networks when traveling

**Alternative: 10.x.x.x (If needed for expansion)**
- **Lab networks**: 10.10.x.x through 10.100.x.x
- **More addresses**: /22 or /23 subnets for large deployments
- **Enterprise-like**: Matches many corporate environments
- **Migration path**: Can be implemented later if growth demands

**Current Plan**: Proceed with 192.168.x.x - migrate to 10.x.x.x only if hitting limitations

## Proposed VLAN Architecture

### VLAN 10 - Management Network
- **Subnet**: 192.168.10.0/24
- **Purpose**: Out-of-band management, iDRAC, hypervisor management
- **Devices**:
  - Dell T640 iDRAC: 192.168.10.10
  - Dell T640 RHEL management: 192.168.10.11
  - Synology management: 192.168.10.20
  - UNAS Pro management: 192.168.10.21
  - Cockpit/libvirt management: 192.168.10.100
- **Note**: M710q machines have no IPMI - managed via primary NICs on their respective VLANs

### VLAN 20 - Core Infrastructure Services
- **Subnet**: 192.168.20.0/24
- **Purpose**: DNS, IPAM, monitoring, certificates, load balancers
- **Devices**:
  - Beelink k0s cluster: 192.168.20.11-13
  - DNS servers (CoreDNS): 192.168.20.53, 192.168.20.54
  - IPAM (phpIPAM/NetBox): 192.168.20.80
  - Monitoring stack: 192.168.20.90-99
  - MetalLB pool: 192.168.20.200-220

### VLAN 30 - OpenShift Compact Cluster
- **Subnet**: 192.168.30.0/24
- **Purpose**: Production-like 3-node OpenShift cluster
- **Devices**:
  - OCP nodes: 192.168.30.11-13
  - API/Ingress VIP: 192.168.30.100
  - Apps wildcard: *.apps.ocp-compact.lab.2bit.name → 192.168.30.101
  - ACM console: 192.168.30.110

### VLAN 40 - OpenShift SNO + Worker
- **Subnet**: 192.168.40.0/24
- **Purpose**: Single Node OpenShift + worker testing
- **Devices**:
  - SNO node: 192.168.40.11
  - Worker node: 192.168.40.12
  - API/Ingress VIP: 192.168.40.100
  - Apps wildcard: *.apps.ocp-sno.lab.2bit.name → 192.168.40.101

### VLAN 50 - Storage Network
- **Subnet**: 192.168.50.0/24
- **Purpose**: High-speed storage traffic, NFS, iSCSI
- **Devices**:
  - Synology storage IP: 192.168.50.20
  - UNAS Pro storage IP: 192.168.50.21
  - Democratic-CSI endpoints: 192.168.50.30-39
  - KVM storage network: 192.168.50.100-109

### VLAN 60 - Container Services
- **Subnet**: 192.168.60.0/24
- **Purpose**: Harbor, Nexus, GitLab, Splunk (if on Synology)
- **Devices**:
  - Harbor registry: 192.168.60.10
  - Nexus repository: 192.168.60.11
  - GitLab/Gitea: 192.168.60.12
  - Splunk: 192.168.60.15
  - Quay registry: 192.168.60.20

### VLAN 70 - KVM Virtual Machines
- **Subnet**: 192.168.70.0/24
- **Purpose**: KVM/libvirt VMs on Dell T640 RHEL hypervisor
- **Devices**:
  - Windows Server (AD): 192.168.70.10
  - SQL Server VMs: 192.168.70.20-29
  - RHEL VMs: 192.168.70.30-39
  - Test k8s clusters: 192.168.70.40-49
  - Client VMs: 192.168.70.100-199
  - Nested ESXi VMs (testing): 192.168.70.200-209

### VLAN 80 - DMZ/External Services
- **Subnet**: 192.168.80.0/24
- **Purpose**: Externally accessible services, reverse proxy
- **Devices**:
  - Reverse proxy/HAProxy: 192.168.80.10
  - External DNS: 192.168.80.53
  - VPN endpoints: 192.168.80.100-109
  - Public service endpoints: 192.168.80.200-220

### VLAN 90 - Guest/Isolated
- **Subnet**: 192.168.90.0/24
- **Purpose**: Guest access, isolated testing
- **Devices**:
  - Guest devices: DHCP pool 192.168.90.100-199
  - Isolated test environments: 192.168.90.200-254

### VLAN 100 - Disconnected/Air-Gapped Network (Optional)
- **Subnet**: 192.168.100.0/24
- **Purpose**: Disconnected workflows, air-gapped testing, compliance scenarios
- **Devices**:
  - Disconnected OpenShift cluster: 192.168.100.11-13
  - Local registries: 192.168.100.20-29
  - Internal DNS/DHCP: 192.168.100.53
  - Test workloads: 192.168.100.100-199
- **Features**: No internet access, internal-only registry and services
- **Alternative**: Can be implemented via KVM NAT networks or isolated VMs

## Disconnected Workflow Implementation Options

### Option 1: Physical VLAN 100 (Production-like)
**Best for**: Customer demos, compliance testing, true air-gap scenarios

```
Advantages:
✅ True network isolation
✅ Realistic customer environment simulation
✅ Can test disconnected installs completely
✅ Multiple machines can participate
✅ Network-level security testing

Disadvantages:
❌ Requires additional VLAN configuration
❌ Uses physical network resources
❌ More complex to set up initially
```

### Option 2: KVM NAT Networks (Flexible Development)
**Best for**: Development testing, quick iterations, resource efficiency

```
Implementation:
- Create isolated libvirt networks on T640
- No internet gateway configuration
- Internal-only DNS and DHCP
- Can snapshot entire disconnected environments

Advantages:
✅ Quick to create/destroy
✅ Resource efficient
✅ Easy to replicate
✅ Snapshot capability
✅ No physical network changes

Disadvantages:
❌ VM-only (no bare metal testing)
❌ Limited to T640 resources
❌ Less realistic networking
```

### Option 3: Hybrid Approach (Best of Both)
**Recommended Implementation Strategy:**

1. **Start with KVM NAT** for development and testing
2. **Add VLAN 100** when needed for:
   - Customer demonstrations
   - Compliance scenarios
   - Multi-node bare metal testing
   - Red Hat certification labs

### Use Cases for Disconnected Networks

- **OpenShift Disconnected Installs**: Mirror registries, air-gap procedures
- **RHEL Satellite**: Disconnected content management
- **Compliance Testing**: Government/financial air-gap requirements
- **Customer Demos**: Realistic enterprise scenarios
- **Certification Prep**: Red Hat exam environments
- **Security Testing**: Network isolation validation

## Network Policies and Routing

### Inter-VLAN Communication Rules

#### Allowed Traffic
- **Management (VLAN 10)** → All VLANs (administrative access)
- **Core Services (VLAN 20)** → All VLANs (DNS, monitoring)
- **OpenShift VLANs (30, 40)** → Storage (VLAN 50), Container Services (VLAN 60)
- **KVM VMs (VLAN 70)** → Storage (VLAN 50), Core Services (VLAN 20)
- **Container Services (VLAN 60)** → Storage (VLAN 50)
- **DMZ (VLAN 80)** → Specific services only (reverse proxy rules)

#### Restricted Traffic
- **Guest (VLAN 90)** → Internet only, no internal access
- **OpenShift clusters** → No direct communication (use ACM for management)
- **Storage (VLAN 50)** → No initiation of connections (receive only)

### DNS Strategy

#### Internal DNS (.lab.2bit.name)
- **Primary**: Technitium on k8s (authoritative + recursive)
- **Base Domain**: `2bit.name` (owned domain)
- **Lab Subdomain**: `lab.2bit.name` (internal lab services)
- **Zones**:
  - `lab.2bit.name` - General internal services
  - `ocp-compact.lab.2bit.name` - Compact OpenShift cluster
  - `ocp-sno.lab.2bit.name` - SNO OpenShift cluster
  - `kvm.lab.2bit.name` - KVM hypervisor infrastructure
  - `storage.lab.2bit.name` - Storage services

#### External DNS & Split-Brain
- **Public domain**: `2bit.name` (owned)
- **Split-brain**: Internal vs external views of `lab.2bit.name`
- **External access**: Selective services via reverse proxy
- **Benefits**: Real domain, SSL certificates, external connectivity

### DNS Strategy Benefits

#### Using .lab.2bit.name (Real Domain)
- ✅ **Valid SSL certificates** via Let's Encrypt/ACME
- ✅ **Split-brain DNS** capability (internal vs external views)
- ✅ **External access** to select services (via DMZ)
- ✅ **Professional setup** for customer demonstrations
- ✅ **No .local limitations** (mDNS conflicts, certificate issues)
- ✅ **Subdomain organization**: 
  - `ocp-compact.lab.2bit.name` - Production-like cluster
  - `ocp-sno.lab.2bit.name` - Edge/testing cluster
  - `harbor.lab.2bit.name` - Container registry
  - `git.lab.2bit.name` - Source code management
  - `storage.lab.2bit.name` - Storage services
  - `kvm.lab.2bit.name` - Virtualization management

## Load Balancing Strategy

### MetalLB Configuration
- **Core Services Pool**: 192.168.20.200-220
- **OpenShift Compact Pool**: 192.168.30.200-220
- **OpenShift SNO Pool**: 192.168.40.200-220
- **Container Services Pool**: 192.168.60.200-220

### HAProxy/Nginx Reverse Proxy
- **Location**: VLAN 80 (DMZ)
- **Purpose**: External access to internal services
- **SSL termination**: Let's Encrypt + internal CA

## Network Services

### DHCP Pools
- **Management**: 192.168.10.150-199 (temporary access)
- **Core Services**: 192.168.20.150-199 (DHCP reservations)
- **OpenShift networks**: Static IPs only
- **KVM VMs**: 192.168.70.100-199
- **Guest**: 192.168.90.100-199

### NTP
- **Primary**: Core services cluster (VLAN 20)
- **Secondary**: UDM SE

### Monitoring
- **SNMP**: UDM SE → Prometheus (VLAN 20)
- **Flow monitoring**: All VLANs → Grafana

## Implementation Notes

### Migration from Current State
1. **Keep existing VLAN 3** during transition
2. **Migrate Beelink cluster** to VLAN 20 during maintenance window
3. **Update democratic-csi** storage network configuration
4. **Test connectivity** between all VLANs before going live

### UDM SE Configuration
- **Enable inter-VLAN routing** with firewall rules
- **Configure DHCP reservations** for static devices
- **Set up DNS forwarding** to CoreDNS
- **Enable traffic analytics** for monitoring
- **Configure port forwarding** for external access

### Security Considerations
- **Firewall rules**: Deny by default, allow specific traffic
- **VPN access**: WireGuard for remote management
- **Certificate management**: Internal CA for lab services
- **Network monitoring**: Traffic analysis and alerting

## Future Considerations

### 10Gbps Network Optimization
- **Jumbo frames**: Enable for storage VLANs
- **Link aggregation**: Consider for high-traffic devices
- **QoS**: Prioritize storage and management traffic

### Scalability
- **Additional subnets**: /23 or /22 if more IPs needed
- **VLAN expansion**: Room for additional clusters
- **IPv6**: Future dual-stack implementation

---
*This networking plan supports the overall homelab architecture and can be adjusted based on specific requirements.*

