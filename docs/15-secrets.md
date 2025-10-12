# Secrets Management (HashiCorp Vault CE)

Goal: Centralize secrets for lab services and Kubernetes without introducing bootstrap dependencies.

Placement (initial)
- Synology (Docker) single-node Vault using Raft storage
- Host IP: 10.20.60.14 (VLAN 60)
- DNS: vault.lab.2bit.name → 10.20.60.14
- Port: 8200/TCP (API/UI)

Docker Compose (already added)
- Service: vault
- Volumes: vault-data (raft), vault-config (vault.hcl), vault-tls (certs)
- Command: server -config=/vault/config/vault.hcl

Recommended vault.hcl (place in vault-config volume)
```hcl
listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_disable   = 0
  tls_cert_file = "/vault/tls/fullchain.pem"
  tls_key_file  = "/vault/tls/privkey.pem"
}

storage "raft" {
  path   = "/vault/data"
  node_id = "vault-raft-1"
}

api_addr     = "https://vault.lab.2bit.name:8200"
cluster_addr = "https://10.20.60.14:8201"

ui = true

disable_mlock = true
```

TLS
- Use TLS from day one.
- Option A: temporary self-signed; Option B: internal CA (preferred) when available.
- Copy fullchain.pem and privkey.pem to vault-tls volume.

Initialize & unseal (once)
- Initialize with Shamir (e.g., 3 shares / threshold 2)
- Store unseal key shares offline (password manager + printed copy)
- Store initial root token offline; create an admin user + policy, then avoid using root

Auth methods (enable)
- kubernetes (for k0s workloads via External Secrets Operator)
- userpass (admin/testing)
- approle (non-human services off-cluster)

Policies (examples)
- eso-read: allow read on kv paths used by ESO
- service-specific: read-only scopes for Nexus, Gitea, Splunk, etc.

Key/Value engine
- Enable kv-v2 at path "secret" (default in examples)
- Suggested paths:
  - secret/gitea/db: { password }
  - secret/nexus/admin: { password }
  - secret/splunk/admin: { password }
  - secret/k8s/registry: { username, password }

Kubernetes integration (later, on k0s)
- Install External Secrets Operator
- In Vault, configure kubernetes auth:
  - Set k8s API address and CA
  - Create a role (eso-read) mapping a ServiceAccount (eso-sa) to policy eso-read
- Create ClusterSecretStore:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault
spec:
  provider:
    vault:
      server: https://vault.lab.2bit.name:8200
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: eso-read
          serviceAccountRef:
            name: eso-sa
            namespace: external-secrets
```

ExternalSecret example:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: gitea-db-credentials
  namespace: apps
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault
    kind: ClusterSecretStore
  target:
    name: gitea-db-credentials
  data:
    - secretKey: password
      remoteRef:
        key: secret/data/gitea/db
        property: password
```

Operational notes
- Firewall: restrict 8200 to admin VLAN (20) and k0s nodes if needed
- Backups: Snapshot vault-data regularly; keep unseal shares offline
- Migration path: Later consider moving to k0s with 3-node Raft for HA

Backups & DR (Raft snapshots)
- Snapshot creation (from host using the running container):
  ```bash
  # Create a snapshot inside the container
  docker exec vault vault operator raft snapshot save /tmp/vault-raft.snap
  # Copy it out to the host (adjust container name if needed)
  docker cp vault:/tmp/vault-raft.snap ./backups/vault-raft-$(date +%Y%m%d%H%M).snap
  ```
- Store snapshots off-box (another NAS or cloud).
- Never commit snapshots or unseal keys to git.

Recovery test (recommended quarterly)
1) Stop the production container (or perform on an isolated test host).
2) Start a temporary Vault container with a fresh data volume.
3) Copy a snapshot into the temp container and restore:
   ```bash
   docker cp ./backups/vault-raft-YYYYMMDDHHMM.snap temp_vault:/tmp/snap.snap
   docker exec -it temp_vault sh -lc "vault operator raft snapshot restore /tmp/snap.snap"
   ```
4) Unseal with copies of your unseal shares.
5) Verify policies, auth methods, and a sample secret are present.
6) Destroy the temp instance/volume when done.

Sensitive material handling
- Unseal shares and root/recovery tokens must be kept offline (password manager + printed copy in a safe).
- TLS private keys are never committed; they live only on the host (mounted at runtime).
- Use policies with least privilege; avoid day-to-day use of root token.
