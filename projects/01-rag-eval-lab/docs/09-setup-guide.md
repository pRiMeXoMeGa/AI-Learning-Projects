# 9. Setup Guide: accounts, keys and the first run

> **Status:** No code exists yet. This page is the **target setup**: the commands below become real as
> F0 (Foundation) and F1–F4 are built, and command names may change slightly along the way. Update this
> page as you go, so it always matches the repo.

## 9.1 What you need, and when

You don't need everything on day 1. Set things up when the milestone that needs them begins.

| Needed from | Tool / account | What it's used for | Cost |
|---|---|---|---|
| **M1** (week 1) | Git, **Docker Desktop** (≥ 8 GB RAM for Docker) | Postgres, Redis, API and worker containers | Free |
| M1 | **uv** (installs Python 3.12 for you) | Python packages and virtual env | Free |
| M1 | **SEC EDGAR** (no account) | Downloading 10-K filings. Needs only a `User-Agent` with your name and email | Free |
| M1 | **Azure OpenAI** *or* **OpenAI** API key | Embeddings and answer generation | Pay per use |
| M2 | **Langfuse Cloud** account (free tier) | Traces, prompts, eval scores | Free tier |
| M2 | **Anthropic** API key | LLM judge (a different model family from the generator) | Pay per use |
| M3 | **Cohere** API key (trial key is fine) | Reranker. Alternative: the local BGE reranker, no key needed | Trial / pay per use |
| M4 | **GitHub** repo with Actions enabled, plus repository secrets | CI eval gate | Free for public repos |
| M6 | **Azure subscription**, Azure CLI, **Terraform** | Cloud deployment | Low with scale to zero; set a budget alert |

## 9.2 Environment variables (`.env`)

`.env.example` is committed; `.env` is **never** committed (it's in `.gitignore`).

```bash
# --- Required from M1 ---
DATABASE_URL=postgresql+psycopg://raglab:raglab@localhost:5432/raglab
REDIS_URL=redis://localhost:6379/0
SEC_USER_AGENT="Your Name your.email@example.com"   # SEC requires a real contact

# Generator + embeddings: fill in ONE provider
AZURE_OPENAI_ENDPOINT=https://<your-resource>.openai.azure.com/
AZURE_OPENAI_API_KEY=
OPENAI_API_KEY=

# --- From M2 ---
ANTHROPIC_API_KEY=                                   # judge
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=https://cloud.langfuse.com

# --- From M3 ---
COHERE_API_KEY=                                      # or set rerank.provider: bge-local

# --- Safety limits ---
EVAL_MAX_TOKENS_PER_RUN=2000000                      # budget guard aborts above this
API_KEYS=dev-key-change-me                           # keys accepted by the query API
```

Model IDs aren't in `.env`. They live in `configs/models.yaml` under the aliases `fast`, `strong` and
`judge` ([03 §3.3](03-low-level-design.md)), so changing a model is a one-line, reviewable diff.

## 9.3 First run (end of M1)

```mermaid
flowchart LR
    A["1 · clone + uv sync"] --> B["2 · cp .env.example .env<br/>fill keys"]
    B --> C["3 · make up<br/>(docker compose)"]
    C --> D["4 · make migrate"]
    D --> E["5 · ingest 2 companies"]
    E --> F["6 · ask a question<br/>with curl"]
    F --> G["7 · (M2) run the eval"]
```

```bash
# 1. Get the code and dependencies
git clone <your-repo-url> && cd rag-eval-lab
uv sync                                   # creates .venv with Python 3.12

# 2. Configure
cp .env.example .env                      # then fill in the keys for M1

# 3–4. Start the stack and create the tables
make up                                   # postgres (pgvector), redis, api, worker
make migrate                              # alembic upgrade head
curl localhost:8000/readyz                # → {"db":"ok","redis":"ok"}

# 5. Ingest a small slice first (2 companies, 3 years)
uv run rag-lab ingest fetch   --tickers PEP,KO --years 2022-2024
uv run rag-lab ingest process --index-version v1-fixed-512__emb-large-1024

# 6. Ask a question (streams Server-Sent Events)
curl -N localhost:8000/v1/query \
  -H "X-API-Key: dev-key-change-me" -H "Accept: text/event-stream" \
  -H "Content-Type: application/json" \
  -d '{"question": "What were PepsiCo'\''s net revenues in fiscal 2024?", "pipeline": "A0"}'

# 7. (From M2) Score the baseline on golden set v0
uv run rag-lab eval run --pipeline A0 --golden v0
```

**From M5**, the same query works in agent mode by adding `"mode": "agent"` (or `"auto"`) to the request
body; you'll see `step` events before the answer.

**You're set up correctly when:**
- [ ] `readyz` returns ok for both the database and Redis
- [ ] `ingest fetch` run twice downloads nothing the second time
- [ ] The curl call streams `meta` → `token`… → `citations` → `done`
- [ ] (M2) The query shows up as a trace in Langfuse, with one span per stage

## 9.4 Keeping costs safe

1. **Set a monthly spending limit** on every provider account before your first call (Azure OpenAI,
   OpenAI, Anthropic, Cohere). Pick an amount you're comfortable losing to a bug.
2. **Develop on 2 companies**, not 20. Only ingest the full corpus in M3.
3. Use the **smoke split (30 questions)** while iterating; run `full` only when you need final numbers.
4. Keep the **eval response cache** on (the default in eval mode): re-running an unchanged config costs
   close to nothing.
5. The **budget guard** (`EVAL_MAX_TOKENS_PER_RUN`) stops an eval run before it goes over the limit.
6. In agent mode, the **guard's budgets** (steps, tool calls, tokens, time) cap each run.
7. In Azure (M6), use **scale to zero** and a **budget alert** on the resource group, and run
   `terraform destroy` when the demo isn't needed.

## 9.5 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `extension "vector" is not available` | Plain Postgres image instead of the pgvector one | Use `pgvector/pgvector:pg16` in `docker-compose.yml` |
| EDGAR returns `403` | Missing or generic `User-Agent` | Set `SEC_USER_AGENT` to your real name and email |
| EDGAR returns `429` or you get blocked | More than 10 requests/second | Keep the fetcher's rate limiter on; wait 10 minutes |
| First parse is very slow | Docling downloads its layout models on first use | Wait once; the models are cached afterwards (mount the cache as a Docker volume) |
| Docker containers get killed | Not enough memory for Docling + Postgres | Give Docker ≥ 8 GB RAM, or parse outside Docker with `uv run` |
| No traces in Langfuse | Wrong keys or host, or traces not flushed before exit | Check the three `LANGFUSE_*` values; call the flush on shutdown in CLI commands |
| Judge scores look random | Judge from the same family as the generator, or an uncalibrated rubric | Check `judge` ≠ generator family in `models.yaml`; run `rag-lab eval calibrate` |
| Eval run is expensive every time | Cache disabled, or config hash changes on every run | Check the cache is on; make sure no timestamps or random values end up in the config |
| Port 5432 / 8000 already in use | Another local Postgres or app | Stop it, or change the host ports in `docker-compose.yml` |
| Agent run stops early with `budget_exceeded` | Budgets too small for the question, or the agent is looping | Check the trace for duplicate calls first; raise budgets only if the calls were all useful |
