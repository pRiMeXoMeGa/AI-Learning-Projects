# 2. High-Level Architecture

## 2.1 Design principles

1. **Everything is measurable.** Every stage emits a trace span with inputs, outputs, latency and tokens.
2. **Pipelines are configuration, not code branches.** A retrieval pipeline is a YAML file. An ablation
   is a diff between two YAML files.
3. **Evaluate chunk-independently.** Ground truth is labelled as *evidence spans in the source document*,
   not chunk IDs, so the same golden set can score any chunking strategy (see [04](04-evaluation-design.md)).
4. **Retrieved text is untrusted input.** It is delimited, never executed, and never allowed to override
   system instructions.
5. **Agents must earn their cost.** The agentic mode shares retrieval and answer generation with the
   fixed pipeline and is compared with it on the same golden set. It's used only where it measurably wins.
6. **Boring infrastructure.** Postgres does vector, full-text and metadata storage. There is no extra database
   until the numbers justify one.

## 2.2 System context (C4 level 1)

```mermaid
flowchart TB
    Analyst([Analyst])
    Dev([Developer])
    GH[GitHub Actions]

    subgraph System["RAG Eval Lab"]
        direction TB
        Core[RAG service + Eval harness]
    end

    EDGAR[(SEC EDGAR)]
    LLM[LLM provider<br/>Azure OpenAI / Anthropic / OpenAI]
    EMB[Embedding provider]
    RER[Reranker<br/>Cohere API or local BGE]
    LF[Langfuse]

    Analyst -->|ask questions| Core
    Dev -->|change configs, inspect traces| Core
    GH -->|run smoke eval on PR| Core
    Core -->|download filings| EDGAR
    Core -->|generate, rewrite, judge| LLM
    Core -->|embed| EMB
    Core -->|rerank| RER
    Core -->|traces, scores| LF
    Dev -->|dashboards| LF
```

## 2.3 Containers (C4 level 2)

```mermaid
flowchart TB
    subgraph Clients
        UI[Streamlit UI]
        CLI[rag-lab CLI<br/>ingest / eval / compare]
        CI[GitHub Actions]
    end

    subgraph App["Application"]
        API[Query API<br/>FastAPI, SSE]
        WRK[Ingestion worker<br/>Celery or arq]
        EVR[Eval runner<br/>Python package]
    end

    subgraph Data["Data"]
        PG[(PostgreSQL 16<br/>pgvector + FTS)]
        RD[(Redis<br/>job queue + caches)]
        BLOB[(Object storage<br/>raw + parsed filings)]
        GIT[(Git repo<br/>golden set + configs)]
    end

    subgraph External
        LLMP[LLM / embedding /<br/>reranker APIs]
        LFS[Langfuse]
    end

    UI -->|HTTP/SSE| API
    CLI --> API
    CLI --> EVR
    CI --> EVR
    API --> PG
    API --> RD
    API --> LLMP
    WRK --> BLOB
    WRK --> PG
    WRK --> LLMP
    RD --> WRK
    EVR -->|calls pipeline in-process| API
    EVR --> PG
    EVR --> GIT
    API -. OTel traces .-> LFS
    EVR -. scores .-> LFS
```

| Container | Responsibility | Tech |
|---|---|---|
| **Query API** | Online query pipeline **and agentic mode**, SSE streaming, document/eval-run read APIs | FastAPI, Pydantic v2, asyncio, LangGraph (agent mode only) |
| **Ingestion worker** | Download → parse → chunk → embed → index, incremental and idempotent | arq (async Redis queue) or Celery |
| **Eval runner** | Runs a pipeline config over a golden-set version, scores it, stores results, enforces the gate | Python, Ragas, DeepEval, custom metrics |
| **PostgreSQL** | Documents, chunks, vectors (pgvector HNSW), full-text (tsvector GIN), eval runs | Postgres 16 + pgvector ≥ 0.7 |
| **Redis** | Job queue; embedding cache; LLM response cache (evals only) | Redis 7 |
| **Object storage** | Raw EDGAR HTML and parsed Docling JSON (for re-chunking without re-parsing) | Local volume / Azure Blob |
| **Langfuse** | Traces, prompt versions, eval scores, cost dashboards | Langfuse Cloud (free tier) or self-hosted |

> The eval runner imports the **same pipeline package** the API uses (in-process), rather than calling it
> over HTTP. This keeps CI fast and deterministic, and guarantees we evaluate the exact code that ships.

## 2.4 Internal module structure

