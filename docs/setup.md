# Setup

## Prerequisites

- A CasaOS host with Docker and `docker compose`.
- A domain with nameservers pointed at Cloudflare.

## 1. Cloudflare

- Add the domain to Cloudflare and switch its nameservers.
- Create a scoped API token: **Zone → DNS → Edit**, restricted to your zone.
  https://dash.cloudflare.com/profile/api-tokens

## 2. Tailscale

- Install Tailscale on the host and note its `100.x` address:
  ```
  curl -fsSL https://tailscale.com/install.sh | sh
  sudo tailscale up
  tailscale ip -4
  ```
- In Cloudflare DNS add an **A** record `*.lab` → the `100.x` address, **DNS-only** (grey cloud).

## 3. Configure

```
cp .env.example .env
```

Fill in `DOMAIN`, `CLOUDFLARE_API_TOKEN`, and (after step 5) `TUNNEL_TOKEN`.

## 4. Start

```
docker compose up -d --build
```

Add a service:

```
cp examples/caddy/private-service.caddy caddy/conf.d/sonarr.caddy
# edit host/port, then:
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

Open `https://sonarr.lab.<domain>` from a device on your tailnet.

## 5. Public access (optional)

> **Required first — edge certificate.** The public tiers (`*.ext`, `*.pub`) are two-level
> subdomains served through the proxied tunnel, so Cloudflare's edge must hold a certificate
> that covers them. Free Universal SSL does **not** (it stops at one subdomain level), and
> without it you get `ERR_SSL_VERSION_OR_CIPHER_MISMATCH` and no Access prompt. Set this up
> before the steps below — buy **Advanced Certificate Manager** and order an advanced cert
> for `*.ext.<domain>` and `*.pub.<domain>` (~$10/mo). Full walkthrough, plus a free
> single-level alternative, in [docs/tls.md](tls.md).

- Create a Cloudflare Tunnel (Zero Trust → Networks → Tunnels). Copy the token into
  `TUNNEL_TOKEN` in `.env` and `docker compose up -d`.
- Add DNS: **CNAME** `*.ext` → `<tunnel-id>.cfargotunnel.com`, proxied (orange).
- Point the tunnel's public hostname `*.ext.<domain>` at `https://caddy:443`.
- Add a Cloudflare Access application on `*.ext.<domain>` with your policy (email / Google / GitHub).
  Scope it to `*.ext.<domain>` exactly — **not** the bare `*.<domain>` wildcard, or it would
  also gate the fully-public space below.
- Give a service an `ext.*` block to expose it:
  ```
  cp examples/caddy/public-service.caddy caddy/conf.d/grafana.caddy
  ```

## 5b. Fully public — no authentication (optional)

For services safe to leave open to anyone (a blog, a status page, a shareable link). Same
tunnel, but the `*.pub.<domain>` space is never covered by a Cloudflare Access app, so
requests reach the service ungated.

> **Warning:** A `*.pub.<domain>` service has no authentication. Anyone on the internet who
> knows or guesses the hostname can reach it. Only expose services that are safe fully open,
> and that handle their own auth and hardening. Never point a `pub.*` block at an admin panel,
> a database UI, or anything that assumes a trusted network.

- Add DNS: **CNAME** `*.pub` → `<tunnel-id>.cfargotunnel.com`, proxied (orange).
- Add the tunnel public hostname `*.pub.<domain>` → `https://caddy:443`.
- Confirm no Cloudflare Access application covers `*.pub.<domain>` (the one from step 5 should
  be scoped to `*.ext.<domain>` only).
- Give a service a `pub.*` block to expose it:
  ```
  cp examples/caddy/fully-public-service.caddy caddy/conf.d/blog.caddy
  # edit host/port, then:
  docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
  ```

## 6. Verify

```
git check-ignore .env caddy/conf.d/sonarr.caddy cloudflared/config.yml
git ls-files | xargs grep -l "<your-domain>"
```

The first lists every real-value file; the second returns nothing. From off the tailnet,
`*.lab.<domain>` resolves but times out; `*.ext.<domain>` prompts Cloudflare Access;
`*.pub.<domain>` loads with no prompt. Confirm the host has no router port-forwards — the
tunnel is outbound-only.
