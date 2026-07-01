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

- Create a Cloudflare Tunnel (Zero Trust → Networks → Tunnels). Copy the token into
  `TUNNEL_TOKEN` in `.env` and `docker compose up -d`.
- Add DNS: **CNAME** `*.ext` → `<tunnel-id>.cfargotunnel.com`, proxied (orange).
- Point the tunnel's public hostname `*.ext.<domain>` at `https://caddy:443`.
- Add a Cloudflare Access application on `*.ext.<domain>` with your policy (email / Google / GitHub).
- Give a service an `ext.*` block to expose it:
  ```
  cp examples/caddy/public-service.caddy caddy/conf.d/grafana.caddy
  ```

## 6. Verify

```
git check-ignore .env caddy/conf.d/sonarr.caddy cloudflared/config.yml
git ls-files | xargs grep -l "<your-domain>"
```

The first lists every real-value file; the second returns nothing. From off the tailnet,
`*.lab.<domain>` resolves but times out; `*.ext.<domain>` prompts Cloudflare Access.
Confirm the host has no router port-forwards — the tunnel is outbound-only.
