---
title: "HOWTO — Set up OmniRoute to use every feature"
version: 3.8.50
lastUpdated: 2026-08-17
---

# HOWTO: Set up OmniRoute to leverage every feature

> OmniRoute is a **free AI gateway**: one OpenAI-compatible endpoint (`http://localhost:20128/v1`)
> in front of **341 providers / 90+ free tiers**, with auto-fallback combos, stacked token
> compression, MCP, A2A, memory, guardrails, media generation, and 34 coding-tool integrations.
> This guide walks you from a fresh `npm run dev` to a fully-loaded instance.

You are already past install — the first `npm run dev` migrated the SQLite DB and the server is
live at **http://localhost:20128**. This guide is about turning on the rest.

---

## 1. What your first boot told you (action items)

The startup log printed three things worth acting on, in priority order:

| #   | Log line                                                                             | Action                                                                              |
| --- | ------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| 1   | `INITIAL_PASSWORD is not set — using default 'CHANGEME'`                             | **Change the dashboard password now** (section 2). Anyone can sign in until you do. |
| 2   | `JWT_SECRET` / `API_KEY_SECRET` / `STORAGE_ENCRYPTION_KEY` auto-generated            | Nothing to do — already secure and persisted.                                       |
| 3   | `Antigravity OAuth (…CLIENT_SECRET)` / `Qoder OAuth (…CLIENT_SECRET)` not configured | Optional. These two providers stay disabled until you add secrets (section 3).      |

