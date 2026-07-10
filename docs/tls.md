# Public-tier TLS & gating

Private (`*.lab.<domain>`) needs nothing here — it is DNS-only, so your browser
reaches Caddy directly and Caddy serves its own Let's Encrypt certificate (issued via
DNS-01, which covers the two-level name fine).

The public tiers (`*.ext.<domain>`, `*.pub.<domain>`) are different. They run through the
Cloudflare Tunnel, and a tunnel **only routes when its DNS record is proxied** (orange
cloud). Proxied means your browser terminates TLS at Cloudflare's edge, not at Caddy — so
there are two independent TLS legs:

```
browser ──leg 1──► Cloudflare edge ──leg 2 (tunnel)──► Caddy
          edge cert                   Caddy cert (noTLSVerify in tunnel config)
          ▲ Cloudflare's to issue     ▲ already works
```

Caddy's certificate secures **leg 2** only. **Leg 1** is Cloudflare's — *Cloudflare* chooses
the certificate, and you can't hand it Caddy's (custom-cert upload is a Business-plan
feature). Cloudflare's free **Universal SSL** covers the apex and **one** subdomain level
(`*.<domain>`) — it does **not** cover a two-level name like `grafana.ext.<domain>`.

So a request to a two-level public name fails the TLS handshake at the edge:

```
ERR_SSL_VERSION_OR_CIPHER_MISMATCH
```

…and Cloudflare Access never runs (there's no TLS session to run HTTP over), which is why
there's **no login prompt** — just the SSL error.

To fix it you must give Cloudflare's edge a certificate covering the two-level names.
There are two supported ways.

---

## Option A — Advanced Certificate Manager + Total TLS (paid, keeps `*.ext` / `*.pub`)

This is the path this repo is built for. It keeps the tier encoded in the hostname
(`lab` / `ext` / `pub`) and the single wildcard Access app. Cost: **Advanced Certificate
Manager (ACM) is a per-zone add-on, ~$10/month** (confirm the current price in the
dashboard). Available on Free, Pro, and Business plans.

### 1. Buy Advanced Certificate Manager

1. Cloudflare dashboard → select your zone.
2. **SSL/TLS → Edge Certificates**.
3. Find **Advanced Certificate Manager** and select **Purchase** (or **Subscriptions**).
   Confirm the ~$10/month charge; it bills to your Cloudflare account payment method.

### 2. Order an advanced certificate for the wildcards

The repo uses one wildcard DNS record per public tier (`*.ext`, `*.pub`), so order a cert
that names those wildcards explicitly — this is what guarantees coverage.

1. **SSL/TLS → Edge Certificates → Create certificate** (Advanced).
2. **Hostnames** — add:
   - `*.ext.<domain>`
   - `*.pub.<domain>`
   - (optional) `ext.<domain>` and `pub.<domain>` if you'll ever use the bare names.

   A single advanced cert allows up to 50 hostnames.
3. **Certificate authority** — Let's Encrypt or Google Trust Services (either is fine).
4. **Validation method** — TXT is simplest. Leave the rest at defaults.
5. Submit. Issuance and edge deployment take a few minutes up to a couple of hours.

> If your zone has **CAA** DNS records, they must permit the CA you picked
> (`letsencrypt.org` or `pki.goog`) — otherwise issuance fails. No CAA records = no
> restriction, nothing to do.

### 3. (Optional) Enable Total TLS

**SSL/TLS → Edge Certificates → Total TLS → On**, and pick a CA. Total TLS auto-issues an
individual certificate for every proxied hostname going forward, so you don't have to
order one per name. It keys off concrete DNS records, though — with the pure wildcard
records this repo uses, the **ordered advanced certificate in step 2 is what actually
covers `*.ext` / `*.pub`**. Total TLS is the set-and-forget complement if you later add
per-service DNS records.

> Total TLS caveat: if you ever delete a Total-TLS-issued certificate, Cloudflare treats
> that hostname as opted out and won't reissue — even if you recreate its DNS record.

### 4. Verify

```
openssl s_client -connect grafana.ext.<domain>:443 -servername grafana.ext.<domain> </dev/null \
  | grep -E "subject=|issuer="
```

You should now see a real certificate (not `no peer certificate available`). In a browser,
`grafana.ext.<domain>` should present the **Cloudflare Access** login instead of the SSL
error; `blog.pub.<domain>` should load with no prompt.

---

## Option B — Single-level names (free)

No ACM. Public services live one level under the apex (`grafana.<domain>`), which
Universal SSL's `*.<domain>` already covers. The tunnel and no-port-forward posture are
unchanged. You give up the tier-in-hostname convention for public services, and gating
moves from one wildcard Access app to per-hostname Access apps.

`*.lab.<domain>` stays two-level and untouched — it's DNS-only, never reaches the edge, so
Universal SSL is irrelevant to it.

**Setup:**

1. DNS: a proxied (orange) wildcard **CNAME** `*` → `<tunnel-id>.cfargotunnel.com`
   (covers every one-level public name at once), *or* one proxied CNAME per public
   service.
2. Tunnel public hostname `*.<domain>` (or per service) → `https://caddy:443`.
3. Caddy blocks use one-level names, e.g. `grafana.{$DOMAIN}` instead of
   `grafana.ext.{$DOMAIN}`.

Both gated and open services coexist — gating is decided per hostname in Cloudflare
Access, using one of two patterns.

### Pattern 1 — default open, gate explicitly (fail-open)

No wildcard Access app. For each service you want gated, create an Access application
scoped to that exact hostname with an **Allow** policy. Any name without an app is public.

```
grafana.<domain>   Access app → Allow (your email)   → gated
status.<domain>    (no app)                           → open
```

**Risk:** forget to add the app for a new sensitive service and it is silently public.
Use this only if most of your public services are meant to be open.

### Pattern 2 — default gated, open explicitly (fail-closed) — recommended

One wildcard Access app gates everything; you carve out the open ones with a **Bypass**
policy on a more-specific hostname app (the more specific hostname always wins).

1. Create an Access application on `*.<domain>` → **Allow** (your email / IdP). Now every
   proxied name is gated by default.
2. For each fully-public service, create an Access application on its exact hostname →
   policy **Bypass**, Include **Everyone**.

```
*.<domain>         Access app → Allow  (your email)     ← gates all proxied names
status.<domain>    Access app → Bypass (Everyone)       ← more specific → open
grafana.<domain>   (no override)                        ← inherits wildcard → gated
```

Forget to configure a new service and it stays **gated**, not exposed — the safe default.
This matches the repo's intent, where "fully public" is a deliberate exception.

> The `*.<domain>` Access app gates **every** proxied first-level name, so if you proxy
> anything else under the domain (a personal site at `www`, etc.), add a Bypass app for it
> too. `*.lab.<domain>` is never affected — it doesn't pass through Cloudflare's edge.

---

## Which to use

| | Tier in hostname | Access model | Public IP exposed | Cost |
|---|---|---|---|---|
| **A** ACM + advanced cert | `lab` / `ext` / `pub` | one `*.ext` wildcard app | no (tunnel) | ~$10/mo |
| **B** single-level | flat, tier in Access only | per-app (Pattern 1 or 2) | no (tunnel) | free |

A is what the committed examples (`*.ext`, `*.pub`) assume. Pick B if you'd rather rework
naming than pay — the DNS, Caddy blocks, and Access setup all change accordingly.
