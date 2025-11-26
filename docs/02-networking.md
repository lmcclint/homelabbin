# Homelab Networking Plan

## Overview
This document outlines the network architecture for the homelab, leveraging the Ubiquiti UDM SE for network management and segmentation.

## Updated Plan (Authoritative)
- Addressing: 10.20.0.0/16 for lab (home stays 192.168.1.0/24)
- Domain: lab.2bit.name (split-horizon)
- VLANs: 10 mgmt, 20 core services, 30 OCP compact, 40 OCP SNO, 50 storage, 60 container services, 70 KVM VMs, 80 DMZ, 90 guest, 100 disconnected (optional), 99 OOB
- DNS: Pi-hole (first hop) → Technitium (authoritative + recursive) on k8s (Technitium PVC 10Gi, default StorageClass)

## Current State
- **UDM SE**: Primary network management
- **Home Network**: VLAN 1 (192.168.1.0/24) - Default/home devices
- **Existing Lab**: VLAN 3 (192.168.3.0/24) - Beelink k0s cluster

## IP Addressing Strategy

### Decision: Use 10.20.0.0/16 for the lab

**Pros of 10.x.x.x:**
- ✅ Avoids overlap with common 192.168.x.x networks
- ✅ Scales easily with /24 per VLAN and room to expand
- ✅ Enterprise-aligned addressing
- ✅ Clear separation from the home network (192.168.1.0/24)

**Plan:**
- Supernet: 10.20.0.0/16
- Per-VLAN /24s using VLAN ID as third octet (e.g., VLAN 10 → 10.20.10.0/24)
- Migrate any remaining 192.168.x.x lab segments during maintenance windows

## Proposed VLAN Architecture

### VLAN 10 - Management Network
- **Subnet**: 10.20.10.0/24
- **Purpose**: Out-of-band management, iDRAC, hypervisor management
- **Access**: Restricted to Admin Jumpbox/VPN only
- **Devices**:
  - Dell T640 iDRAC: 10.20.10.10
  - Dell T640 RHEL management: 10.20.10.11
  - Synology management: 10.20.10.20
  - UNAS Pro management: 10.20.10.21
  - Cockpit/libvirt management: 10.20.10.100
- **Note**: M710q machines have no IPMI - managed via primary NICs on their respective VLANs

### VLAN 20 - Core Infrastructure Services (Mgmt Cluster)
- **Subnet**: 10.20.20.0/24
- **Purpose**: Dedicated Management Cluster (Beelinks) running DNS, IPAM, Monitoring. No generic workloads.
- **Devices**:
  - Beelink k0s cluster nodes: 10.20.20.11-13
  - DNS servers (CoreDNS/Technitium VIP): 10.20.20.53, 10.20.20.54
  - IPAM (phpIPAM or NetBox): 10.20.20.80
  - Monitoring stack: 10.20.20.90-99
  - MetalLB pool: 10.20.20.200-220

### VLAN 30 - OpenShift Compact Cluster
- **Subnet**: 10.20.30.0/24
- **Purpose**: Production-like 3-node OpenShift cluster
- **Devices**:
  - OCP nodes: 10.20.30.11-13
  - API/Ingress VIP: 10.20.30.100
  - Apps wildcard: *.apps.ocp-compact.lab.2bit.name → 10.20.30.101
  - ACM console: 10.20.30.110

### VLAN 40 - OpenShift SNO + Worker
- **Subnet**: 10.20.40.0/24
- **Purpose**: Single Node OpenShift + worker testing
- **Devices**:
  - SNO node: 10.20.40.11
  - Worker node: 10.20.40.12
  - API/Ingress VIP: 10.20.40.100
  - Apps wildcard: *.apps.ocp-sno.lab.2bit.name → 10.20.40.101

### VLAN 50 - Storage Network
- **Subnet**: 10.20.50.0/24
- **Purpose**: High-speed storage traffic (NFS, iSCSI). Layer 2 ONLY.
- **Routing**: **NO GATEWAY**. Isolated traffic.
- **MTU**: 9000 (Jumbo Frames) - Must be enabled end-to-end.
- **Strategy**: Multi-homed. Compute nodes have a dedicated virtual/physical interface on this VLAN.
- **Devices**:
  - Synology storage IP: 10.20.50.20
  - UNAS Pro storage IP: 10.20.50.21
  - Democratic-CSI endpoints: 10.20.50.30-39
  - KVM storage network: 10.20.50.100-109

### VLAN 60 - Container Services
- **Subnet**: 10.20.60.0/24
- **Purpose**: Harbor, Nexus, GitLab, Splunk (if on Synology)
- **Devices**:
  - Harbor registry: 10.20.60.10
  - Nexus repository: 10.20.60.11
  - GitLab/Gitea: 10.20.60.12
  - Splunk: 10.20.60.15
  - Quay registry: 10.20.60.20

### VLAN 70 - KVM Virtual Machines
- **Subnet**: 10.20.70.0/24
- **Purpose**: General purpose Compute/VMs on Dell T640 RHEL hypervisor.
- **Devices**:
  - Windows Server (AD): 10.20.70.10
  - SQL Server VMs: 10.20.70.20-29
  - RHEL VMs: 10.20.70.30-39
  - Test k8s clusters: 10.20.70.40-49
  - Client VMs: 10.20.70.100-199
  - Nested ESXi VMs (testing): 10.20.70.200-209

### VLAN 80 - DMZ/External Services
- **Subnet**: 10.20.80.0/24
- **Purpose**: Externally accessible services, reverse proxy
- **Devices**:
  - Reverse proxy/HAProxy: 10.20.80.10
  - External DNS: 10.20.80.53
  - VPN endpoints: 10.20.80.100-109
  - Public service endpoints: 10.20.80.200-220

### VLAN 90 - Guest/Isolated
- **Subnet**: 10.20.90.0/24
- **Purpose**: Guest access, isolated testing
- **Devices**:
  - Guest devices: DHCP pool 10.20.90.100-199
  - Isolated test environments: 10.20.90.200-254

### VLAN 100 - Disconnected/Air-Gapped Network (Optional)
- **Subnet**: 10.20.100.0/24
- **Purpose**: Disconnected workflows, air-gapped testing, compliance scenarios
- **Devices**:
  - Disconnected OpenShift cluster: 10.20.100.11-13
  - Local registries: 10.20.100.20-29
  - Internal DNS/DHCP: 10.20.100.53
  - Test workloads: 10.20.100.100-199
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
- **Core Services Pool**: 10.20.20.200-220
- **OpenShift Compact Pool**: 10.20.30.200-220
- **OpenShift SNO Pool**: 10.20.40.200-220
- **Container Services Pool**: 10.20.60.200-220

### HAProxy/Nginx Reverse Proxy
- **Location**: VLAN 80 (DMZ)
- **Purpose**: External access to internal services
- **SSL termination**: Let's Encrypt + internal CA

## Network Services

### DHCP Pools
- **Management**: 10.20.10.150-199 (temporary access)
- **Core Services**: 10.20.20.150-199 (DHCP reservations)
- **OpenShift networks**: Static IPs only
- **KVM VMs**: 10.20.70.100-199
- **Guest**: 10.20.90.100-199

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