> Ignore the `[ProxyFetch] … ECONNREFUSED 127.0.0.1:20128` lines — those happen while the dev
> server is still booting and the proxy isn't listening yet. They stop once `[Next] dev server
listening` appears. The `compression_run_telemetry` cleanup error is a harmless, pre-existing
> cleanup-log bug (see Troubleshooting).

---

## 2. Harden security first (2 minutes)

1. **Change the password** — two options:
   - Dashboard → **Settings → Security**, or
   - stop the server and set `INITIAL_PASSWORD` in `.env` to a strong value, then restart.
     `INITIAL_PASSWORD` only seeds the _first_ login; after that the dashboard owns it.

2. **Generate fresh secrets** (only needed if you ever re-seed `.env` from scratch):

   ```bash
   # Windows Git Bash / macOS / Linux
   openssl rand -base64 48   # → JWT_SECRET
   openssl rand -hex 32      # → API_KEY_SECRET / STORAGE_ENCRYPTION_KEY
   ```

3. Know your data locations (Windows, from the boot log):

   ```text
   SQLite DB   : %APPDATA%\omniroute\storage.sqlite
   Secrets     : %APPDATA%\omniroute\server.env
   DB backups  : %APPDATA%\omniroute\db_backups\
   ```

For multi-user / public deployments, also review `REQUIRE_API_KEY=true` and
`AUTH_COOKIE_SECURE=true` in `.env` (see `docs/reference/ENVIRONMENT.md`).

---

## 3. Connect providers

### 3.1 Free providers (no key, no card) — works out of the box

`auto` already answers on a fresh install via the **OpenCode Free** and **Felo** keyless
providers. Add more free tiers in **Dashboard → Providers → Add Provider**:

| Provider                   | Models                | Auth                 |
| -------------------------- | --------------------- | -------------------- |
| **OpenCode Free** (`oc/…`) | several models        | none (pre-wired)     |
| **Felo** (`felo/…`)        | several models        | none (pre-wired)     |
| **Kiro**                   | free Claude           | one-click connect    |
| **Pollinations**           | GPT / Claude / Gemini | none                 |
| **Qoder**                  | free models           | OAuth (needs secret) |

Free-tier guidance: `docs/getting-started/FREE-TIERS-GUIDE.md`, `docs/reference/FREE_TIERS.md`.

### 3.2 OAuth providers (subscription / coding plans)

Dashboard → **Providers → Add Provider**, pick the provider, click **Connect** and complete the
OAuth flow in the browser. Supported OAuth providers include Claude Code, Codex, GitHub
Copilot, Cursor, GitLab Duo, Kimi, and more.

- **Antigravity** and **Qoder** require a client secret before OAuth works. Add them to `.env`
  (or `%APPDATA%\omniroute\server.env`) — the exact key names are printed in the boot log:
  `ANTIGRAVITY_OAUTH_CLIENT_SECRET` and `QODER_OAUTH_CLIENT_SECRET`.
- If an OAuth provider's env gets corrupted, use the one-click **Repair env** action on its
  provider page.

### 3.3 API-key providers

Dashboard → **Providers → Add Provider** → pick the provider → paste the key. Works for OpenAI,
Anthropic, Gemini, DeepSeek, Groq, xAI, OpenRouter, Mistral, and hundreds more. Keys are
encrypted at rest with AES-256-GCM.

### 3.4 Verify

```bash
curl http://localhost:20128/v1/models -H "Authorization: Bearer YOUR_KEY"
```

Or, from the CLI:

```bash
omniroute providers list
omniroute providers test <id-or-name>
omniroute providers test-all
omniroute doctor
```

---

## 4. Create an API key and smoke-test

1. Dashboard → **Endpoints** → create a key (shown once — store it).
2. Smoke test:

   ```bash
   curl http://localhost:20128/v1/chat/completions \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer YOUR_KEY" \
     -d '{"model":"auto","messages":[{"role":"user","content":"Hello!"}]}'
   ```

   The response carries `X-OmniRoute-Decision` (strategy / provider / latency) and
   `X-OmniRoute-*` cost/usage headers.

> Copy-paste quickstarts (Python / Node / PHP / cURL): `examples/quickstart/`.

---

## 5. Point your coding tools at OmniRoute

Every tool uses the same three values:

```text
Base URL : http://localhost:20128/v1
API Key  : <from Dashboard → Endpoints>
Model    : auto            (or any provider/model prefix)
```

### 5.1 One-command setup (recommended)

Let OmniRoute write each tool's own config from the live model catalog:

```bash
omniroute setup-codex      # ~/.codex/*.config.toml profiles
omniroute setup-claude     # ~/.claude/profiles/<name>/settings.json
omniroute setup-opencode   # ~/.config/opencode/opencode.json
omniroute setup-cline      # Cline CLI + VS Code extension
omniroute setup-kilo       # Kilo Code
omniroute setup-continue   # ~/.continue/config.yaml
omniroute setup-cursor     # prints Cursor's in-app steps
omniroute setup-roo        # Roo Code
omniroute setup-crush      # Crush
omniroute setup-goose      # Goose
omniroute setup-aider      # ~/.aider.conf.yml
omniroute setup-qwen       # ~/.qwen/settings.json + .env
```

Zero-config launchers (write no config, just inject env):

```bash
omniroute launch          # Claude Code
omniroute launch-codex --model auto
```

Remote instance: any `setup-*` accepts `--remote <url> --api-key <key>`; use `--dry-run` to preview.

### 5.2 Manual / editors without a Bearer header

If a tool can't send `Authorization: Bearer`, use the tokenized base URL:

```text
Base URL   : http://localhost:20128/api/v1/vscode/YOUR_KEY/
Models URL : http://localhost:20128/api/v1/vscode/YOUR_KEY/models
Chat URL   : http://localhost:20128/api/v1/vscode/YOUR_KEY/chat/completions
```

Full per-tool guide: `docs/reference/CLI-TOOLS.md`, `docs/guides/CLI-INTEGRATIONS.md`.

---

## 6. Routing: `auto` and custom combos

### 6.1 Zero-config `auto` variants

| Model ID       | Optimizes for                                               |
| -------------- | ----------------------------------------------------------- |
| `auto`         | Balanced default (LKGP — sticks to your last good provider) |
| `auto/coding`  | Quality-first for code generation                           |
| `auto/fast`    | Lowest latency first                                        |
| `auto/cheap`   | Cheapest per token first                                    |
| `auto/offline` | Most quota / rate-limit headroom first                      |
| `auto/smart`   | Quality-first + 10% exploration                             |

### 6.2 Build your own combos — 19 strategies

Dashboard → **Combos** chains models with automatic fallback across **19 strategies**: `priority`,
`fill-first`, `weighted`, `round-robin`, `p2c`, `least-used`, `random`, `strict-random`,
`cost-optimized`, `headroom`, `reset-window`, `reset-aware`, `context-relay`,
`context-optimized`, `cache-optimized`, `lkgp`, `auto`, `fusion` (panel + judge), `pipeline`.

Resilience (circuit breakers, per-key cooldown, per-model lockout) is automatic — see
`docs/routing/AUTO-COMBO.md` and `docs/architecture/RESILIENCE_GUIDE.md`.

---

## 7. Compression (save 15–95% tokens)

Enable stacked compression in **Settings → AI** (or via the Context/Cache pages):

- **RTK** — command-aware compression (shell, git, test, build, package, Docker, infra, JSON, stack traces).
- **Caveman** — language-aware rule packs (EN + DE/FR/JA/Chinese).
- **Compression Combos** — named pipelines like `rtk → caveman`; the default stacked math reaches
  ~89% average and 78–95% eligible-context savings.

Docs: `docs/compression/COMPRESSION_ENGINES.md`, `docs/compression/COMPRESSION_GUIDE.md`.

---

## 8. Protocol integrations: MCP + A2A

### 8.1 MCP server (109 tools, 3 transports)

```bash
omniroute --mcp                     # stdio transport
# HTTP transports:
#   Streamable HTTP  →  /api/mcp/stream
```

Register with an MCP client (Claude Code example):

```bash
claude mcp add-server omniroute --type http --url http://localhost:20128/api/mcp/stream
```

Cursor / Cline (`mcpServers` JSON): `"command": "omniroute", "args": ["--mcp"]`.

> Full docs: `open-sse/mcp-server/README.md`, `docs/frameworks/MCP-SERVER.md`.

### 8.2 A2A server (JSON-RPC 2.0)

```bash
curl http://localhost:20128/.well-known/agent.json   # Agent Card

