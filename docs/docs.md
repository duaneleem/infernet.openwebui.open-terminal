# infernet.openwebui.open-terminal — planning brief

Use this document as the **source of truth for scope and constraints** when designing Docker Compose/Kubernetes manifests, environment configuration, and Open WebUI integration for [Open Terminal](https://github.com/open-webui/open-terminal).

## Purpose

Run a **self-hosted, sandboxed shell environment** that Open WebUI agents can use to execute commands, read/write files, and browse the filesystem via Open Terminal’s REST API — without exposing that API on the public internet.

## Primary consumer

- **Open WebUI** ([infernet.openwebui](https://github.com/open-webui/open-webui) stack) is the intended client.
- Wire it under **Admin Settings → Integrations → Open Terminal** (system-level), **not** as an OpenAPI tool server. System-level connections are proxied through the Open WebUI **backend**, so the terminal only needs to be reachable on the Docker network from `open-webui`.

## Related repos

| Repo | Role |
|------|------|
| [infernet.openwebui](https://github.com/open-webui/open-webui) | Host stack: LiteLLM, SearXNG, Open WebUI on shared Docker network `infernet` |
| [infernet.mcp.tools](https://github.com/open-webui/mcpo) | MCP tools via mcpo (Trello, Playwright, Context7) — **complementary**, not a substitute for terminal/CLI |
| **This repo** | Packaging, pinning, and ops for Open Terminal only |

## In scope

- Official upstream image `ghcr.io/open-webui/open-terminal` with **pinned version tags** (or digest).
- Docker Compose service(s) on the **internal** `infernet` network (or equivalent in K8s).
- Persistent volume for multi-user homes (`open-terminal-data` → `/home` when `OPEN_TERMINAL_MULTI_USER=true`).
- `OPEN_TERMINAL_API_KEY` and other config via `.env` modeled on [`techops/production/template.env`](../techops/production/template.env).
- Documentation for how admins configure Open WebUI to point at the internal URL.

## Out of scope (unless explicitly expanded later)

- Publishing Open Terminal ports to the host or the public internet.
- Mounting the host Docker socket (effectively root on the host) unless a future doc revision explicitly approves it.
- Committing API keys, tokens, or a populated `.env` with real secrets.
- Bare-metal / `pip install open-terminal` on the host (Docker-only for this deployment).
- Per-user browser-direct connections (user settings → Integrations) when the design goal is a single shared internal sandbox proxied by the server.

## Architecture constraints

- **Network:** Open Terminal listens on port **8000 inside the container**. Do **not** map host ports; traffic is **Docker-internal only** (e.g. `http://open-terminal:8000` from the `open-webui` service on network `infernet`).
- **Integration mode:** Prefer **system-level** Open Terminal in Open WebUI so requests go server → terminal, not browser → terminal.
- **Upstream:** Use the official [open-terminal](https://github.com/open-webui/open-terminal) project only — no forks unless security or pinning requirements force a thin wrapper Dockerfile.
- **Versioning:** Pin **specific release tags** (e.g. `v0.11.34`), not floating tags like `latest`, `main`, or unqualified `slim`/`alpine`. At image selection time, prefer the **newest stable release tag** from [GitHub releases](https://github.com/open-webui/open-terminal/releases) and record the chosen tag in the Dockerfile/Compose comment and this doc when implemented.
- **Runtime:** Target **Docker Compose** first (parity with infernet.openwebui); K8s manifests are a later phase.

## Upstream image variants (choose at planning time)

| Tag suffix | Best for | Size (approx.) | Notes |
|------------|----------|----------------|--------|
| *(default / full)* | AI agent sandboxes, rich tooling | ~4 GB | Node, gcc, ffmpeg, optional runtime apt/pip/npm installs |
| `slim` | Hardened internal deploy | ~430 MB | glibc; no runtime package env vars |
| `alpine` | Minimal footprint | ~230 MB | musl; some pip wheels may compile at build time |

**Deployed choice:** **full** image `0.11.34` (GHCR tag without `v` prefix; multi-user requires full). Avoid `latest` in production manifests.

Pinned in Compose:

```text
ghcr.io/open-webui/open-terminal:0.11.34
```

## Configuration surface

Settings resolve in order: CLI flags → environment variables → config TOML → defaults (see [upstream README](https://github.com/open-webui/open-terminal#configuration)).

### Required for this deployment

| Variable | Purpose |
|----------|---------|
| `OPEN_TERMINAL_API_KEY` | **Required** — server refuses to start without it; use a strong secret in `.env` (not committed) |

### Common optional variables (plan per environment)

| Variable | Purpose |
|----------|---------|
| `OPEN_TERMINAL_MULTI_USER` | **`true` in production** — per-user Linux accounts; **not** production-grade isolation; only for small trusted groups |
| `OPEN_TERMINAL_PACKAGES` / `OPEN_TERMINAL_PIP_PACKAGES` / `OPEN_TERMINAL_NPM_PACKAGES` | Install extra tooling at container start (**full image only**) |
| `OPEN_TERMINAL_SESSION_CWD_TTL` | Session working-directory TTL (seconds) |
| Host/port | Usually `0.0.0.0:8000` inside the container; no host publish |

Copy [`techops/production/template.env`](../techops/production/template.env) to **`techops/production/.env`** (gitignored). Root [`template.env`](../template.env) redirects to that path.

## Open WebUI wiring (after deploy)

1. Ensure `open-terminal` and `open-webui` share the same Docker network (`infernet` in infernet.openwebui).
2. In Open WebUI: **Admin Settings → Integrations → Open Terminal**.
3. **URL:** `http://<compose-service-name>:8000` (e.g. `http://open-terminal:8000`).
4. **API key:** same value as `OPEN_TERMINAL_API_KEY` in this repo’s `.env`.
5. Enable the connection; assign access by user/group as needed.

Do **not** register Open Terminal under OpenAPI tool servers — that path is for mcpo/OpenAPI integrations (see infernet.mcp.tools).

## Production assets (repo layout)

| Path | Role |
|------|------|
| [`techops/production/docker-compose.production.yaml`](../techops/production/docker-compose.production.yaml) | Pinned `0.11.34` full image, multi-user, volume `/home`, external `infernet`, no host ports |
| [`techops/production/template.env`](../techops/production/template.env) | Committed env template |
| [`techops/production/.env`](../techops/production/.env) | Local secrets (gitignored); copy from `template.env` |
| [`techops/production/README.md`](../techops/production/README.md) | Runbook, Open WebUI wiring, smoke tests |
| [`docs/docs.md`](docs.md) | This planning brief |
| [`README.md`](../README.md) | High-level deploy overview |

No `production.Dockerfile` — upstream semver tags are pinned directly in Compose.

## Security and compliance

- **No secrets in git:** API keys only in `.env` or external secret managers.
- **No host port exposure** for Open Terminal; rely on Docker network isolation.
- **Do not mount** `/var/run/docker.sock` unless explicitly approved — it grants effective host root.
- **API key:** treat `OPEN_TERMINAL_API_KEY` like a service credential; rotate via Compose redeploy.
- **Multi-user mode:** document clearly if enabled; upstream warns it is not a security boundary for untrusted tenants.
- **CORS:** restrict `cors_allowed_origins` in production if the service is ever reachable beyond the intended proxy path.

## Operations

- **Health:** use HTTP readiness against `/docs` or a lightweight API route once Compose exists.
- **Persistence:** named volume `open-terminal-data` → `/home` (multi-user layout).
- **Restart:** `restart: unless-stopped` (consistent with infernet.openwebui).
- **Upgrades:** bump pinned tag in Dockerfile/Compose, `docker compose pull` / `build --no-cache`, verify Open WebUI integration still works; check [release notes](https://github.com/open-webui/open-terminal/releases) for breaking changes.

## Implementation checklist (for agents)

Use as a task breakdown; order may shift after a smoke test with infernet.openwebui.

1. [x] Choose image variant: **full** `0.11.34` (required for multi-user); documented in Compose.
2. [x] Flesh out [`techops/production/template.env`](../techops/production/template.env).
3. [x] Add `techops/production/` Compose; join external network `infernet`.
4. [x] **No** `ports:` mapping on `open-terminal`.
5. [x] Volume `open-terminal-data` → `/home`; backup notes in techops README.
6. [x] Open WebUI admin steps in README and techops README.
7. [x] Smoke test (infra): `oi-open-terminal` on `infernet`, no host ports, `curl` from `oi-open-webui` → 200 on `/docs`. UI: list files / `echo hello` after Admin integration wiring.
8. [ ] Optional: K8s Service + Deployment with internal ClusterIP only; secret refs for API key.

## Resolved decisions

| Topic | Choice |
|-------|--------|
| Image | Full `ghcr.io/open-webui/open-terminal:0.11.34` (multi-user) |
| Multi-user | `OPEN_TERMINAL_MULTI_USER=true` |
| Compose location | Standalone stack in this repo; external network `infernet` |
| Resource limits | 4G memory limit, 1G reservation |
| Host ports | None |

## Open decisions (fill in as the project evolves)

- **Egress:** whether upstream egress firewall defaults are sufficient or need custom allowlists.
- **Stronger isolation:** adopt [Terminals](https://github.com/open-webui/terminals) if untrusted multi-tenant use appears.

## Success criteria

- A new engineer or agent can deploy Open Terminal and connect Open WebUI **without guessing** internal URL, network name, or secret location.
- Open Terminal is **not** reachable from outside the Docker network.
- Image version is **pinned and documented**, not `latest`.
- infernet.openwebui agents gain CLI/filesystem capabilities through the official Open Terminal integration path.

## References

- [Open Terminal (upstream)](https://github.com/open-webui/open-terminal)
- [Open Terminal releases](https://github.com/open-webui/open-terminal/releases)
- [Open WebUI](https://github.com/open-webui/open-webui)
- [Terminals (per-user containers)](https://github.com/open-webui/terminals) — if stronger isolation is required later
- [infernet.mcp.tools planning brief](https://github.com/open-webui/mcpo) — MCP/OpenAPI tools (separate concern)
