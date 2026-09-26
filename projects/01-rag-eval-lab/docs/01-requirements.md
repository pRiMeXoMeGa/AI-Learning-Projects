# 1. Requirements

## 1.1 Problem statement

Analysts need to answer questions such as *"How did PepsiCo's gross margin change between FY2023 and
FY2024, and what did management attribute it to?"* by reading hundreds of pages of annual reports.
A naive RAG chatbot gives answers that sound plausible but are often wrong or uncited, and nobody knows
**how** wrong they are.

**Goal:** Build a RAG service whose answer quality is **measured, explainable and protected against
regressions**. Every retrieval improvement must come with numbers.

## 1.2 Users & use cases

| Actor | Use case |
|---|---|
| Analyst (end user) | Ask natural-language questions and get a streamed answer with clickable citations |
| Analyst | Filter by company, fiscal year or 10-K section |
| Analyst | Ask complex questions (comparisons across companies or years, multi-step reasoning, calculations) and see the steps the agent took |
| Developer | Change chunking, retrieval, reranking or prompts and see the quality impact before merging |
| Developer | Inspect a single query's trace (retrieved chunks, scores, prompt, latency, cost) |
| CI pipeline | Run the eval suite on each PR and block merges that regress quality |
| Operator | Ingest new filings incrementally without re-processing unchanged ones |

## 1.3 Corpus

