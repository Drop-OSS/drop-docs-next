---
title: Exposing your instance
---

Exposing your instance allows it to be accessible from other computers than the one you're hosting it on.

## Setting `EXTERNAL_URL`

Drop uses `EXTERNAL_URL` for creating invitation links, OIDC redirects, and everything else. It should be passed as an environment variable, and include the protocol, ip/domain, and port (if applicable). Examples include:

- `http://192.168.0.100:3000`
- `https://drop.my.domain/`
- `http://drop.home.arpa:3000`

Drop automatically parses and formats the URL, so there are no requirements on the format, as long as it is a valid URL.

## LAN

The `compose.yaml` provided in the [Quickstart guide](/admin/quickstart/) already exposes the Drop instance on port 3000. If you're on the same LAN as your Drop instance, you can find it's IP and then use:

```
http://[instance IP]:3000
```

as the connection URL when setting up your Drop client.

## Reverse Proxy


If you're hosting more than one web service, or you want your Drop web instance to be accessible from `www.yourdomain.tld`, you'll need a reverse proxy. A reverse proxy is a software that sits on ports `80 (HTTP)` and `443 (HTTPS)` and "gives" the connection to the correct service
based on your configuration. There are many to choose from, but we will use Caddy as an example. When using a reverse proxy, it's best practice to use `expose:` instead of `ports:` in your docker compose, to eliminate open ports on the host machine. `expose:` only opens the port inside
the container to it's internal docker network.

### Docker Compose
```yaml compose.yml
services:
  postgres:
    image: postgres:14-alpine
    healthcheck:
      test: pg_isready -d drop -U drop
      interval: 30s
      timeout: 60s
      retries: 5
      start_period: 10s
    volumes:
      - ./db:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=drop
      - POSTGRES_USER=drop
      - POSTGRES_DB=drop
  drop:
    image: ghcr.io/drop-oss/drop:0.4.0-rc-3
    depends_on:
      postgres:
        condition: service_healthy
    expose:
      - 3000:3000
    volumes:
      - ./library:/library
      - ./data:/data
    environment:
      - DATABASE_URL=postgres://drop:drop@postgres:5432/drop
      - EXTERNAL_URL=http://localhost:3000 # default, customise if accessing from another computer or behind a reverse proxy

   caddy: # Reverse Proxy
    image: caddy:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./caddy-data/etc/caddy:/etc/caddy
      - ./caddy-data/data:/data
      - ./caddy-data/config:/config
```

Finally, create a file at `./caddy-data/etc/caddy/Caddyfile` with the following contents
###Caddyfile
```Caddyfile
drop.mydomainname.tld {
  reverse_proxy drop:3000
}
```

## VPN

This is not a detailed guide for exposing Drop to a VPN. There are far too many approaches that depend on your use case. The simplest way to put Drop **entirely** behind a VPN is to use:
```compose.yml
services:
  drop:
    ...
    network_mode: service:vpn
    ...
  vpn:
    image: # your vpn of choice...
    ...
```
**Note:** this requires removing the `ports`, `expose`, and `networks` sections of Drop on the `compose.yml`. This routes all network traffic through the VPN, which may introduce more routing complexities to the client. Point the client to the internal VPN of your `vpn:3000` service.

A slightly more advanced and more optioned approach is to expose an entire subnet or vlan on your router or the internal docker network using Tailscale/[Headscale](https://github.com/juanfont/headscale). Read more [about Tailscale Subnet Routers](https://tailscale.com/docs/features/subnet-routers). This has been proven to allow secure access through both a reverse proxy and a VPN.
