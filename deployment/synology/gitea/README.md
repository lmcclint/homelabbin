# Gitea on Synology (split deployment)

This folder contains a standalone Gitea + Postgres stack.

## Usage

1. Copy the environment template and edit values (especially passwords, TZ, and network IP):
   
   ```bash
   cp .env.example .env
   ```

2. Optionally change data paths in `.env` to Synology shared folders (e.g., `/volume1/docker/gitea` and `/volume1/docker/gitea-postgres`).

3. Start:
   
   ```bash
   docker compose up -d
   ```

4. Access Gitea at `http://10.20.60.11:3000/` (or whatever `GITEA_IP` you set). First run will prompt for setup; DB settings are already provided via environment.

## Networking

- This stack uses a macvlan network (`vlan60`) with a static IP for the Gitea container and a separate internal bridge network between Gitea and Postgres.
- Ensure `GITEA_IP` is free on your LAN. If the old monolithic stack is still running, verify no other service uses that IP.
- Synology/macvlan caveat: the host cannot directly reach containers on macvlan. Access Gitea from other devices on the LAN, or set up a reverse proxy (Caddy) on the host network to front it.

## Notes

- When we add Caddy, we’ll connect it to the same `vlan60` network and set Gitea’s `ROOT_URL` accordingly.
- The main combined compose at `deployment/synology/docker-compose.yml` still contains a Gitea service; use only one stack at a time to avoid conflicts. We can remove it once you confirm the split.
