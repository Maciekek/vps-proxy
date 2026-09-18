# vps-proxy

One Caddy for the whole VPS. Terminates TLS (Let's Encrypt, automatic) and routes each
domain to the right container. Built on [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy):
Caddy reads Docker labels, so **this stack has no knowledge of any app** and apps have no
knowledge of each other.

## Install (once per server)

```bash
mkdir -p ~/proxy && cd ~/proxy
curl -fsSLO https://raw.githubusercontent.com/Maciekek/vps-proxy/main/docker-compose.yml
curl -fsSLO https://raw.githubusercontent.com/Maciekek/vps-proxy/main/Caddyfile
curl -fsSLO https://raw.githubusercontent.com/Maciekek/vps-proxy/main/update.sh
curl -fsSL  https://raw.githubusercontent.com/Maciekek/vps-proxy/main/.env.example -o .env
chmod +x update.sh
nano .env            # ACME_EMAIL
docker compose up -d
```

This creates the Docker network `web`. Ports 80 and 443 belong to this stack only.

## Plugging an app in

In the app's own `docker-compose.yml`, nothing else is needed on the proxy side:

```yaml
services:
  myapp:
    image: ghcr.io/me/myapp:latest
    networks: [web]
    labels:
      caddy: ${DOMAIN}
      caddy.import: secure
      caddy.reverse_proxy: "{{upstreams 3000}}"
      # optional www -> apex redirect
      caddy_1: www.${DOMAIN}
      caddy_1.redir: https://${DOMAIN}{uri} permanent

networks:
  web:
    external: true
```

`DOMAIN` lives in the app's `.env`. `{{upstreams 3000}}` resolves to the container's IP on
the `web` network, so the app must not publish that port on the host. Caddy picks the change
up within seconds of `docker compose up -d`; certificates are issued on first request once
DNS points at the server.

`secure` is the snippet from `Caddyfile` (gzip, HSTS, nosniff, referrer policy). Apps that
need different headers add their own labels, e.g. `caddy.header.X-Frame-Options: DENY`.

## Updating

```bash
./update.sh
```

## Debugging

```bash
docker compose logs -f                                   # config reloads, ACME
docker exec proxy cat /config/caddy/Caddyfile.autosave   # generated config
```
