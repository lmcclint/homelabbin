# Homelab

> A comprehensive homelab for experimenting with OpenShift, Kubernetes, container technologies, and enterprise integration scenarios.

## 🎯 Project Goals

- **Multi-cluster OpenShift management** with ACM
- **Hybrid cloud scenarios** (bare metal + virtualized + containerized)
- **Enterprise integration testing** (AD, SQL Server, SIEM)
- **Customer environment replication**
- **Certification preparation**
- **GitOps and automation workflows**

## 📋 Documentation (Priority Order)

### Phase 1: Foundation
1. **[Overview & Hardware Allocation](./docs/01-overview.md)** - Complete architecture overview
2. **[Networking Plan](./docs/02-networking.md)** - VLAN strategy and network design
3. **[Core Services Setup](./docs/03-core-services.md)** *(TBD)* - k0s cluster, DNS, monitoring
4. **[Storage Configuration](./docs/04-storage.md)** *(TBD)* - Synology, democratic-csi

### Phase 2: Container Platform
5. **[Container Registry Setup](./docs/05-container-registry.md)** *(TBD)* - Harbor deployment
6. **[Git Repository Setup](./docs/06-git-repository.md)** *(TBD)* - Gitea/GitLab on Synology
7. **[Artifact Repository](./docs/07-artifact-repository.md)** *(TBD)* - Nexus/Artifactory

### Phase 3: OpenShift Clusters
8. **[OpenShift Compact Cluster](./docs/08-openshift-compact.md)** *(TBD)* - 3-node production-like
9. **[OpenShift SNO + Worker](./docs/09-openshift-sno.md)** *(TBD)* - Edge computing setup
10. **[ACM Configuration](./docs/10-acm-setup.md)** *(TBD)* - Multi-cluster management

### Phase 4: vSphere Platform
11. **[vSphere Installation](./docs/11-vsphere-setup.md)** *(TBD)* - Dell T640 virtualization
12. **[Windows Infrastructure](./docs/12-windows-infrastructure.md)** *(TBD)* - AD, SQL Server
13. **[VM Templates & Automation](./docs/13-vm-automation.md)** *(TBD)* - Template creation

### Phase 5: Advanced Services
14. **[Monitoring & Observability](./docs/14-monitoring.md)** *(TBD)* - Prometheus, Grafana, Splunk
15. **[Security & Compliance](./docs/15-security.md)** *(TBD)* - ACS, certificates, auditing
16. **[Backup & DR](./docs/16-backup-dr.md)** *(TBD)* - Backup strategies

## 🚀 Quick Start Deployment

### Prerequisites
- Ubiquiti UDM SE configured
- Hardware powered and networked
- Initial VLAN setup (see [networking plan](./docs/02-networking.md))

### Deployment Scripts

```bash
# Phase 1: Foundation
./scripts/01-network-setup.sh
./scripts/02-core-services-deploy.sh

# Phase 2: Container Platform  
./deployment/synology/docker-compose.yml  # Gitea, Nexus, Splunk
./scripts/03-storage-setup.sh

# Phase 3: OpenShift
./deployment/openshift/compact-cluster/
./deployment/openshift/sno-cluster/

# Phase 4: vSphere
./deployment/vsphere/
```

## 📁 Repository Structure

```
homelab-plan/
├── README.md                    # This file
├── docs/                        # Documentation (numbered by priority)
│   ├── 01-overview.md
│   ├── 02-networking.md
│   └── ...
├── deployment/                  # Deployment configurations
│   ├── synology/               # Docker Compose files for Synology
│   ├── k0s/                    # k0s cluster manifests
│   ├── openshift/              # OpenShift installation configs
│   └── vsphere/                # vSphere automation
├── scripts/                     # Automation scripts
└── .gitignore                   # Excludes sensitive data
```

## 🔧 Technology Stack

### Infrastructure
- **Networking**: Ubiquiti UDM SE
- **Storage**: Synology DS1621+, Ubiquiti UNAS Pro
- **Compute**: 3x Beelink S12 Pro, 5x Lenovo M710q, Dell T640

### Container Platforms
- **OpenShift 4.14+**: Two clusters (compact + SNO)
- **k0s**: Core services cluster
- **vSphere**: VM workloads

### Core Services
- **Container Registry**: Nexus
- **Git Repository**: Gitea/GitLab CE
- **Artifact Repository**: Nexus/Artifactory
- **Monitoring**: Prometheus + Grafana
- **Logging**: Splunk Enterprise
- **Secrets**: HashiCorp Vault CE (Synology)
- **DNS**: Technitium DNS (on k8s)
- **Load Balancing**: MetalLB + HAProxy

## 🔐 Security Notes

- **No sensitive data** is stored in this repository
- **Secrets management** via external-secrets-operator
- **Certificate management** via cert-manager
- **Network segmentation** via VLANs and firewall rules

## 🤝 Contributing

This is a personal homelab project, but feel free to:
- Submit issues for questions or suggestions
- Fork for your own homelab adaptations
- Share improvements via pull requests

## 📞 Next Steps

1. **Create GitHub repository** for version control
2. **Start with Phase 1** foundation setup
3. **Build deployment automation** as we go
4. **Document lessons learned** for future reference

---

**Status**: 🚧 Planning & Initial Development

**Last Updated**: June 2025

