# 10. Setup Guide: accounts, keys and the first run

> **Status:** No code exists yet. This page is the **target setup**: commands become real as F0–F6 are
> built and may change slightly. Keep this page in sync with the repo as you go.

## 10.1 What you need, and when

| Needed from | Tool / account | Used for | Cost |
|---|---|---|---|
| **M1** | Git, **Docker Desktop** (≥ 8 GB RAM for Docker; Keycloak + Postgres + LGTM are the heavy ones) | Local stack | Free |
| M1 | **uv** (Python 3.12), **Node 24 LTS + pnpm** | Python and TS workspaces | Free |
| M1 | **MCP Inspector** (`npx @modelcontextprotocol/inspector`) | Debugging servers | Free |
| M1 | **Claude Desktop** and/or **VS Code** | Interop testing | Free tiers |
| M1 (release) | **PyPI** account with a trusted publisher for the repo; GitHub account for the **MCP Registry** namespace (`io.github.<you>/…`) | Publishing india-mf-mcp | Free |
| M2 | **Anthropic** and **OpenAI** API keys | Own client + evals (two model families) | Pay per use |
| M2 | **Langfuse Cloud** (same account as Project 1) | LLM traces | Free tier |
| M4 | **npm** account with provenance publishing | Publishing fx-rates-mcp | Free |
| M4 (full plan) | **GitHub OAuth app** (read-only scopes) | F16 GitHub upstream | Free |
| M5 | **k6** | Load tests | Free |
| M6 | **Azure** subscription, Azure CLI, **Terraform**, a domain (optional) | Deployment | Low with scale to zero; set a budget alert |

## 10.2 Environment variables (`.env`)

```bash
# --- Core (M1) ---
DATABASE_URL=postgresql+asyncpg://hub:hub@localhost:5432/hub
REDIS_URL=redis://localhost:6379/0
AMFI_USER_AGENT="india-mf-mcp/0.1 (+https://github.com/<you>/mcp-hub)"

# --- Identity (M2) ---
KEYCLOAK_URL=http://localhost:8080          # must be the SAME hostname for browser and containers
KEYCLOAK_REALM=mcp-hub
HUB_RESOURCE=http://localhost:8000/mcp       # the gateway's resource identifier (token audience)
GATEWAY_CLIENT_ID=gateway
GATEWAY_CLIENT_SECRET=                       # for token exchange

# --- LLMs (M2, client + evals) ---
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=https://cloud.langfuse.com

# --- Gateway secrets (M3) ---
CONFIRM_HMAC_KEYS='{"2026-11":"<random 32 bytes, base64>"}'   # key ring with kid
TOKEN_KEK=<random 32 bytes, base64>          # envelope-encryption key (Key Vault in the cloud)

# --- Safety limits ---
EVAL_MAX_TOKENS_PER_RUN=3000000
```

Generate random keys with `python -c "import secrets,base64;print(base64.b64encode(secrets.token_bytes(32)).decode())"`.

## 10.3 First run

```mermaid
flowchart LR
    A["1 · clone · uv sync ·<br/>pnpm install"] --> B["2 · .env"]
    B --> C["3 · make up<br/>(pg, redis, keycloak,<br/>opa, lgtm, nginx)"]
    C --> D["4 · make migrate"]
    D --> E["5 · ingest AMFI<br/>(daily + small backfill)"]
    E --> F["6 · server over stdio<br/>in Inspector / Claude Desktop"]
    F --> G["7 · (M3) client → gateway<br/>login + tool call"]
```

```bash
# 1–2
git clone <your-repo-url> && cd mcp-hub
uv sync && pnpm install
cp .env.example .env

# 3–4
make up && make migrate
curl -s localhost:8080/realms/mcp-hub/.well-known/openid-configuration | head -c 200

# 5. Data (small backfill first; the full 10 years takes a while)
uv run india-mf ingest daily
uv run india-mf ingest backfill --years 1

# 6. Try the server locally
npx @modelcontextprotocol/inspector uv run india-mf-mcp --transport stdio --data postgres

# 7. (From M3) Through the gateway with the own client
uv run hubctl login --hub http://localhost:8000/mcp       # opens the browser (Keycloak: alice / see realm README)
uv run hubctl chat --model claude "Compare 5y CAGR of two large-cap index funds"
```

**Claude Desktop config (local stdio, after the v0.1 release):**
```json
{
  "mcpServers": {
    "india-mf": { "command": "uvx", "args": ["india-mf-mcp"] }
  }
}
```

**You're set up correctly when:**
- [ ] `make up` shows all containers healthy and Grafana has a test trace
- [ ] Inspector lists the six read tools with output schemas
- [ ] (M2) `hubctl login` returns a token with `aud = HUB_RESOURCE` and a `tenant` claim
- [ ] (M3) Two gateway replicas answer alternately (check the `x-replica` response header) and the call appears in the audit log

## 10.4 Keeping costs safe

1. Spending limits on the Anthropic and OpenAI accounts before the first eval run.
2. Develop evals with the **scripted fake LLM** first; switch to real models only for smoke and final runs.
3. Keep the LLM response cache on; run the full matrix only on demand.
4. `EVAL_MAX_TOKENS_PER_RUN` stops runaway eval runs.
5. Azure: budget alert, scale to zero for servers, `terraform destroy` when the demo isn't needed.

## 10.5 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Tokens rejected with "invalid issuer" | Keycloak reached as `keycloak:8080` inside Docker but `localhost:8080` from the browser, so `iss` differs | Set Keycloak's hostname explicitly so both see the same issuer URL (or use one hostname via `/etc/hosts`) |
| Endless 401 loop in the client | Token audience ≠ `HUB_RESOURCE` | Check the audience mapper / `resource` parameter (F4 spike notes) |
| `403 insufficient_scope` every time | Client re-requests only the new scope | Request the **union** of old and new scopes |
| Streamed responses arrive all at once | nginx buffering | `proxy_buffering off` for `/mcp` |
| Confirmation fails on retry | Different HMAC keys on the two replicas, or Redis down | Same `CONFIRM_HMAC_KEYS` everywhere; check Redis |
| Tool disappeared from `tools/list` | Definition changed → quarantined | `hubctl tools diff <name>` then approve |
| Claude Desktop doesn't show tools | Wrong config path or `uvx` not on PATH | Use the full path to `uvx`; check the app's MCP logs |
| Keycloak very slow to start | First start builds its caches | Wait; give Docker more memory |
| Clock-skew token errors | Container clock drift | Leeway is 60 s; resync the host clock |
