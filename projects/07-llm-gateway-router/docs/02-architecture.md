# 2. High-Level Architecture

## 2.1 Design principles

1. **Measure every saving against its quality cost.** No cache or router ships without a number for what
   it saves and what it breaks.
2. **Never trade correctness for a cache hit across trust boundaries.** Caches are scoped by tenant,
   model, system-prompt version and tool set. Anything built from untrusted content isn't cached by
   default.
3. **Cheap checks first.** The request pipeline runs from cheapest to most expensive: auth → budget →
   exact cache → semantic cache → routing → provider.
4. **Fail open for reliability, fail closed for money and safety.** A cache outage means a miss. A budget
   store outage means deny (configurable). A router outage means the default model.
5. **The provider's numbers are the truth.** Cost comes from provider-reported usage, including cached
   tokens, not from our estimates.
6. **Small, pinned, auditable.** A gateway sees every prompt and every key, so dependencies are few,
   pinned and scanned. This is the lesson of the 2026 LiteLLM compromise.

## 2.2 System context (C4 level 1)

```mermaid
flowchart TB
    APPS(["Apps: P1 · P5 · P6 · load generator"])
    ADM(["Platform admin / team lead"])
    subgraph SYS["Switchboard"]
        G["gateway + admin + ledger"]
    end
    ANT["Anthropic API"]
    OAI["OpenAI API"]
    EMB["Embedding model<br/>(semantic cache)"]
    OBS["Prometheus · Grafana · Langfuse"]
    APPS -->|"OpenAI-compatible HTTPS"| G
    ADM -->|"admin API / CLI / Grafana"| G
    G --> ANT & OAI
    G --> EMB
    G -.-> OBS
```

## 2.3 Containers (C4 level 2)

```mermaid
flowchart TB
    subgraph GW["gateway (Python, FastAPI + uvicorn, stateless replicas)"]
        IN["ingress: parse · auth (virtual key) ·<br/>limits · budget check"]
        C1["exact cache lookup"]
        C2["semantic cache lookup"]
        RT["router (policy per app)"]
        PC["prompt-cache helper"]
        AD["provider adapters<br/>(official SDKs)"]
        RL["reliability: retry · breaker · fallback · timeouts"]
        ST["stream relay + usage capture"]
        LG["ledger writer (async, batched)"]
    end
    RD[("Redis<br/>limits · budgets (hot) · exact cache ·<br/>breaker state")]
    VS[("Postgres + pgvector<br/>semantic cache (per tenant) ·<br/>keys · policies · ledger")]
    CLS["router model<br/>(ONNX, in-process)"]
    PROM["Prometheus"]
    GRAF["Grafana"]
    LF["Langfuse"]
    IN --> C1 --> C2 --> RT --> PC --> RL --> AD
    AD --> ST --> LG
    IN & C1 & RL --> RD
    C2 --> VS
    LG --> VS
    RT --> CLS
    GW -.->|"metrics"| PROM --> GRAF
    GW -.->|"OTel traces"| LF
```

| Container | Responsibility |
|---|---|
| **gateway** | Everything on the request path; stateless, scales horizontally |
| **Redis** | Hot state: rate-limit buckets, budget counters, exact cache, breaker state |
| **Postgres + pgvector** | Keys (hashed), policies, semantic cache (vectors + responses, partitioned by tenant), ledger |
| **router model** | A small classifier exported to ONNX, loaded in-process (≤ 2 ms inference) |
| **Prometheus + Grafana** | Metrics and dashboards (cost, latency, cache, routing, reliability) |
| **Langfuse** | Traces (as in the other projects) |

## 2.4 The request pipeline

```mermaid
flowchart TB
    R["request (virtual key, model or alias)"] --> A{"key valid?<br/>model allowed?"}
    A -- no --> E401["401 / 403"]
    A -- yes --> L{"rate limit ok?<br/>budget remaining ≥ estimate?"}
    L -- no --> E429["429 / 402-style"]
    L -- yes --> X{"exact cache<br/>eligible + hit?"}
    X -- hit --> HIT["return cached (x-cache: hit)"]
    X -- miss --> S{"semantic cache<br/>eligible + score ≥ τ<br/>(+ verify-on-hit)?"}
    S -- hit --> SHIT["return cached (x-cache: semantic)"]
    S -- miss --> RT["route: alias → model<br/>(policy + reason)"]
    RT --> PC["prompt-cache helper<br/>(stable prefix, cache_control)"]
    PC --> CALL["provider call with timeouts,<br/>retries, breaker, fallback"]
    CALL --> STR["stream to caller;<br/>capture usage"]
    STR --> W["write caches (if eligible) ·<br/>ledger · metrics"]
```