| Property | Value |
|---|---|
| Source | SEC EDGAR 10-K annual reports (public domain) |
| Companies | 20 CPG / consumer companies (e.g. PepsiCo, Coca-Cola, P&G, Colgate-Palmolive, Kimberly-Clark, Mondelez, General Mills, Keurig Dr Pepper, Hershey, Kraft Heinz, Clorox, Church & Dwight, Estée Lauder, Tyson Foods, Conagra, Campbell's, Hormel, J.M. Smucker, McCormick, Molson Coors) |
| Years | Last 3 fiscal years, giving **about 60 filings** (check each company has 3 consecutive 10-Ks; swap out any that were acquired or file 20-F) |
| Format | EDGAR HTML (primary document) |
| Why this corpus | Real enterprise-style text: long documents, dense tables, cross-year and cross-company comparisons, and a vocabulary that is identical across companies (every 10-K has "Item 1A. Risk Factors"), which makes retrieval hard. It also relates to your CPG domain background. |

## 1.4 Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-1 | Ingest 10-K filings from EDGAR: download, parse (text + tables), chunk, embed and index | Must |
| FR-2 | Incremental ingestion: skip filings whose content hash hasn't changed | Must |
| FR-3 | Query API that returns a **streamed** answer (SSE) with inline citations `[1]`, `[2]` mapped to source chunks (filing, section, page/anchor) | Must |
| FR-4 | Metadata filters: `company`, `fiscal_year`, `section` (explicit, or extracted from the question) | Must |
| FR-5 | **Abstain** ("not found in the filings") when evidence is insufficient | Must |
| FR-6 | Retrieval pipeline stages are switchable by **config** (dense, sparse, hybrid, reranker, query rewrite, contextual headers, parent expansion) | Must |
| FR-7 | Multiple chunking strategies and embedding models can be **indexed side by side** (index versions) | Must |
| FR-8 | Eval runner: run any pipeline config against a golden-set version and store per-question and aggregate scores | Must |
| FR-9 | CI gate: GitHub Actions runs a smoke eval on PRs and fails the PR if a metric drops beyond a tolerance | Must |
| FR-10 | Trace every query and eval item in Langfuse (spans per stage, tokens, cost, latency) | Must |
| FR-11 | Compare two eval runs (diff view: which questions got better or worse) | Should |
| FR-12 | Minimal UI (Streamlit) to ask questions, view citations and browse eval runs | Should |
| FR-13 | Online feedback (thumbs up/down) attached to traces | Could |
| FR-14 | Multi-tenancy, per-user ACLs | **Won't** (Project 6 covers this) |
| FR-15 | **Agentic mode:** a read-only research agent (LangGraph) that plans, calls retrieval and calculation tools several times, and answers with the same citation and abstention contract. Request field `mode: pipeline \| agent \| auto`; `auto` routes by question type | Should |
| FR-16 | **Trajectory evals:** score the agent's tool calls (validity, selection, search coverage, efficiency, redundancy) and compare agent vs. pipeline on the same golden set | Should |
| FR-17 | Agent steps are streamed to the client (`step` SSE events) and traced as nested spans | Should |

## 1.5 Non-functional requirements

| ID | Category | Target |
|---|---|---|
| NFR-1 | Latency | Time-to-first-token (TTFT) **p50 ≤ 1.5 s, p95 ≤ 3 s**; full answer p95 ≤ 8 s |
| NFR-2 | Retrieval latency | Hybrid retrieval + rerank **p95 ≤ 600 ms** |
| NFR-3 | Quality (target after ablations) | Faithfulness ≥ 0.90, context recall@8 ≥ 0.85, abstention accuracy ≥ 0.90, numeric answer accuracy ≥ 0.80 |
| NFR-4 | Cost | ≤ ~8k input tokens + 600 output tokens per query (reported per query in Langfuse) |
| NFR-5 | Eval cost | Smoke eval (30 Qs) finishes in **≤ 5 min** in CI; full eval (150 Qs) runs nightly or on demand |
| NFR-6 | Reproducibility | Every eval run records its git SHA, pipeline config hash, golden-set version, index version and model IDs |
| NFR-7 | Security | API-key auth; secrets in env / Key Vault; retrieved text treated as untrusted (prompt-injection aware) |
| NFR-8 | Portability | Runs fully with `docker compose up`; the LLM provider can be swapped by config (Azure OpenAI / Anthropic / OpenAI) |
| NFR-9 | Availability | Portfolio scale: a single region and single replica is fine; stateless API so it can scale out |
| NFR-10 | Agent mode | First `step` event ≤ 2 s; full answer p95 ≤ 20 s; hard budgets of 6 steps, 12 tool calls, 40k tokens, 45 s; on budget exhaustion return a partial cited answer or abstain, never an error |

## 1.6 Capacity estimates (back of the envelope)

| Quantity | Estimate |
|---|---|
| Filings | 20 companies × 3 years = **60** |
| Size per 10-K | ~80–150 pages, ~60–100k tokens |
| Total corpus | ~60 × 80k ≈ **~5M tokens** |
| Chunks (512-token target, ~15% overlap) | ~5M / ~435 ≈ **~11–12k chunks** per index version |
| Index versions kept (for ablations) | ~4 (chunking strategy × embedding model) → **~50k vectors** |
| Vector storage | 50k × 1024 dims × 4 B ≈ **200 MB** (+ HNSW overhead ≈ 2×), which fits comfortably in a small Postgres |
| Embedding cost (one-off per index version) | ~5M tokens per version |
| Query load | Portfolio/demo: < 1 QPS. Design must still show how it scales (see [05](05-non-functional.md)) |
| Eval run (full) | 150 Qs × (1 generation + ~4 judge calls) ≈ **750 LLM calls** |

**Conclusion:** At this scale **one Postgres instance with pgvector** is sufficient. We don't need a
separate vector database. The engineering difficulty is in **quality measurement**, not throughput.

## 1.7 Success criteria (definition of done)

1. The ablation table shows the effect of each technique on retrieval and answer metrics, with confidence intervals.
2. The final config meets the NFR-3 quality targets, or the README explains honestly why not.
3. The judge's agreement with human labels is reported (Cohen's κ ≥ 0.6 target).
4. A demo PR that degrades retrieval is **automatically blocked** by the CI gate (included as evidence in the README).
5. A Langfuse dashboard shows p50/p95 latency and cost per query.
6. An **agent-vs-pipeline report** shows, per question type and with confidence intervals, where the agent
   is better and what it costs, and the `auto` router is configured from those numbers.

## 1.8 Out of scope
- Authentication beyond an API key, multi-tenancy, ACL-aware retrieval (→ Project 6)
- Agents that **act** (write tools, side effects), human-in-the-loop approvals, multi-agent systems and
  agent-framework comparisons (→ Project 3). The agent here is read-only and single-agent.
- Semantic caching and model routing (→ Project 7)
- Fine-tuning embedding or generation models
