# Gitea on Synology (split deployment)

This folder contains a standalone Gitea + Postgres stack.

## Usage

1. Copy the environment template and edit values (passwords, TZ, network IP, and public URLs):
   
   ```bash
   cp .env.example .env
   ```

2. Optionally change data paths in `.env` to Synology shared folders (e.g., `/volume1/docker/gitea` and `/volume1/docker/gitea-postgres`).

3. Start:
   
   ```bash
   docker compose up -d
   ```

4. Access Gitea at `http://10.20.60.11:3000/` (or whatever `GITEA_IP` you set) initially. After Caddy is up, use `https://git.lab.2bit.name/`.

## Networking

- This stack uses a macvlan network (`vlan60`) with a static IP for the Gitea container and a separate internal bridge network between Gitea and Postgres.
- Ensure `GITEA_IP` is free on your LAN. If the old monolithic stack is still running, verify no other service uses that IP.
- Synology/macvlan caveat: the host cannot directly reach containers on macvlan. Access Gitea from other devices on the LAN, or use Caddy as the reverse proxy.

## DNS and SSH

- Create DNS A records:
  - `git.lab.2bit.name` -> CADDY_IP (Caddy’s macvlan IP)
  - `git-ssh.lab.2bit.name` -> `GITEA_IP` (Gitea’s macvlan IP) for SSH
- In `.env`, set:
  - `GITEA_ROOT_URL=https://git.lab.2bit.name/`
  - `GITEA_DOMAIN=git.lab.2bit.name`
  - `GITEA_SSH_DOMAIN=git-ssh.lab.2bit.name`

## Notes

- Use this split stack for Gitea; the monolithic compose has had Gitea removed to avoid conflicts.
