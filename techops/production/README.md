# Production stack (Docker)

Open Terminal runs as a **standalone** service on the shared Docker network `infernet` (same as [infernet.openwebui](https://github.com/open-webui/open-webui)). There is **no host port publish**—only `open-webui` reaches it at `http://open-terminal:8000`.

## Files

| File | Role |
|------|------|
| `docker-compose.production.yaml` | Pinned `ghcr.io/open-webui/open-terminal:0.11.34`, multi-user, volume at `/home`, external `infernet` network. |
| `template.env` | Committed env template; copy to `.env` (gitignored) for secrets. |

## Environment

| Variable | Required | Notes |
|----------|----------|--------|
| `OPEN_TERMINAL_API_KEY` | Yes | Strong secret; same value in Open WebUI Admin → Integrations → Open Terminal. |
| `OPEN_TERMINAL_MULTI_USER` | Yes (default `true`) | Per-user Linux homes; **full image only** (not slim/alpine). |
| `OPEN_TERMINAL_SESSION_CWD_TTL` | No | Session cwd TTL in seconds. |

## Multi-user security

`OPEN_TERMINAL_MULTI_USER=true` gives each Open WebUI user a separate Linux account and home directory inside one container. Upstream states this is **not** production-grade tenant isolation (shared kernel/network). Use only for **small, trusted internal** groups. For per-user containers, see [Terminals](https://github.com/open-webui/terminals).

## Run (from repository root)

Prerequisite: **infernet.openwebui** stack running (creates network `infernet`), or:

```bash
docker network create infernet
```

```bash
cp techops/production/template.env techops/production/.env
# Set OPEN_TERMINAL_API_KEY in .env (e.g. openssl rand -hex 32)
docker compose -f techops/production/docker-compose.production.yaml \
  --env-file techops/production/.env up -d
```

Pull/update image:

```bash
docker compose -f techops/production/docker-compose.production.yaml \
  --env-file techops/production/.env pull
docker compose -f techops/production/docker-compose.production.yaml \
  --env-file techops/production/.env up -d
```

## Open WebUI integration

After deploy:

1. **Admin Settings → Integrations → Open Terminal** (not OpenAPI tool servers).
2. **URL:** `http://open-terminal:8000`
3. **API key:** `OPEN_TERMINAL_API_KEY` from `techops/production/.env`
4. Enable; assign users/groups as needed.

## Smoke tests

```bash
# Both containers on infernet
docker network inspect infernet --format '{{range .Containers}}{{.Name}} {{end}}'

# Reachable from open-webui (expect 200)
docker exec oi-open-webui curl -sf -o /dev/null -w "%{http_code}\n" http://open-terminal:8000/docs

# No host publish (open-terminal should not appear in docker port output)
docker port oi-open-terminal 2>&1 || true
```

In the UI: file browser and `echo hello` via terminal. With two WebUI users, confirm isolated homes.

## Persistence

Workspace data lives in Docker volume `infernet-open-terminal_open-terminal-data` (Compose project prefix may vary). Back up that volume before major upgrades or `docker compose down -v`.

## Upgrades

Bump the image tag in `docker-compose.production.yaml`, pull, recreate, and re-test Open WebUI integration. See [release notes](https://github.com/open-webui/open-terminal/releases).
