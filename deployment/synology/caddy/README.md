# Caddy reverse proxy (Synology, macvlan)

Fronts Gitea (and later other services) with a single entrypoint and TLS.

## Why use a reverse proxy instead of ports per app?

- Single ingress and DNS: consolidate to 80/443 with hostnames (git.example, nexus.example) instead of odd ports. Easier for users and certificates.
- TLS automation: Caddy can issue and renew certs automatically (Let's Encrypt for public names or `tls internal` for LAN). You avoid per-app TLS config.
- Security controls: centralized headers, HSTS, rate limits, timeouts, IP allow/deny, and hiding backend topology.
- Protocol features: HTTP/2/3, compression, caching and proper proxy headers across all apps consistently.
- Flexibility: route by hostname or path; swap/upscale backends without changing client URLs.
- Synology/macvlan reality: host cannot talk to macvlan containers directly; a proxy on the same macvlan is a clean way to front multiple services.

## Usage

1. Copy env and edit:

```bash
cd deployment/synology/caddy
cp .env.example .env
vi .env   # set TZ, parent interface, CADDY_IP, GITEA_FQDN, TLS_MODE
```

- If `GITEA_FQDN` is only on your LAN, leave `TLS_MODE=internal`.
- For public DNS with inbound 80/443 open, set `TLS_MODE` to an email address to use ACME.

2. DNS
- Create a DNS A record for `GITEA_FQDN` pointing to `CADDY_IP`.

3. Start
```bash
docker compose up -d
```

4. Gitea settings
- In the Gitea stack, set `GITEA__server__ROOT_URL=https://<GITEA_FQDN>/` (and DOMAIN/SSH_DOMAIN accordingly) so web links are correct.

## Files
- `docker-compose.yml` — Caddy on macvlan with static IP.
- `Caddyfile` — reverse proxies `GITEA_FQDN` to the Gitea container’s HTTP port.
- `.env.example` — network and TLS knobs.
