# 11. Market Alignment Review (September 2026)

> **Question answered:** Does every approach in Project 1 match how RAG is actually built, evaluated and
> hired for in 2026, and what's missing?
>
> **Method:** ~30 web searches across vendor docs, 2026 benchmarks, arXiv papers (2025–2026), practitioner
> write-ups and job-market guides (sources at the end). Each area of the design was compared with current
> practice and rated. Recommended changes were then **applied to the design docs**; optional ones are listed
> as a backlog.

## 11.1 Verdict

**The design is well aligned, and in several places ahead of typical 2026 practice.** The evidence
strongly supports its three core bets:

1. **Evaluation first.** Retrieval is where RAG fails most of the time (one 2026 industry analysis puts it
   at 73% of failures), and systematic evaluation from day one is now common but not yet universal (~60%
   of new deployments). Span-level ground truth, confidence intervals and a *calibrated* judge put this
   project in the minority that does it properly. Judge calibration is called "the practice most teams
   under-invest in".
2. **Hybrid retrieval + a cross-encoder reranker** is the recommended 2026 default, and a FinanceBench
   study found that **contextual chunk metadata** (our contextual headers) gave the largest gain on 10-K
   questions, with the reranker essential for precision.
3. **Agentic RAG as a measured alternative, not a default.** An ACL 2026 industry paper found neither
   enhanced nor agentic RAG universally better, with agentic up to **3.6× more expensive**, and that an
   explicit reranking step inside agent pipelines helps. That is exactly the F18/F19 design (the agent's
   search tool reuses the reranker, and a router is set from measured results).

**But nine areas needed updating**, mostly because the tooling moved on in 2026:

