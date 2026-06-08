# Infernet — Open Terminal and integrations (agent knowledge)

Paste this document into Open WebUI **Knowledge** (workspace or global) so agents know how to use the internal sandbox and Trello credentials. **Do not add API keys or tokens to this file** — they live in per-user `~/.oi_secrets` in Open Terminal (and in server `.env` for operators only).

---

## Open Terminal (sandbox)

- **What it is:** A sandboxed Linux environment on the internal Docker network `infernet`. You can run shell commands, read/write files under your user home, and use standard CLI tools (`curl`, `jq`, `git`, Python, Node, etc.).
- **How you access it:** Through Open WebUI’s **Open Terminal** integration (Admin → Integrations → Open Terminal). Requests are proxied by the Open WebUI backend; you do not connect to it from the browser directly.
- **Not the same as MCP/OpenAPI tools:** Open Terminal is **not** registered under OpenAPI tool servers. Use terminal commands for shell, scripts, and direct HTTP (`curl`). Use MCP tools (below) for structured tool calls when they are enabled.

### Rules for Open Terminal

1. Prefer running real commands over describing what you would run.
2. Stay inside the sandbox; do not assume access to the host machine outside the container.
3. Multi-user mode gives each WebUI user a separate home directory; do not assume you can read another user’s files.
4. Never print, log, or echo secret environment variables (`TRELLO_TOKEN`, `OPEN_TERMINAL_API_KEY`, or any `*_TOKEN` / `*_SECRET`).

---

## Trello — terminal (curl) vs MCP

Two ways to work with Trello exist in this stack. Choose based on the task.

| Approach | When to use |
| -------- | ----------- |
| **Open Terminal + curl** | Scripts, `jq` filtering, bulk reads/writes, custom REST paths, one-off API exploration, pipelines |
| **Trello MCP** (infernet.mcp.tools / mcpo) | Simple conversational ops: list boards, create/move cards, typical board workflows via named tools |

**Default for Trello board work in chat:** Use **Trello MCP** if those tools are available and the task fits the tool surface.

**Switch to Open Terminal + curl when:** the user asks for scripting, raw API access, bulk export, or MCP does not expose the endpoint you need.

---

## Trello credentials in Open Terminal (`~/.oi_secrets`)

Open Terminal runs in **multi-user** mode: Docker `.env` variables are **not** passed into your shell. Credentials live in a dotfile in your home directory:

| Path | Purpose |
| ---- | ------- |
| `~/.oi_secrets` | Per-user exports (`TRELLO_API_KEY`, `TRELLO_TOKEN`, optional `TRELLO_BOARD_ID`, `TRELLO_WORKSPACE_ID`) |

**Before any Trello `curl` or script**, load secrets in the **same command** (do not assume vars are already exported):

```bash
set -a && source ~/.oi_secrets && set +a
```

Or prefix one-liners:

```bash
source ~/.oi_secrets && curl -s "https://api.trello.com/1/members/me/boards?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" | jq .
```

**Never** ask the user for API keys or tokens. **Never** run `cat ~/.oi_secrets`, `echo $TRELLO_TOKEN`, `printenv`, or a full `env` dump — that exposes secrets in logs.

If `~/.oi_secrets` is missing or sourcing fails, tell the user to create it in their Open Terminal home (`.bashrc` may not be writable). Do not request secret values in chat.

| Variable (after `source`) | Purpose |
| ------------------------- | ------- |
| `TRELLO_API_KEY` | Trello API key (query param `key=`) |
| `TRELLO_TOKEN` | Trello user token (query param `token=`) |
| `TRELLO_BOARD_ID` | Optional default board id |
| `TRELLO_WORKSPACE_ID` | Optional default workspace id |

### Verify credentials (safe)

```bash
source ~/.oi_secrets 2>/dev/null && test -n "$TRELLO_API_KEY" && test -n "$TRELLO_TOKEN" && echo "Trello env OK" || echo "Trello env missing (check ~/.oi_secrets)"
```

### API pattern

Base URL: `https://api.trello.com/1/`

Always **source first**, then pass key and token as **query parameters**:

```bash
source ~/.oi_secrets && curl -s "https://api.trello.com/1/members/me/boards?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" | jq .
```

More examples:

```bash
source ~/.oi_secrets && curl -s "https://api.trello.com/1/boards/${TRELLO_BOARD_ID}/lists?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" | jq .

source ~/.oi_secrets && curl -s "https://api.trello.com/1/lists/LIST_ID/cards?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" | jq .

source ~/.oi_secrets && curl -s -X POST "https://api.trello.com/1/cards?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"idList":"LIST_ID","name":"Card title","desc":"Optional description"}' | jq .
```

Board id hint: for a board URL `https://trello.com/b/SHORTID/name`, the board id is in `https://trello.com/b/SHORTID.json` (first `id` field).

Official reference: [Trello REST API](https://developer.atlassian.com/cloud/trello/rest/)

---

## Related infernet services (context only)

| Service | Role |
| ------- | ---- |
| **infernet.openwebui** | Open WebUI, LiteLLM, SearXNG on network `infernet` |
| **infernet.mcp.tools (mcpo)** | Trello, Playwright, Context7 MCP behind one HTTP gateway — wire in Open WebUI as **OpenAPI tool server**, not Open Terminal |
| **Open Terminal** (this stack) | CLI/filesystem sandbox at `http://open-terminal:8000` on `infernet` |

---

## Quick decision checklist

1. Need a shell, file edit, or `curl`? → **Open Terminal**
2. Need Trello and MCP tools are enabled? → **Trello MCP** first unless user wants curl/scripting
3. Need Trello REST not covered by MCP? → **Open Terminal**: `source ~/.oi_secrets` then curl with `${TRELLO_API_KEY}` and `${TRELLO_TOKEN}`
4. Need library/docs context for code? → **Context7 MCP** (infernet.mcp.tools), not terminal
5. Need browser automation? → **Playwright MCP** (infernet.mcp.tools), not terminal
