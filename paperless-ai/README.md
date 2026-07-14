# paperless-ai

AI-powered document tagging and semantic search for
[paperless-ngx](../paperless-ngx). Watches paperless for new documents,
analyzes them with an LLM (OpenAI / Ollama / DeepSeek / any OpenAI-compatible
backend) and assigns title, tags, document type and correspondent — plus a
RAG-powered chat over your archive.

Upstream: <https://github.com/clusterzx/paperless-ai>

> **Note:** upstream is currently in maintenance limbo (the author is rewriting
> the codebase and paperless-ngx is gaining its own AI integration). Pinned to
> `3.0.9`, the last tagged release.

## Changes from the vendor default

- **Hardened**: `cap_drop: [ALL]`, `security_opt: [no-new-privileges]`, runs as
  non-root `568:568`. The image ships no privilege-dropping entrypoint, so
  `HOME`/`PM2_HOME` are pointed at the writable data volume for the PM2 runtime,
  and `/app/logs` (a hardcoded, root-owned path in the image) is backed by a
  `tmpfs` so the non-root user can write to it. App logs still stream to
  `docker compose logs` via pm2's stdout.
- **No published host ports**: reached only over the external `reverse-proxy`
  network (port `3000`) via the reverse proxy — no `-p 3000:3000`.
- **Bind mount** via `VOLUME_DATA_SRC/DST` instead of a named volume. The data
  dir holds the SQLite config DB and the RAG vector store, so it lives on SSD.
- **Image pinned** by tag + digest.

The bundled Python RAG service (listens on `127.0.0.1:8000` inside the
container) stays enabled via `RAG_SERVICE_ENABLED=true`.

## First-time setup

```bash
cd paperless-ai

# Create the bind-mount dir with the correct ownership.
sudo mkdir -p /mnt/ssd/appdata/paperless-ai/data
sudo chown -R 568:568 /mnt/ssd/appdata/paperless-ai/data

docker compose config   # validate
docker compose up -d
```

Then open the setup wizard through your reverse proxy and configure:

- **Paperless-ngx connection** — API URL `https://paperless.pc-ziegert.de/api`
  and an API token (paperless-ngx → *Settings → My Profile → API Token*).
- **AI provider** — OpenAI / Ollama / DeepSeek / custom OpenAI-compatible
  endpoint, model and key.

All of this is stored in the data volume, so it survives restarts and image
updates — nothing sensitive needs to live in `.env`.

## Reverse proxy

Point your reverse proxy (zoraxy / nginx-proxy-manager) at the `paperless-ai`
container on port `3000` over the `reverse-proxy` network. TLS is terminated at
the proxy.
