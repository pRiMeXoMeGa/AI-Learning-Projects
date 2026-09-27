# 10. Setup Guide: accounts, keys and the first run

> **Status:** No code exists yet. This page is the **target setup**: commands become real as F0–F8 are
> built and may change slightly. Keep this page in sync with the repo as you go.

## 10.1 What you need, and when

| Needed from | Tool / account | Used for | Cost |
|---|---|---|---|
| **M1** | Git, **Docker Desktop** (≥ 6 GB RAM for Docker; Postgres + Langfuse) | Local stack | Free |
| M1 | **uv** (Python 3.12) | Workspace, running everything | Free |
| M1 | **MCP Inspector** (`npx @modelcontextprotocol/inspector`, needs Node) | Checking opsdesk-mcp | Free |
| **M2** | **Anthropic** API key | Raw loop, LangGraph, Claude Agent SDK, OpenAI SDK via LiteLLM (E1) | Pay per use |
| M2 | **OpenAI** API key | E2 model swap; OpenAI mini model for dev/CI | Pay per use |
| M2 | **Langfuse** (local in compose, or Langfuse Cloud as in Projects 1–2) | Traces | Free tier / self-hosted |
| M3 | Nothing new: the Claude Agent SDK bundles its CLI | — | — |
| M4 | **Node 24 LTS + pnpm** | Approval inbox (React) | Free |
| M5 | **PyPI** account with a trusted publisher for the repo | Publishing `opssim` | Free |
| M6 (full plan) | **Azure** subscription, Azure CLI, **Terraform**, a GitHub OAuth app for demo login | Public demo | Low with scale to zero; set a budget alert |

**Provider limits:** before running full experiments, check your API tier's requests-per-minute and
tokens-per-minute limits. They decide how many parallel workers you can use ([06 §6.1](06-non-functional.md#61-throughput-and-wall-time)).

## 10.2 Environment variables (`.env`)

```bash
# --- Core (M1) ---
DATABASE_URL=postgresql://harness:harness@localhost:5432/harness
OPSSIM_URL=http://localhost:8100
OPSSIM_RUNS_DIR=./.runs                  # per-run SQLite files + Claude SDK session dirs

# --- Approvals (M2) ---
APPROVALS_URL=http://localhost:8200
APPROVAL_HMAC_KEYS='{"2027-01":"<random 32 bytes, base64>"}'   # key ring with kid

# --- Models (M2) ---
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
MODELS_FILE=config/models.yaml           # dated model IDs: main, cheap, judge
PRICES_FILE=config/prices.yaml

# --- Tracing (M2) ---
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=http://localhost:3000      # or https://cloud.langfuse.com

# --- Safety limits ---
HCTL_MAX_COST_USD=20                     # default cap per `hctl run`; raise per experiment deliberately
HCTL_PARALLEL=4                          # start low; raise after checking provider limits
```

Generate random keys with `python -c "import secrets,base64;print(base64.b64encode(secrets.token_bytes(32)).decode())"`.

## 10.3 First run

```mermaid
flowchart LR
    A["1 · clone ·<br/>uv sync"] --> B["2 · .env"]
    B --> C["3 · make up<br/>(postgres, langfuse)"]
    C --> D["4 · opssim serve<br/>+ approvals"]
    D --> E["5 · hctl qa<br/>(oracle, no model)"]
    E --> F["6 · hctl play S1-01<br/>(you are the agent)"]
    F --> G["7 · one live run,<br/>cheap model"]
    G --> H["8 · small matrix<br/>+ report"]
```

```bash
# 1–2
git clone <repo> && cd agent-reliability-harness
uv sync
cp .env.example .env        # fill in keys

# 3–4
make up
uv run opssim serve &                     # MCP server + management API on :8100
uv run python -m services.approvals &     # approval service on :8200

# 5. Prove scenarios and graders work (free)
uv run hctl qa

# 6. Solve one scenario by hand
uv run hctl play S1-01

# 7. One live run on the cheap model
uv run hctl run --impl raw --model cheap --scenarios S1-01 --k 1 --max-cost 1

# 8. A small matrix and a report
uv run hctl run --impl raw,langgraph --model cheap --scenarios dev:10 --k 2 --max-cost 5
uv run hctl report results/latest
```

## 10.4 Keeping costs safe

| Control | Setting |
|---|---|
| Cost cap per command | `--max-cost` (default from `HCTL_MAX_COST_USD`); the runner stops scheduling when reached |
| Cheap model for development | `--model cheap` until a feature works; the main model only for recorded experiments |
| Stub models in tests | Unit and adapter tests never call a real API unless marked `@live` |
| Prompt caching | On in every adapter; check the cached-token share in reports |
| CI smoke | 20 runs on the cheap model, ≤ $1 per PR |
| Provider-side limits | Set a monthly spend limit in the Anthropic and OpenAI consoles |
| Demo | Per-user daily run cap and cheap model (F17/F21) |

## 10.5 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Runs interfere with each other | Two runs share a run ID or SQLite file | Check `POST /runs` returns unique IDs; isolation test in F2 |
| Many `infra_error` results | Provider 429s at high parallelism | Lower `HCTL_PARALLEL`; check limiter settings |
| Claude Agent SDK resume fails after a kill | Session directory not per run, or deleted | Set the per-run session dir under `OPSSIM_RUNS_DIR` (F11) |
| Built-in tools appear in Claude Agent SDK runs | Allowed-tools list or settings sources not restricted | See F11 options; the schema snapshot test should fail |
| OpenAI SDK traces go to OpenAI | Default exporter still on | Replace the SDK's trace processors with the OTel processor only (F12); don't just disable tracing, or Langfuse gets nothing |
| LiteLLM → Claude errors on tool calls | Adapter mismatch (beta) | Pin versions; fallback in [08 §8.6](08-tech-stack.md#86-things-to-verify-in-the-first-week-of-building) |
| Oracle fails a scenario after a simulator change | Rule or generator regression | `hctl qa -v S1-04`; fix before any agent runs |
| Report refuses to render | Results hash or partial matrix mismatch | Re-run missing cells or pass `--allow-partial` (marked in the report) |
