# Project 1: RAG Eval Lab

> A production-style RAG service over public SEC filings. Every retrieval technique is **measured**,
> and quality regressions are **blocked in CI**.

**Target roles:** GenAI / Applied AI Engineer (primary), Agent Engineer (agentic mode)
**Gaps it closes:** evals (Ragas/DeepEval, LLM-as-judge, CI gates), observability (Langfuse/OTel),
reranking, measured retrieval quality, and **agentic RAG with trajectory evals**
**Status:** 🟡 Design and build plan done, including agentic mode (no code yet)

> **New here?** Start with [0 · Start here](docs/00-start-here.md): the project in plain English, one
> question's journey through the system, and which document to read next.

## Design documents

| # | Document | What it answers |
|---|---|---|
| 0 | [Start Here](docs/00-start-here.md) | The project in plain English, a question's journey step by step, reading paths, FAQ |
| 1 | [Requirements](docs/01-requirements.md) | What we're building, for whom, scope, success metrics, capacity estimates |
| 2 | [High-Level Architecture](docs/02-architecture.md) | Components, data flows, sequence diagrams, deployment |
| 3 | [Low-Level Design](docs/03-low-level-design.md) | Data model, API contracts, pipeline internals, configuration |
| 4 | [Evaluation Design](docs/04-evaluation-design.md) | Golden dataset, metrics, judge calibration, ablation plan, CI gate |
| 5 | [Non-Functional Design](docs/05-non-functional.md) | Latency, cost, observability, security, failure modes, scaling |
| 6 | [Architecture Decision Records](docs/06-decisions.md) | Why each major choice was made, and the alternatives considered |
| 7 | [Tech Stack](docs/07-tech-stack.md) | Every technology used, why it was chosen, alternatives rejected, what it adds to your profile |
| 8 | [Build Plan](docs/08-build-plan/README.md) | 20 features in 6 milestones: master dependency diagram, timeline, and a page per feature with diagrams, tasks and acceptance criteria |
| 9 | [Setup Guide](docs/09-setup-guide.md) | Accounts and API keys (and when you need them), `.env`, first run, cost safety, troubleshooting |
| 10 | [Glossary](docs/10-glossary.md) | Plain-English definitions of every term, and where each one is used |

## The system at a glance

```mermaid
flowchart LR
    subgraph Offline["Offline: Ingestion"]
        EDGAR[(SEC EDGAR<br/>10-K filings)] --> Parse[Parse<br/>Docling]
        Parse --> Chunk[Chunk +<br/>contextual headers]
        Chunk --> Embed[Embed]
    end

    subgraph Store["Storage: PostgreSQL"]
        PG[(pgvector HNSW<br/>+ full-text index<br/>+ metadata)]
    end

    subgraph Online["Online: Query"]
        Q([User question]) --> RW[Query rewrite]
        RW --> HY[Hybrid retrieval<br/>dense + sparse → RRF]
        HY --> RR[Rerank]
        RR --> GEN[Generate answer<br/>with citations]
        GEN --> A([Streamed answer])
    end

    subgraph Agentic["Agent mode (complex questions)"]
        AQ([Comparison /<br/>multi-hop question]) --> AG[LangGraph agent<br/>plan → tools → reflect]
        AG -->|"search_filings"| HY
        AG -->|"evidence pool"| GEN
    end

    subgraph Quality["Quality loop"]
        GS[(Golden set<br/>150 Qs, versioned)] --> EV[Eval runner<br/>Ragas + DeepEval + judge]
        EV --> CI{CI gate}
        EV --> TJ[Trajectory evals<br/>agent vs pipeline]
    end

    Embed --> PG
    HY <--> PG
    EV -. calls .-> RW
    Online -. traces .-> LF[Langfuse / OTel]
    EV -. scores .-> LF
```

## Planned deliverables
1. A FastAPI service with SSE streaming, answering questions about 10-K filings with citations
2. A config-driven retrieval pipeline, so each technique can be switched on or off for ablations
3. A 150-question golden dataset, versioned in git
4. An eval harness plus a GitHub Actions gate that fails PRs on quality regressions
5. An ablation results table and a Langfuse latency/cost dashboard
6. An agentic mode (LangGraph research agent) plus an **agent-vs-pipeline report** that decides, per
   question type, when the agent is worth its cost
7. A blog/LinkedIn post about the findings
