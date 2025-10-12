# Vault on Synology (Docker) - Safe bootstrap

This folder contains a sample configuration for running Vault CE on Synology via the main docker-compose. No secrets are stored here. TLS certs/keys are not committed and are ignored by .gitignore.

Service details
- Service name: vault (docker-compose)
- IP/Port: 10.20.60.14:8200 (HTTPS)
- Storage: Raft (persistent in a named volume)
- DNS: vault.lab.2bit.name

Files in this folder
- vault.hcl.sample: Example configuration with placeholders (no secrets)
- README.md: This file

How to provide TLS and config without committing secrets
Option A: Override file binds (recommended)
1) Create deployment/synology/docker-compose.override.yml (ignored by git):
   ```yaml
   services:
     vault:
       volumes:
         - /path/on/synology/vault/config:/vault/config:ro
         - /path/on/synology/vault/tls:/vault/tls:ro
   ```
2) On the Synology host, place:
   - /path/on/synology/vault/config/vault.hcl (based on vault.hcl.sample)
   - /path/on/synology/vault/tls/fullchain.pem and privkey.pem
3) Start: `docker compose -f deployment/synology/docker-compose.yml up -d vault`

Option B: Copy into named volumes
1) Start once to create volumes: `docker compose -f deployment/synology/docker-compose.yml up -d vault` (it may not be healthy yet)
2) Copy files:
   ```bash
   # Find container name
   docker ps | findstr vault
   # Copy config and TLS into the running container
   docker cp deployment/synology/vault/vault.hcl.sample <container>:/vault/config/vault.hcl
   docker cp <your-fullchain.pem> <container>:/vault/tls/fullchain.pem
   docker cp <your-privkey.pem> <container>:/vault/tls/privkey.pem
   # Restart
   docker compose -f deployment/synology/docker-compose.yml restart vault
   ```

Initialization checklist (do this once)
- Initialize: 3 shares / threshold 2 (or your preference)
- Unseal with threshold shares
- Store unseal keys and initial root token offline (password manager + printed copy)
- Enable auth methods (kubernetes, userpass, approle)
- Create policies (eso-read, service-specific)
- Enable kv-v2 at path `secret`

Security notes
- .gitignore already excludes *.pem, *.key, .env, etc.
- Do not commit vault.hcl with real paths or any tokens; keep host-specific overrides in docker-compose.override.yml.
- Restrict access to 8200 in the UDM firewall to admin VLAN and k0s nodes as needed.
