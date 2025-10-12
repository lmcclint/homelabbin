# S3 Object Storage (GarageHQ) on Synology

Goal
- Provide on-prem S3-compatible object storage for backups (e.g., Velero/Restic later), artifacts, and general lab needs.
- Prefer GarageHQ over MinIO (recent reliability concerns).

Placement
- Host: Synology (Docker)
- IP: 10.20.60.16 (VLAN 60)
- DNS: garage.lab.2bit.name → 10.20.60.16
- Ports: S3 API and admin UI as configured by Garage (set in config)

Design notes
- Start with single-node Garage on Synology (simple). Consider multi-node later if you add more storage nodes.
- Persist data and config on named volumes; snapshot on the NAS.
- Store S3 access/secret keys in Vault; never in git.
- Optionally front with a reverse proxy for TLS; or terminate TLS directly in Garage.

Docker Compose (scaffold – validate image/tag)
- Add later to the main compose once you confirm the official image name/tag. Suggested structure:
```yaml
# Example scaffold only – do not apply as-is without validating image and ports
services:
  garage:
    image: ghcr.io/garagehq/garage:latest   # validate this tag/name first
    container_name: garage
    restart: unless-stopped
    networks:
      - garage-network
    # Configure port maps per your Garage config (S3/API/admin)
    # ports:
    #   - "10.20.60.16:3900:3900"  # Example S3 API port (validate)
    #   - "10.20.60.16:3901:3901"  # Example admin port (validate)
    environment:
      # Use files/volumes for secrets – do not embed here
      - RUST_LOG=info
    volumes:
      - garage-data:/var/lib/garage
      - garage-config:/etc/garage

networks:
  garage-network:
    driver: bridge

volumes:
  garage-data:
  garage-config:
```

Initial configuration
- Generate S3 access/secret keys; store in Vault at secret/garage/admin (for example).
- Create buckets for:
  - backups (vault-snapshots, future velero, restic, etc.)
  - artifacts (CI/CD artifacts, binary caches)
  - app-data (as needed)
- Define bucket policies as needed (private by default).

DNS and firewall
- DNS: garage.lab.2bit.name → 10.20.60.16
- UDM firewall: allow admin VLAN (20) to Garage API; block other VLANs except where explicitly needed.

Backups
- Snapshot garage-data regularly on Synology.
- Consider periodic object-level parity or replication later if you add nodes.

Next steps
1) Reserve 10.20.60.16 and create DNS A record garage.lab.2bit.name.
2) Confirm the official Garage image/tag and default ports.
3) Add a validated service block to the main docker-compose (with correct ports and volumes).
4) Generate S3 credentials and store in Vault.
5) Create initial buckets and test S3 access from a client.