curl -X POST http://localhost:20128/a2a \
  -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":"1","method":"message/send","params":{
        "skill":"quota-management",
        "messages":[{"role":"user","content":"Give me a short quota summary."}]}}'
```

> Docs: `src/lib/a2a/README.md`, `docs/frameworks/A2A-SERVER.md`.

---

## 9. Memory, Skills, Cloud Agents, Guardrails

- **Memory** — off by default. Enable in **Settings → AI**; supports int8 vector quantization,
  typed decay, and per-request `x-omniroute-no-memory`. Docs: `docs/frameworks/MEMORY.md`.
- **Skills** — extensible sandboxed skill framework. Docs: `docs/frameworks/SKILLS.md`.
- **Cloud Agents** — Codex Cloud, Devin, Jules. Docs: `docs/frameworks/CLOUD_AGENT.md`.
- **Guardrails** — prompt-injection guard (on by default), opt-in credential masking (redacts
  leaked API keys/secrets in both directions), PII redaction, free DuckDuckGo last-resort search.
  Docs: `docs/security/GUARDRAILS.md`.
- **Webhooks** — event-driven integrations. Docs: `docs/frameworks/WEBHOOKS.md`.
- **Evals** — evaluation suites. Docs: `docs/frameworks/EVALS.md`.

---

## 10. The full API surface

One endpoint, many capabilities (all OpenAI-compatible, auth via `Authorization: Bearer`):

| Capability          | Path                       |
| ------------------- | -------------------------- |
| Chat Completions    | `/v1/chat/completions`     |
| Responses API       | `/v1/responses`            |
| Models              | `/v1/models`               |
| Embeddings          | `/v1/embeddings`           |
| Reranking           | `/v1/rerank`               |
| Image generation    | `/v1/images/generations`   |
| Video generation    | `/v1/videos/generations`   |
| Audio transcription | `/v1/audio/transcriptions` |
| Audio translation   | `/v1/audio/translations`   |
| Text-to-speech      | `/v1/audio/speech`         |
| OCR                 | `/v1/ocr`                  |
| Moderations         | `/v1/moderations`          |
| WebSocket bridge    | `/v1/ws`                   |

Reference: `docs/reference/API_REFERENCE.md`, `docs/openapi.yaml`.

---

## 11. Remote access & deployment

- **Tunnels** — Dashboard → Endpoints offers Cloudflare Quick Tunnel, Tailscale Funnel, and
  ngrok. Docs: `docs/ops/TUNNELS_GUIDE.md`.
- **Remote mode** — drive a remote OmniRoute with scoped tokens: `omniroute connect`,
  `omniroute contexts`, `omniroute tokens`; plus `setup-* --remote <url>`. Docs: `docs/guides/REMOTE-MODE.md`.
- **Split-port mode** — `PORT=20128 DASHBOARD_PORT=20129 omniroute` (API and dashboard separated).
- **Docker** — `docs/guides/DOCKER_GUIDE.md`; Compose files: `docker-compose.yml`, `docker-compose.prod.yml`.
- **Desktop (Electron)** — `npm run electron:dev` / `electron:build[:win|:mac|:linux]`.
- **Termux / PWA** — `docs/guides/TERMUX_GUIDE.md`, `docs/guides/PWA_GUIDE.md`.

---

## 12. Useful env vars to know about

Edit `.env` and restart to change behavior. Highlights (full list in `docs/reference/ENVIRONMENT.md`):

| Variable                                | Purpose                                                |
| --------------------------------------- | ------------------------------------------------------ |
| `INITIAL_PASSWORD`                      | Seeds the first dashboard password                     |
| `REQUIRE_API_KEY`                       | Require an API key for all `/v1/*` calls               |
| `AUTH_COOKIE_SECURE`                    | Secure cookies (set `true` behind HTTPS)               |
| `PORT` / `API_PORT` / `DASHBOARD_PORT`  | Ports                                                  |
| `OMNIROUTE_ALLOW_PRIVATE_PROVIDER_URLS` | Enable self-hosted providers (Ollama, LM Studio, vLLM) |
| `OMNIROUTE_BASE_PATH`                   | Serve under a reverse-proxy subpath                    |
| `REDIS_URL`                             | Opt-in Redis rate limiter                              |
| `ENABLE_TLS_FINGERPRINT`                | TLS JA3/JA4 fingerprint spoofing (anti-blocking)       |
| `OMNIROUTE_MCP_ENFORCE_SCOPES`          | Scope-based access control for MCP                     |

---

## 13. CLI cheat sheet

```bash
omniroute                    # start server (http://localhost:20128)
omniroute setup              # guided password + first-provider onboarding
omniroute doctor             # local health checks (no server)
omniroute providers …        # discover / list / validate / test providers
omniroute config …           # CLI tool configuration
omniroute status             # offline status (version, DB, tools)
omniroute logs --follow      # stream usage logs
omniroute update             # check / apply updates
omniroute --mcp              # start MCP server (stdio)
omniroute launch[-codex]     # zero-config Claude Code / Codex launchers
omniroute setup-<tool>       # configure one of 12+ coding tools
omniroute --help
```

---

## 14. Troubleshooting

| Symptom                                                   | Fix                                                                                              |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `ECONNREFUSED 127.0.0.1:20128` during boot                | Normal — proxy not listening yet; clears once the dev server is up.                              |
| `Error cleaning compression_run_telemetry: no such table` | Harmless pre-existing cleanup-log bug; no data loss.                                             |
| Node version warning                                      | Use Node 22 LTS (`>=22 <23` or `>=24 <27`).                                                      |
| Self-hosted provider (Ollama/LM Studio) rejected          | Set `OMNIROUTE_ALLOW_PRIVATE_PROVIDER_URLS=true`.                                                |
| Redis error spam with no Redis running                    | Remove/comment `REDIS_URL` (in-memory limiter is the default).                                   |
| Provider 403/429 on one account                           | Per-connection cooldown is automatic; check **Resilience** dashboard for breaker/cooldown state. |

More: `docs/guides/TROUBLESHOOTING.md`, `docs/reference/RELAY_TROUBLESHOOTING.md`.

---

## 15. Where to go next

- [README.md](README.md) — overview, providers, combos, support
- [docs/getting-started/QUICK-START.md](docs/getting-started/QUICK-START.md) — 3-minute start
- [docs/guides/SETUP_GUIDE.md](docs/guides/SETUP_GUIDE.md) — full setup reference
- [docs/guides/USER_GUIDE.md](docs/guides/USER_GUIDE.md) — dashboard walkthrough
- [docs/reference/API_REFERENCE.md](docs/reference/API_REFERENCE.md) — API + OpenAPI
- [docs/reference/ENVIRONMENT.md](docs/reference/ENVIRONMENT.md) — every env var
- [docs/guides/FEATURES.md](docs/guides/FEATURES.md) — feature gallery
