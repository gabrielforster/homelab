# Self-hosted FrankMD

[FrankMD](https://github.com/akitaonrails/FrankMD) is a self-hosted markdown
note-taking app (Ruby on Rails 8) that stores every note as a plain `.md` file on disk.
Reachable on your tailnet at `frankmd.lab.<domain>` through the existing Caddy setup.

## Setup

```
cp -r examples/frankmd apps/frankmd
cp examples/caddy/frankmd.caddy caddy/conf.d/frankmd.caddy
```

Generate a production secret and add it to `.env` (with `DOMAIN`):

```
echo "SECRET_KEY_BASE=$(openssl rand -hex 64)" >> .env
```

Start it and reload Caddy:

```
docker compose --env-file .env -f apps/frankmd/docker-compose.yml up -d
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

Notes land in `apps/frankmd/notes/`, local images in `apps/frankmd/images/` — both
gitignored with the rest of `apps/`.

## Configuration

FrankMD reads settings from three places, in priority order: the `.fed` file in the
notes directory (highest) → environment variables → built-in defaults. The `.fed` file is
created automatically on first run with every option present but commented out, so most
tuning happens in-app rather than in the compose file.

Environment variables set in [examples/frankmd/docker-compose.yml](../examples/frankmd/docker-compose.yml):

| Variable | Feature it enables | Default |
|---|---|---|
| `SECRET_KEY_BASE` | Rails session/encryption secret. Required in production. | — |
| `NOTES_PATH` | Where `.md` notes are stored (mapped to the `notes` volume). | `./notes` |
| `IMAGES_PATH` | Enables **local** image storage; unset means local images are disabled. | disabled |
| `FRANKMD_LOCALE` | UI language: `en`, `pt-BR`, `pt-PT`, `es`, `he`, `ja`, `ko`. | `en` |

Options exposed through the `.fed` file (edit `apps/frankmd/notes/.fed`, no restart needed):

| Group | Keys | What you get |
|---|---|---|
| Appearance | `theme`, `editor_font`, `editor_font_size`, `preview_zoom`, `sidebar_visible`, `typewriter_mode` | Editor look and behaviour (e.g. `theme = gruvbox`, typewriter scrolling). |
| AI / LLM | `ai_provider` plus per-provider `*_api_key` / `*_model` for Ollama, OpenRouter, Anthropic, Gemini, OpenAI | In-app AI assistance. `ai_provider = auto` picks whichever key is present; point `ollama_api_base` at a local model to stay fully offline. |
| Web search | `youtube_api_key`, `google_api_key`, `google_cse_id` | YouTube lookups and Google Custom Search from inside notes. |
| Image hosting | `aws_access_key_id`, `aws_secret_access_key`, `aws_s3_bucket`, `aws_region` | Upload pasted images to S3 instead of local disk — see below. |

Keys placed in `.fed` override the environment variables of the same name. Keep secrets
(API keys) in `.fed` inside the gitignored `notes/` dir, or in `.env` — never committed.

## Optional: Image Hosting (AWS S3)

By default images go to the `IMAGES_PATH` volume. Set the four AWS variables (in `.env`,
already wired into the compose file, or in `.fed`) to push images uploaded through the UI
to an S3 bucket instead:

```
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_S3_BUCKET=your-bucket
AWS_REGION=us-east-1
```

### IAM permissions the key needs

Create a dedicated IAM user (not a root key) scoped to **only** this one bucket. The app
uploads, serves, and can replace/remove images, so grant these actions:

| Action | Why |
|---|---|
| `s3:PutObject` | upload an image |
| `s3:GetObject` | read/serve an image back |
| `s3:DeleteObject` | remove an image deleted in the app |
| `s3:ListBucket` | check for existing keys / avoid collisions |

Least-privilege policy — replace `your-bucket`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "FrankMDObjects",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::your-bucket/*"
    },
    {
      "Sid": "FrankMDList",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::your-bucket"
    }
  ]
}
```

Note the two `Resource` forms: object actions target `arn:aws:s3:::your-bucket/*` (the
objects), while `s3:ListBucket` targets `arn:aws:s3:::your-bucket` (the bucket itself).

### Serving the images publicly

The IAM key above lets the **app** read and write. For a browser to load an uploaded
image, the object must also be reachable. Two ways:

- **Bucket policy for public read** (simplest): turn off "Block public access" for the
  bucket and add a bucket policy allowing `s3:GetObject` to everyone on
  `arn:aws:s3:::your-bucket/*`. The app then needs no ACL permission.
- **Keep the bucket private** and rely on the app issuing pre-signed URLs. If instead the
  app sets a `public-read` ACL per object, add `s3:PutObjectAcl` to the policy above and
  leave ACLs enabled on the bucket — but modern buckets block ACLs by default, so the
  bucket-policy route is preferred.
