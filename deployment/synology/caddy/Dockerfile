# Build Caddy with Porkbun DNS provider for DNS-01 challenges
FROM caddy:2.8-builder AS builder
RUN xcaddy build --with github.com/caddy-dns/porkbun

FROM caddy:2.8
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
