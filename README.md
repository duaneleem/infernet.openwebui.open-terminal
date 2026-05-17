# infernet.openwebui.open-terminal

Self-hosted [Open Terminal](https://github.com/open-webui/open-terminal) for the **infernet.openwebui** stack: sandboxed CLI and filesystem access for AI agents, internal Docker network only.

## How this is deployed

1. **infernet.openwebui** runs first (creates Docker network `infernet` and `oi-open-webui`).
2. Copy [`techops/production/template.env`](techops/production/template.env) to `techops/production/.env` and set `OPEN_TERMINAL_API_KEY`.
3. Start this stack:

```bash
docker compose -f techops/production/docker-compose.production.yaml \
  --env-file techops/production/.env up -d
```

4. In Open WebUI: **Admin Settings → Integrations → Open Terminal** — URL `http://open-terminal:8000`, API key from `.env`.

Details: [techops/production/README.md](techops/production/README.md). Planning brief: [docs/docs.md](docs/docs.md).

## Defaults

| Setting | Value |
|---------|--------|
| Image | `ghcr.io/open-webui/open-terminal:0.11.34` (full; pinned) |
| Multi-user | `OPEN_TERMINAL_MULTI_USER=true` |
| Host ports | None (internal `infernet` only) |
| Service DNS | `open-terminal:8000` |
