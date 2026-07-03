# Self-hosted Excalidraw

A full-stack Excalidraw with both sharing modes, reachable on your tailnet at
`*.lab.<domain>` through the existing Caddy setup.

- **Live collaboration** — start a session, share the link, multiple people edit in
  real time. Handled by `excalidraw-room` (WebSocket relay). The room is end-to-end
  encrypted and not stored server-side.
- **Persistent links** — "Export to link" produces a board that survives after everyone
  leaves. Handled by the storage backend + Redis.

## Services

| Service | Image | Hostname | Role |
|---|---|---|---|
| frontend | `kiliandeca/excalidraw` | `draw.lab.<domain>` | the editor |
| room | `excalidraw/excalidraw-room` | `draw-collab.lab.<domain>` | live-collab WebSocket |
| storage | `kiliandeca/excalidraw-storage-backend` | `draw-storage.lab.<domain>` | persistent scenes |
| redis | `redis:7-alpine` | internal | scene datastore |

The frontend's backend URLs are read from environment variables **at container start**,
so no image rebuild is needed. They must be the browser-facing Caddy hostnames — the
browser talks to the room and storage services directly.

## Setup

```
cp -r examples/excalidraw apps/excalidraw
cp examples/caddy/excalidraw.caddy caddy/conf.d/excalidraw.caddy
docker compose --env-file .env -f apps/excalidraw/docker-compose.yml up -d
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

`DOMAIN` comes from `.env`. All three hostnames are single-label under `.lab`, so the
existing `*.lab.<domain>` wildcard record and certificate already cover them.

## Verify

- `https://draw.lab.<domain>` loads the editor with a valid certificate.
- Start a live session, open the link on a second device on the tailnet — edits sync.
- Export a shareable link (`.../#json=...`), open it in a browser with cleared storage —
  the drawing loads. Restart the `redis` container and reopen it — still loads.

## Notes

- `kiliandeca/*` images can lag upstream Excalidraw. Pin explicit tags once verified;
  `alswl/excalidraw-*` is a more actively maintained alternative.
- If exporting a link fails with a CORS error in the browser console, adjust the storage
  backend's allowed-origin setting.
- Changing a hostname later means recreating the `excalidraw` container, not just Caddy.

## Going public

To share with people off your tailnet, add `ext.*` blocks (see
[examples/caddy/public-service.caddy](../examples/caddy/public-service.caddy)), point the
frontend env URLs at the `ext` hostnames, recreate the container, and add a Cloudflare
Access policy on `*.ext.<domain>` (or bypass it for open sharing).