| # | Gap | Why it matters in 2026 | Change applied |
|---|---|---|---|
| 1 | **Structured financial data (XBRL) not used** | Numeric questions are where financial RAG fails most; SEC publishes every reported number as XBRL facts, and current 10-K systems (e.g. Red Hat's agentic GraphRAG for EDGAR) use them | XBRL ingestion, agent tool `get_financial_fact`, golden numeric answers cross-checked against XBRL |
| 2 | **Sparse retrieval isn't true BM25** | `ts_rank_cd` isn't BM25; real BM25 is now native in Postgres (`pg_textsearch` v1.0, April 2026; ParadeDB `pg_search`); one 2026 text+table benchmark found BM25 beating `text-embedding-3-large` on most metrics | BM25 becomes an explicit ablation row (A2b), no longer conditional |
| 3 | **Embedding candidates are 2024-era** | 2026 leaders include Qwen3-Embedding (open), Voyage, Cohere Embed v4, Gemini Embedding; open models now match closed ones on retrieval | A-emb compares `text-embedding-3-large` (baseline) with Qwen3-Embedding-0.6B (local) and one current commercial model |
| 4 | **Reranker candidates are dated** | Cohere Rerank v4, zerank-2, Voyage rerank-2.5, and open Qwen3-Reranker / Jina v3 lead 2026 comparisons; BGE-v2-m3 is still the most deployed open one | A-rr compares Cohere Rerank v4 with Qwen3-Reranker and bge-reranker-v2-m3 |
| 5 | **No long-context baseline** | "RAG vs. long context" is one of 2026's most-asked design questions; a single 10-K (~80k tokens) fits in today's context windows | New ablation **A-LC**: whole filing(s) in context vs. RAG, on single-filing questions (smoke subset, prompt caching) |
| 6 | **No MCP interface** | JDs and guides now treat MCP as the standard way agents consume retrieval | The RAG service also exposes `search_filings` and `ask_filings` as an **MCP server** (FastMCP) |
| 7 | **Ingestion-time poisoning not covered** | OWASP LLM08 (vector & embedding weaknesses): hidden text (e.g. `display:none`, white-on-white in HTML) passes through extraction and steers answers | Hidden-text detection in parsing; poisoned-document cases in the golden security set |
| 8 | **Nugget/claim-level correctness not named** | Claim-level (RAGChecker) and nugget-based (TREC RAG AutoNuggetizer) scoring correlate better with humans than holistic scores | Our `key_facts` *are* nuggets: add **nugget recall** as an explicit metric alongside the judge |
| 9 | **Query rewriting assumed to help** | 2026 benchmarks: with a strong reranker, rewriting/HyDE add little, especially for **numeric** queries; HyDE underperforms on financial documents | A4 stays as an ablation with a **null hypothesis**; HyDE explicitly excluded; filter extraction (company/year scoping) emphasised instead |

**Effort impact:** about **+9.5 h** of recommended work, partly offset by moving A-llmctx to *Could*, so the plan
goes from ~94 h to **~102 h** (details in §11.4). Project 1 stays at **7 weeks** at ~14.5 h/week (the upper
part of the 12–15 h/week range); the F17 feedback loop stays a stretch item. The roadmap doesn't change.

## 11.2 Scorecard

✅ aligned · ⚠️ updated · ➕ added · 🔸 optional backlog · ⛔ considered and rejected (with reason)

| Area | Status | Summary |
|---|---|---|
| Problem framing: measured, eval-gated RAG | ✅ | Matches the 2026 shift to "ship evals alongside features" |
| Corpus: SEC 10-Ks of 20 CPG companies | ✅ ➕ | Same document type as FinanceBench; **XBRL facts added** |
| Parsing: Docling | ✅ ⚠️ | Docling's TableFormer is strong on financial tables; **hidden-text detection added** |
| Chunking: fixed / structural / tables / contextual headers / parent-child | ✅ | 2026 studies: complex chunking rarely beats simple baselines; contextual metadata is the big win |
| Semantic chunking, late chunking | ⛔ | See §11.3 |
| Embeddings | ⚠️ | Candidate list updated |
| Vector store: pgvector (HNSW, `halfvec`, iterative scan) | ✅ 🔸 | Production-grade in 2026; binary quantization + rescoring as an optional efficiency ablation |
| Sparse retrieval | ⚠️ | True BM25 ablation added |
| Learned sparse (SPLADE / BGE-M3 sparse) | 🔸 | Optional ablation |
| Fusion: RRF | ✅ | Still the standard |
| Reranking | ⚠️ | Candidate list updated |
| Query understanding | ⚠️ | Kept as an ablation with a null hypothesis; HyDE excluded |
| Metadata filters (company / year / section) | ✅ | Domain-scoped retrieval reduces "vector search dilution" (2026 paper) |
| Long context vs. RAG | ➕ | A-LC ablation |
| Generation: numbered sources, citation validation, abstention | ✅ 🔸 | Optional A-cite: provider-native citations (Anthropic `search_result` blocks) vs. our parser |
| Reasoning / thinking budgets for numeric answers | 🔸 | Optional A-think on the numeric subset; the agent's `calculate` tool covers the core need |
| Agentic RAG (LangGraph, budgets, router) | ✅ ➕ | Strongly supported by evidence; **XBRL tool added** |
| GraphRAG / LightRAG | ⛔ | See §11.3 |
| Multimodal / ColPali | ⛔ | See §11.3 |
| Eval: span-based retrieval metrics, bootstrap CIs | ✅ | Ahead of common practice |
| Eval: LLM judge with κ calibration | ✅ 🔸 | Optional: optimise the judge prompt with DSPy GEPA to raise κ |
| Eval: nugget / claim-level correctness | ➕ | Nugget recall from `key_facts` |
| Eval: external benchmark | 🔸 | Optional: FinanceBench open 150-question subset as an external check |
| Golden-set governance (splits, holdout, review) | ✅ ⚠️ | Review metadata fields added (reviewer, decision, source revision) |
| CI gate with tolerances, pinned judge | ✅ | Matches 2026 guidance (tolerance bands, pinned judge) |
| Tools: Ragas + DeepEval | ✅ | Both remain the standard open-source pair |
| Observability: Langfuse + OTel | ✅ ⚠️ | Langfuse was acquired by ClickHouse (Jan 2026) and stays open source; OTel GenAI conventions are still "Development", so pin the version |
| Security: sources as data, injection probes | ✅ ➕ | Ingestion poisoning added |
| Serving: FastAPI + SSE | ✅ ➕ | MCP interface added |
| Embedding fine-tuning | 🔸 | Optional A-ft (synthetic pairs from non-golden filings) |
| Semantic caching, model routing | ✅ | Deliberately in Project 7 |
| Deployment: Terraform → Azure Container Apps | ✅ | — |

## 11.3 Considered and rejected (and how to defend it in an interview)

| Technique | Why it's not in the core plan | Evidence |
|---|---|---|
| **Semantic chunking** | Highest retrieval recall in one evaluation (91.9%) but lower end-to-end accuracy than recursive 512-token splitting (54% vs. 69%); fragments too small for 10-K tables and explanations | Chroma / Vecta 2026 benchmarks; SIGIR 2026 chunking taxonomy ("rarely significant wins over simple baselines") |
| **Late chunking** | Needs a long-context embedding model with token-level pooling (not supported by the chosen embedding APIs); contextual headers address the same problem cheaply and are proven on FinanceBench | Metadata-driven financial RAG paper; SIGIR 2026 study |
| **HyDE** | Underperforms plain dense retrieval on financial documents; limited value for numeric queries with a strong reranker | 2026 text+table retrieval benchmark; query-rewriting studies |
| **GraphRAG / LightRAG** | Helps "global" synthesis questions over a corpus; our questions are mostly fact, table and comparison questions. The XBRL tool gives most of the structured-data benefit at a fraction of the cost. Guidance: only adopt a graph if a pilot beats the hybrid baseline | 2026 GraphRAG decision guides; Red Hat's EDGAR system uses XBRL-derived structure rather than LLM-extracted graphs |
| **ColPali / visual retrieval** | The big gains are on **PDFs** with complex layouts (62% vs. 84% recall on financial PDFs). EDGAR 10-Ks are **HTML**, with tables parsed as HTML tables, so the visual advantage mostly disappears | ColPali / REAL-MM-RAG benchmarks; FinRAGBench-V |
| **LangChain / LlamaIndex in the core** | Transparency of each measured stage (ADR-013). JDs still list them; mention the optional LlamaIndex comparison | 2026 RAG-engineer JD guides |

## 11.4 Changes applied to the design

| # | Change | Where | Effort |
|---|---|---|---|
| A1 | Ingest **XBRL company facts** for the 20 companies (`data.sec.gov` companyfacts API) into an `xbrl_fact` table | F1, [03 §3.1](03-low-level-design.md) | +2 h |
| A2 | Agent tool **`get_financial_fact(ticker, concept, fiscal_year)`**; answers cite the filing chunk, with the XBRL value as a cross-check | F18 | +1 h |
| A3 | Golden set: numeric answers **cross-checked against XBRL** where a concept exists (flags labelling errors) | F11 | +0.5 h |
| A4 | Ablation **A2b: BM25** (`pg_textsearch` or `pg_search`) vs. `ts_rank_cd`; check availability on Azure Flexible Server | F5, [04 §4.7](04-evaluation-design.md), ADR-004 | +1 h |
| A5 | Updated **embedding** candidates for A-emb | F4, [07](07-tech-stack.md) | 0 h (config) |
| A6 | Updated **reranker** candidates, new ablation A-rr | F6, [07](07-tech-stack.md) | 0 h (config) |
| A7 | Ablation **A-LC**: long-context baseline on single-filing questions (smoke subset) | F14, ADR-019 | +1.5 h |
| A8 | **MCP server** exposing `search_filings` and `ask_filings` (FastMCP, same pipeline) | F9, ADR-020 | +2 h |
| A9 | **Hidden-text detection** at parse time + poisoned-document probes in the golden set | F2, F11, [05 §5.4](05-non-functional.md) | +1 h |
| A10 | **Nugget recall** metric from `key_facts`; review-metadata fields in the golden schema | F12, F11, [04](04-evaluation-design.md) | +0.5 h |
| A11 | A4 (query rewriting) hypothesis restated as "no gain expected with a strong reranker"; HyDE excluded | [04 §4.7](04-evaluation-design.md) | 0 h |
| | **Total added** | | **≈ +9.5 h** |
| | Offset: A-llmctx → Could (saves ~2 h of runs and cost); F17 feedback loop already stretch | F14, F17 | −2 h |

## 11.5 Optional backlog (add only if time allows)

| ID | Item | Why it's interesting | Effort |
|---|---|---|---|
| O1 | **FinanceBench open 150** as an external benchmark | Credibility: comparable with published numbers | ~3 h |
| O2 | **A-cite**: provider-native citations vs. our `[n]` parser | Shows you know the managed alternative and its trade-offs | ~1 h |
| O3 | **A-think**: reasoning mode / thinking budget on numeric questions | Measures whether "just use a reasoning model" fixes numeric errors | ~0.5 h |
| O4 | **Binary quantization + rescoring** in pgvector | Efficiency story: ~32× smaller index for a small recall cost | ~1 h |
| O5 | **Learned sparse** (BGE-M3 sparse weights) as a third retriever | Three-way hybrid is a current research direction | ~2 h |
| O6 | **Judge prompt optimisation with DSPy GEPA** to raise κ | 2026 production case studies report large κ gains | ~2 h |
| O7 | **Embedding fine-tuning** on synthetic pairs from non-golden filings (A-ft) | Reported ~7% retrieval gains with a few thousand pairs | ~3 h |

## 11.6 Market keywords this project now covers

Hybrid search (dense + **BM25**), rerankers, contextual retrieval, pgvector, metadata filtering,
**structured + unstructured retrieval (XBRL)**, agentic RAG (LangGraph), **long-context vs. RAG routing**,
**MCP**, RAG evaluation (Ragas, DeepEval, LLM-as-judge, calibration, **nugget-based**), CI eval gates,
Langfuse, OpenTelemetry, OWASP LLM08 (RAG security), FastAPI streaming, Terraform on Azure.

## 11.7 Sources

**RAG practice and trends**
- [RAG Techniques Compared: 2026 guide (Starmorph)](https://blog.starmorph.com/blog/rag-techniques-compared-best-practices-guide)
- [Enterprise RAG in 2026 (Atolio)](https://www.atolio.com/blog/enterprise-rag-guide)
- [RAG Production Guide 2026 (Lushbinary)](https://lushbinary.com/blog/rag-retrieval-augmented-generation-production-guide/)
- [Is Agentic RAG worth it? (ACL 2026 Industry)](https://aclanthology.org/2026.acl-industry.5/)
- [From Naive RAG to Deep Agentic Retrieval (arXiv 2607.24791)](https://arxiv.org/abs/2607.24791)
- [How to hire RAG engineers in 2026 (Kore1)](https://www.kore1.com/hire-rag-engineers-2026/)
- [RAG Engineer skills 2026 (Second Talent)](https://www.secondtalent.com/occupations/rag-engineer/)

**Financial RAG**
- [Metadata-Driven RAG for Financial QA (arXiv 2510.24402)](https://arxiv.org/abs/2510.24402)
- [From BM25 to Corrective RAG: text-and-table benchmark (arXiv 2604.01733)](https://arxiv.org/html/2604.01733v1)
- [Stop chunking tables: agentic GraphRAG for financial disclosures (Red Hat)](https://developers.redhat.com/articles/2026/07/22/how-we-built-agentic-graphrag-financial-disclosures)
- [finrag-eval: RAG over 10-Ks measured on FinanceBench (GitHub)](https://github.com/krish-117/finrag-eval)
- [SEC EDGAR APIs (XBRL company facts)](https://www.sec.gov/search-filings/edgar-application-programming-interfaces)
- [FinRAGBench-V (arXiv 2505.17471)](https://arxiv.org/pdf/2505.17471)

**Retrieval components**
- [Best embedding models for RAG 2026 (PremAI)](https://www.premai.io/blog/best-embedding-models-for-rag-2026-ranked-by-mteb-score-cost-and-self-hosting/)
- [8 embedding models compared, 2026 (Tensoria)](https://tensoria.fr/en/blog/embedding-models-2026-guide)
- [Best rerankers for RAG 2026 (FutureAGI)](https://futureagi.com/blog/best-rerankers-for-rag-2026/)
- [Reranker leaderboard (Agentset)](https://agentset.ai/rerankers)
- [Postgres vector search compared 2026](https://www.web3aiblog.com/blog/postgres-vector-search-compared-pgvector-pgvectorscale-paradedb-lantern-2026)
- [pg_textsearch v1.0 (PostgreSQL news)](https://www.postgresql.org/about/news/pg_textsearch-v10-3264)
- [Hybrid search in PostgreSQL (ParadeDB)](https://www.paradedb.com/blog/hybrid-search-in-postgresql-the-missing-manual)
- [pgvector quantization guide (halfvec, binary)](https://tomodahinata.com/en/blog/pgvector-index-tuning-hnsw-ivfflat-quantization-iterative-scan-guide)
- [Hybrid search: BM25, SPLADE and vectors (PremAI)](https://www.premai.io/blog/hybrid-search-for-rag-bm25-splade-and-vector-search-combined/)
- [HyDE, multi-query, decomposition: what moves recall (DEV)](https://dev.to/gabrielanhaia/hyde-multi-query-decomposition-which-query-rewrite-actually-moves-recall-1m08)
- [Better Together: query rewriting under a strong baseline (arXiv 2609.05637)](https://arxiv.org/html/2609.05637)

**Chunking and parsing**
- [When is complex chunking worth it? (arXiv 2608.16586)](https://arxiv.org/html/2608.16586)
- [Chunking taxonomy and evaluation (SIGIR 2026)](https://doi.org/10.1145/3805712.3808575)
- [RAG chunking strategies 2026 (Firecrawl)](https://www.firecrawl.dev/blog/best-chunking-strategies-rag)
- [Docling vs Marker vs MinerU 2026](https://adityamangal98.medium.com/docling-vs-marker-vs-mineru-the-ultimate-open-source-pdf-parser-benchmark-2026-which-is-best-a36ecbb6c6b1)
- [ColPali multimodal document RAG (Spheron)](https://www.spheron.network/blog/colpali-multimodal-document-rag-gpu-cloud/)

**Long context**
- [Long-context vs RAG decision framework (TianPan)](https://tianpan.co/blog/2026-04-09-long-context-vs-rag-production-decision-framework)
- [Long context vs RAG: when 1M windows replace RAG (SitePoint)](https://www.sitepoint.com/long-context-vs-rag-1m-token-windows/)

**Evaluation**
- [Best RAG evaluation tools 2026 (Braintrust)](https://www.braintrust.dev/articles/best-rag-evaluation-tools)
- [LLM evaluation tools comparison 2026 (Inference.net)](https://inference.net/content/llm-evaluation-tools-comparison/)
- [Ragas metrics (docs)](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/)
- [The Great Nugget Recall (TREC RAG, arXiv 2504.15068)](https://arxiv.org/pdf/2504.15068)
- [Golden evaluation sets for RAG (DevLeader)](https://www.devleader.ca/2026/09/09/golden-evaluation-sets-for-rag-synthetic-data-human-calibration-and-drift)
- [DSPy optimizers incl. GEPA (FutureAGI)](https://futureagi.com/blog/dspy-optimizers-explained/)

**Observability, citations, security, MCP**
- [OTel GenAI semantic conventions status 2026 (DEV)](https://dev.to/azena-ai/opentelemetrys-genai-semantic-conventions-are-not-stable-yet-heres-what-actually-shipped-in-2026-3mke)
- [Langfuse alternatives 2026, incl. acquisition news (OpenObserve)](https://openobserve.ai/blog/langfuse-alternatives/)
- [Claude search result blocks (docs)](https://platform.claude.com/docs/en/build-with-claude/search-results)
- [OWASP LLM08: Vector and embedding weaknesses](https://genai.owasp.org/llmrisk/llm08-excessive-agency/)
- [RAG + MCP servers 2026 overview](https://visionvix.com/best-mcp-servers-for-rag/)
