# Architecture

One reverse proxy (Caddy) is the only front door. Two paths reach it: Tailscale for
private access, a Cloudflare Tunnel for the subset you choose to expose publicly —
either gated behind Cloudflare Access or fully open.

```mermaid
flowchart LR
    dev["Your devices<br/>(on Tailscale)"]
    pub["Public internet"]

    subgraph cf["Cloudflare"]
        edge["Edge + Access<br/>(SSO / OTP)"]
        edgeopen["Edge<br/>(no Access)"]
    end

    subgraph host["CasaOS host (Docker)"]
        cfd["cloudflared"]
        caddy["Caddy :443<br/>auto-HTTPS"]
        app1["CasaOS app :PORT"]
        app2["Your app :PORT"]
        dash["CasaOS dashboard :80"]
    end

    dev -->|"*.lab.&lt;domain&gt; → 100.x"| caddy
    pub -->|"*.ext.&lt;domain&gt;"| edge
    pub -->|"*.pub.&lt;domain&gt;"| edgeopen
    edge -->|"tunnel (outbound only)"| cfd
    edgeopen -->|"tunnel (outbound only)"| cfd
    cfd --> caddy
    caddy --> app1
    caddy --> app2
    caddy --> dash
```

## Hostname spaces

| Space | Resolves to | Reachable from | Guard |
|---|---|---|---|
| `*.lab.<domain>` | Tailscale IP `100.x` (A record, DNS-only) | tailnet devices only | Tailscale membership |
| `*.ext.<domain>` | Cloudflare Tunnel (CNAME, proxied) | public internet | Cloudflare Access |
| `*.pub.<domain>` | Cloudflare Tunnel (CNAME, proxied) | public internet | **none — fully open** |

A public wildcard A-record pointing at a `100.x` address resolves for everyone but is
only *routable* from inside the tailnet — so no self-hosted DNS is needed.

## TLS

Caddy issues real Let's Encrypt certificates via the DNS-01 challenge using a scoped
Cloudflare API token. DNS-01 works even though names resolve to a private IP, and covers
the wildcard, so every new subdomain is trusted immediately. This requires a custom Caddy
build with the Cloudflare DNS module (`caddy/Dockerfile`).

Caddy's certificate is the whole story only for `*.lab` — DNS-only, so the browser reaches
Caddy directly (one TLS hop). The proxied public tiers (`*.ext`, `*.pub`) split TLS in two:
browser → Cloudflare edge, then edge → Caddy through the tunnel. Caddy's cert secures only
the second hop; the edge hop needs a certificate Cloudflare issues, and free Universal SSL
does not cover two-level names. Covering them requires Advanced Certificate Manager (or
flattening public names to one level) — see [tls.md](tls.md).

## Reaching services

CasaOS apps publish ports on the host, so Caddy proxies to `host.docker.internal:<port>`
(enabled by `host-gateway` in `docker-compose.yml`). This works for containers CasaOS
launched, with no shared Docker network.

## Adding a service

Drop one file in `caddy/conf.d/` (gitignored), then reload:

```
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

Use a `lab.*` hostname for private, `ext.*` for public behind Access, `pub.*` for fully
open. See `examples/caddy/`.

## Ports

CasaOS holds host port 80. Caddy binds 443 only; DNS-01 needs no port 80, so there is no
conflict. Reach the CasaOS dashboard through a `casaos.lab.<domain>` block pointing at `:80`.

## Public repo boundary

Everything committed is a generic template. Real values live only in gitignored files:

| Committed (generic) | Gitignored (your values) |
|---|---|
| `docker-compose.yml`, `caddy/`, `cloudflared/config.example.yml`, `docs/`, `examples/` | `.env`, `caddy/conf.d/*.caddy`, `cloudflared/config.yml`, `cloudflared/*.json`, `caddy_data/` |

The tunnel config uses two wildcard ingresses (`*.ext.<domain>` and `*.pub.<domain>`, both
→ caddy), so even it names no individual service. What is exposed is decided only by your
`ext.*` and `pub.*` blocks; whether a name is gated is decided only by the scope of the
Cloudflare Access app (`*.ext.<domain>`, never the bare `*.<domain>`).