```mermaid
flowchart LR
    subgraph ragkit["ragkit (shared core package)"]
        direction TB
        CFG[config<br/>PipelineConfig YAML]
        ING[ingestion<br/>fetch · parse · chunk · embed]
        RET[retrieval<br/>dense · sparse · fusion · filters]
        RNK[rerank]
        QRY[query<br/>rewrite · decompose · self-query]
        GEN[generation<br/>prompt · citations · abstain]
        AGT[agent<br/>LangGraph graph · tools · guard · router]
        PRV[providers<br/>LLM · embeddings · reranker adapters]
        OBS[observability<br/>tracing decorators]
        STO[storage<br/>repositories]
    end

    AGT -. "tools call" .-> RET
    AGT -. "tools call" .-> RNK
    AGT -. "answer via" .-> GEN
    API[api] --> ragkit
    WORK[worker] --> ING
    EVAL[evals] --> ragkit
```

## 2.5 Data flow A: ingestion (offline)

```mermaid
flowchart LR
    A[EDGAR full-text / submissions API] -->|filing index| B{Content hash<br/>changed?}
    B -- no --> SKIP[Skip]
    B -- yes --> C[Download primary<br/>10-K HTML → blob]
    C --> D[Parse with Docling<br/>→ DoclingDocument JSON → blob]
    D --> E[Section detection<br/>Item 1, 1A, 7, 7A, 8…]
    E --> F[Chunker<br/>strategy from config]
    F --> G[Tables → separate chunks<br/>serialised as Markdown + caption]
    F --> H[Contextual header<br/>company · FY · section · heading path]
    G --> I[Embed batch<br/>with cache]
    H --> I
    I --> J[(Upsert into chunks<br/>for index_version)]
    J --> K[Refresh tsvector<br/>+ HNSW index]
```

**Key points**
- **Parse once, chunk many times.** The parsed Docling JSON is stored, so a new chunking strategy doesn't
  re-download or re-parse anything.
- **Index versions.** `(chunking_strategy, embedding_model)` → `index_version`. Several versions live
  side by side, and a pipeline config picks one.
- **Idempotent.** Filing identity = accession number; content identity = SHA-256 of the HTML. Chunk
  identity = hash(index_version, document_id, char_start, char_end).
- **SEC fair-access rules:** a descriptive `User-Agent` header with contact email, and ≤ 10 requests/second.

## 2.6 Data flow B: online query

```mermaid
sequenceDiagram
    autonumber
    actor U as Analyst
    participant API as Query API
    participant QP as Query processor
    participant R as Retriever
    participant PG as Postgres
    participant RR as Reranker
    participant G as Generator (LLM)
    participant LF as Langfuse

    U->>API: POST /v1/query {question, filters?, config?}
    API->>LF: start trace
    API->>QP: process(question)
    QP->>G: rewrite + extract filters (small, fast model, JSON output)
    G-->>QP: {rewritten_queries[], filters{company, fy, section}}
    par dense
        R->>PG: ANN search (HNSW, cosine) top-50 + filters
    and sparse
        R->>PG: full-text search (ts_rank_cd) top-50 + filters
    end
    PG-->>R: candidates
    R->>R: Reciprocal Rank Fusion (k=60) → top-40
    R->>RR: rerank(query, 40 candidates)
    RR-->>R: scores → top-8 (drop below min score)
    R->>R: parent/neighbour expansion, dedupe, token budget
    alt no chunk above abstain threshold
        API-->>U: SSE: "Not found in the filings" + closest sources
    else evidence found
        API->>G: stream(system prompt + numbered sources + question)
        G-->>API: tokens…
        API-->>U: SSE token events
        API->>API: parse citations [n] → chunk ids, validate
        API-->>U: SSE: citations event + done event
    end
    API->>LF: spans: rewrite, dense, sparse, fusion, rerank, generate (+tokens, cost)
```

## 2.6b Data flow B2: agentic query (`mode: agent`, or `auto` for complex questions)

```mermaid
sequenceDiagram
    autonumber
    actor U as Analyst
    participant API as Query API
    participant RT as Router
    participant AG as Agent graph (LangGraph)
    participant GD as Guard
    participant T as Tools
    participant G as Generator (F8)
    participant CP as Checkpointer (Postgres)

    U->>API: POST /v1/query {question, mode: "auto"}
    API->>RT: route(QueryPlan.type)
    alt factoid / other
        RT-->>API: pipeline → Data flow B
    else comparison / multi-hop
        RT->>AG: run(question, filters)
        AG->>AG: plan → sub-questions
        API-->>U: SSE step (plan)
        loop until finish() or budget exhausted
            AG->>GD: proposed tool calls
            GD-->>AG: allowed / duplicate / budget exhausted
            AG->>T: search_filings · read_chunk_context · list_filings · calculate (parallel)
            T-->>AG: results → evidence pool
            AG->>CP: checkpoint state
            API-->>U: SSE step (tools, n_results, ms)
        end
        AG->>G: answer(question, evidence pool, calculations)
        G-->>API: tokens + citations (same contract as the pipeline)
        API-->>U: SSE token … citations … done {mode, steps, tool_calls}
    end
```