## 2.5 Data flow A: a streamed request with fallback

```mermaid
sequenceDiagram
    autonumber
    participant C as Caller (ClauseDesk)
    participant G as Gateway
    participant A as Anthropic
    participant O as OpenAI
    C->>G: POST /v1/chat/completions (model: "smart", stream)
    G->>G: key, limits, budget, caches (miss), route → claude-mid
    G->>A: messages.stream (first-token timeout 4 s)
    A-->>G: 529 overloaded (before any token)
    G->>G: retry once (jitter) → 529 again → breaker count++
    G->>O: fallback: openai-mid (tools + JSON supported ✓)
    O-->>G: stream tokens
    G-->>C: SSE tokens (x-model-used: openai-mid, x-fallback: 1)
    O-->>G: usage (input, cached, output)
    G->>G: ledger row · metrics · (no cache write: temperature > 0)
```

A retry or fallback is only allowed **before the first token is sent** to the caller. After that, an error
is passed through, because a half-streamed answer can't be silently restarted.

## 2.6 Data flow B: semantic cache with scoping and verification

```mermaid
flowchart LR
    Q["request"] --> EL{"eligible?<br/>app opted in · no tool results ·<br/>no untrusted docs (unless opted in) ·<br/>temperature ≤ 0.3"}
    EL -- no --> MISS["skip cache"]
    EL -- yes --> K["scope key = tenant · model ·<br/>system-prompt hash · tools hash"]
    K --> EMB["embed last user turn<br/>(+ short context summary)"]
    EMB --> NN["pgvector top-1 within scope"]
    NN --> TH{"cosine ≥ τ(app)?"}
    TH -- no --> MISS
    TH -- yes --> VER{"verify-on-hit?<br/>(high-risk apps)"}
    VER -- "cheap model: 'does this answer<br/>fit this question?' → no" --> MISS
    VER -- yes/skip --> HIT["serve (x-cache: semantic, score)"]
```

## 2.7 Data flow C: cascade routing

```mermaid
sequenceDiagram
    autonumber
    participant C as Caller
    participant G as Gateway
    participant S as Small model
    participant B as Big model
    C->>G: model "auto" (policy: cascade)
    G->>S: request (+ ask for a self-rated confidence in JSON, if the app allows)
    S-->>G: answer + confidence 0.55
    G->>G: confidence < θ, or the verifier rejects → escalate
    G->>B: same request
    B-->>G: answer
    G-->>C: big model's answer (x-route: cascade:escalated)
```

Cascades cost more on hard requests (two calls) and less on easy ones. The evaluation measures the
break-even point per workload ([04 §4.4](04-evaluation-design.md#44-routing-evaluation)).

## 2.8 Deployment view

```mermaid
flowchart TB
    subgraph Azure["Azure Container Apps (same env as earlier projects)"]
        G1["gateway ×2"]
        AD["admin API"]
        PR["Prometheus + Grafana<br/>(or Azure Monitor managed Prometheus)"]
    end
    RED[("Azure Cache for Redis<br/>(or Upstash)")]
    PG[("Postgres Flexible + pgvector")]
    KV["Key Vault<br/>(provider keys)"]
    APPS["P1 · P5 · P6"] --> G1
    G1 --> RED & PG & KV
```

Provider keys live only in Key Vault and the gateway's memory. Callers only ever hold **virtual keys**.

## 2.9 Proposed repository layout

```
switchboard/
├─ gateway/            # FastAPI app: api/, auth/, limits/, cache/{exact,semantic,prompt}/, router/, providers/, reliability/, ledger/
├─ router-training/    # data prep from earlier projects' evals, classifier training, ONNX export
├─ admin/              # admin API + CLI (swctl)
├─ evals/              # trace replay, quality scoring, cache study, poisoning, Pareto, overhead, build-vs-buy
├─ traces/             # recorded, anonymized request traces from P1/P6 (no customer data)
├─ dashboards/         # Grafana JSON
├─ infra/              # Terraform, compose
└─ docs/
```
