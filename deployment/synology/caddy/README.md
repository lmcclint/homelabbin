# Caddy reverse proxy (Synology, macvlan)


Test host
- Add DNS A: whoami.lab.2bit.name -> CADDY_IP, then visit https://whoami.lab.2bit.name to verify TLS and routing.
Fronts Gitea (and later other services) with a single entrypoint and TLS.

## Why use a reverse proxy instead of ports per app?

- Single ingress and DNS: consolidate to 80/443 with hostnames (git.example, nexus.example) instead of odd ports. Easier for users and certificates.
- TLS automation: Caddy can issue and renew certs automatically (Let's Encrypt via DNS-01 with Porkbun, or `tls internal` for LAN). You avoid per-app TLS config.
- Security controls: centralized headers, HSTS, rate limits, timeouts, IP allow/deny, and hiding backend topology.
- Protocol features: HTTP/2/3, compression, caching and proper proxy headers across all apps consistently.
- Flexibility: route by hostname or path; swap/upscale backends without changing client URLs.
- Synology/macvlan reality: host cannot talk to macvlan containers directly; a proxy on the same macvlan is a clean way to front multiple services.

## Usage

1. Copy env and edit:

```bash
cd deployment/synology/caddy
cp .env.example .env
vi .env   # set TZ, interface, CADDY_IP, ACME_EMAIL, PORKBUN_* keys, and upstreams
```

- For public certs: set `ACME_EMAIL` and provide `PORKBUN_API_KEY` + `PORKBUN_API_SECRET_KEY` (scoped to DNS edits on 2bit.name). No public ports on Synology are required with DNS-01.

2. DNS
- Create an internal DNS A record for `git.lab.2bit.name` -> `CADDY_IP` (and any other hostnames you plan to proxy). Public A records are not required for issuance with DNS-01, but Porkbun must host the public zone so TXT records can be created by ACME.

3. Start
```bash
docker compose up -d --build
```

4. Gitea settings
- In the Gitea stack, set `GITEA__server__ROOT_URL=https://git.lab.2bit.name/` (and DOMAIN/SSH_DOMAIN accordingly) so web links are correct.

## Files
- `Dockerfile` — builds Caddy with Porkbun DNS plugin.
- `docker-compose.yml` — Caddy on macvlan with static IP.
- `Caddyfile` — wildcard vhost for `*.lab.2bit.name`, routes `git.lab.2bit.name` to Gitea.
- `.env.example` — network, ACME email, Porkbun API keys, and upstreams.