**Key points**
- The agent **reuses** the pipeline's retrieval (as the `search_filings` tool) and its answer generator, so
  any retrieval improvement from the ablations helps both modes.
- The guard is plain Python and runs **before** every tool call: step, tool-call, token and time budgets,
  plus duplicate-call detection.
- Tools are read-only. The worst an injected instruction in a filing can do is waste the agent's budget,
  which the guard caps.

## 2.7 Data flow C: evaluation & CI gate

```mermaid
flowchart TB
    PR[Pull request changes<br/>code / prompt / config] --> GA[GitHub Actions: eval-smoke job]
    GA --> LOAD[Load golden set vX<br/>smoke split: 30 Qs]
    LOAD --> RUN[Run pipeline in-process<br/>for each question, concurrency 8]
    RUN --> CACHE{LLM response<br/>cache hit?}
    CACHE -- yes --> SC
    CACHE -- no --> LLM[Call LLM] --> SC
    SC[Score<br/>retrieval metrics · Ragas · DeepEval · custom judge] --> STORE[(eval_runs +<br/>eval_results)]
    STORE --> CMP[Compare with baseline run<br/>from main branch]
    CMP --> GATE{Any metric drop ><br/>tolerance?}
    GATE -- yes --> FAIL[❌ Fail check<br/>+ PR comment with diff table]
    GATE -- no --> PASS[✅ Pass<br/>+ PR comment with scores]
    STORE -. scores .-> LF[Langfuse dataset run]

    NIGHT[Nightly schedule on main] --> FULL[Full eval: 150 Qs] --> BASE[Update baseline]
```

## 2.8 Deployment view

**Local (primary for development and CI)**

```mermaid
flowchart LR
    subgraph compose["docker compose"]
        api[api:8000]
        worker[worker]
        ui[streamlit:8501]
        pg[(postgres + pgvector)]
        redis[(redis)]
    end
    api --- pg
    api --- redis
    worker --- pg
    worker --- redis
    ui --> api
    api -.-> ext[LLM APIs · Langfuse Cloud]
```

**Cloud (Azure, to match your Azure experience)**

```mermaid
flowchart LR
    GHA[GitHub Actions<br/>build · test · eval] -->|push image| ACR[Azure Container Registry]
    ACR --> ACA_API[Container Apps: api]
    ACR --> ACA_W[Container Apps: worker]
    ACR --> ACA_UI[Container Apps: ui]
    ACA_API --> PGF[(Azure Database for PostgreSQL<br/>Flexible Server + pgvector)]
    ACA_W --> PGF
    ACA_API --> RC[(Azure Cache for Redis)]
    ACA_W --> BLOB[(Blob Storage)]
    ACA_API --> KV[Key Vault]
    ACA_API --> AOAI[Azure OpenAI]
    ACA_API -.-> LFC[Langfuse Cloud]
```

Infrastructure is defined in **Terraform** (`infra/`), and the API scales from 0–3 replicas on HTTP
concurrency. An AWS equivalent is ECS Fargate + RDS Postgres (pgvector) + ElastiCache + Bedrock.

## 2.9 Proposed repository layout

```
01-rag-eval-lab/
├── docs/                      # these design docs
├── configs/
│   ├── pipelines/             # A0-naive.yaml … A7-full.yaml (ablations), AG1-agent.yaml, AG2-auto.yaml
│   └── prompts/               # versioned prompt templates
├── data/golden/               # golden set JSONL (v1, v2 …) + human labels
├── src/ragkit/                # core package (see 2.4), incl. ragkit/agent/ (LangGraph)
├── src/api/                   # FastAPI app
├── src/worker/                # ingestion jobs
├── src/evals/                 # runner, metrics, gate, report
├── ui/                        # Streamlit
├── infra/                     # Terraform (Azure)
├── .github/workflows/         # ci.yml, eval-smoke.yml, eval-nightly.yml
├── docker-compose.yml
└── pyproject.toml             # uv-managed
```
