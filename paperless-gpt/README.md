# paperless-gpt

AI-powered document processing for [paperless-ngx](../paperless-ngx). Watches
paperless for documents tagged `paperless-gpt` / `paperless-gpt-auto`,
analyzes them with an LLM and assigns title, tags, correspondent, document
type and created date — plus optional LLM-based OCR. This deployment is
pinned to **Ollama** as the only LLM backend (`LLM_PROVIDER: ollama` is
hardcoded in the compose file) — documents never leave the local network.

Upstream: <https://github.com/icereed/paperless-gpt>

Replaces [paperless-ai](../disabled/paperless-ai) (upstream in maintenance
limbo, moved to `disabled/`).

## Changes from the vendor default

- **Hardened**: `cap_drop: [ALL]`, `security_opt: [no-new-privileges]`. The
  image's entrypoint must start as root (it chowns the data mounts, then drops
  to `PUID:PGID` via su-exec), so instead of a `user:` override it runs with
  `PUID/PGID=568` and only the caps the entrypoint needs re-added
  (`CHOWN`, `DAC_OVERRIDE`, `SETGID`, `SETUID`).
- **No published host ports**: reached only over the external `reverse-proxy`
  network (port `8080`) via the reverse proxy — no `-p 8080:8080`.
  paperless-gpt has **no built-in authentication**, so never expose it
  directly.
- **Bind mounts** via `VOLUME_*_SRC/DST` instead of relative paths: `prompts`
  (customizable prompt templates), `db` (SQLite processing state) and `config`,
  all on SSD.
- **Configuration via `.env`** instead of inline values — the paperless API
  token is a secret and stays out of git. `LLM_PROVIDER` is hardcoded to
  `ollama` (no cloud providers, no API keys).
- **Image pinned** by tag + digest.
- **Healthcheck added** (busybox `wget` — the image ships no curl).

## First-time setup

```bash
cd paperless-gpt

# Create the bind-mount dirs; ownership (568:568) is fixed by the
# image entrypoint on startup.
sudo mkdir -p /mnt/ssd/appdata/paperless-gpt/{prompts,db,config}

cp .env.example .env   # then fill in PAPERLESS_API_TOKEN, OLLAMA_HOST + model

docker compose config   # validate
docker compose up -d
```

In paperless-ngx, create the tags `paperless-gpt` (manual review queue) and
`paperless-gpt-auto` (fully automatic processing), e.g. as part of a
consumption workflow. paperless-gpt polls for documents carrying those tags.

## Reverse proxy

Point your reverse proxy (zoraxy / nginx-proxy-manager) at the `paperless-gpt`
container on port `8080` over the `reverse-proxy` network. TLS is terminated
at the proxy. Since the app has no authentication of its own, restrict access
at the proxy (e.g. Authentik forward auth or an access list).
