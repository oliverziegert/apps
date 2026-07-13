# authentik

Identity provider / SSO. Adapted from the vendor `compose.yml` to this repo's
security and infrastructure conventions.

## Changes from the vendor default

- **Hardened all services**: `cap_drop: [ALL]`, `security_opt: [no-new-privileges]`,
  explicit non-root users (`999:999` for postgres, `568:568` for server/worker).
- **Dropped the Docker socket integration**: the worker no longer mounts
  `/var/run/docker.sock` and no longer runs as `root`. authentik can no longer
  auto-deploy outpost containers on the host — run outposts as standalone
  containers, or use the embedded proxy outpost inside the server.
- **No published host ports**: the server is reached only over the external
  `reverse-proxy` network (port `9000`) via the reverse proxy. TLS is terminated
  at the proxy, so authentik's own `9443` listener is unused.
- **Bind mounts** via `VOLUME_*_SRC/DST` env vars instead of a named volume.
- Postgres pinned to `17.10-alpine3.22` (repo standard); explicit healthchecks.

> Note: current authentik no longer requires Redis — it is not part of this stack.

## First-time setup

```bash
cd authentik

# Create the bind-mount directories with the correct ownership.
# DB lives on SSD, everything else on HDD (matches this repo's layout).
sudo mkdir -p /mnt/ssd/appdata/authentik/db
sudo mkdir -p /mnt/hdd/appdata/authentik/{data,certs,custom-templates}
sudo chown -R 999:999 /mnt/ssd/appdata/authentik/db
sudo chown -R 568:568 /mnt/hdd/appdata/authentik/{data,certs,custom-templates}

docker compose config   # validate
docker compose up -d
```

Then create the initial admin account at:

```text
https://<your-authentik-host>/if/flow/initial-setup/
```

## Reverse proxy

Point your reverse proxy (zoraxy / nginx-proxy-manager) at the `server`
container on port `9000` over the `reverse-proxy` network.
