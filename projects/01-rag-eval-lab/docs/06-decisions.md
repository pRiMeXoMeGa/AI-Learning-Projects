# 6. Architecture Decision Records

Format: **Context → Decision → Alternatives → Consequences.** Status is *Proposed* until it is validated
while building.

---

### ADR-001: Use SEC 10-K filings of CPG companies as the corpus
- **Context:** We need a public, realistic, hard corpus with ground truth we can verify.
- **Decision:** Last 3 years of 10-Ks for 20 consumer/CPG companies from EDGAR.
- **Alternatives:** Wikipedia (too easy, already in model training data); RBI/SEBI circulars (good, but
  less structured, with fewer numeric tables); synthetic company documents (not credible).
- **Consequences:** Real tables and numbers make numeric-accuracy evaluation possible, and the domain
  links to your CPG experience. HTML parsing of tables needs care.

### ADR-002: PostgreSQL + pgvector as the only datastore
- **Context:** About 50k vectors across index versions; we need vector, full-text and metadata filters.
- **Decision:** Postgres 16 with pgvector (HNSW, `halfvec`) and built-in full-text search.
- **Alternatives:** Pinecone (you've already used it, so it adds nothing new to the portfolio; managed
  cost); Qdrant (excellent, but a second datastore to run); Azure AI Search (strong hybrid, but costly and
  vendor-specific).
- **Consequences:** One system to operate; SQL joins for metadata; a transactional ingest. Scale ceiling
  and path are documented in [05 §5.6](05-non-functional.md). Pinecone and Qdrant can be added later as a
  retriever adapter for a comparison ablation.

### ADR-003: Store embeddings as 1024-dimensional `halfvec`
- **Context:** pgvector HNSW limits: 2,000 dims (`vector`), 4,000 (`halfvec`). Large embedding models output 3,072 dims.
- **Decision:** Request 1024-dim embeddings (Matryoshka truncation) and store them as `halfvec(1024)`.
- **Alternatives:** Full 3,072 dims as `halfvec` (4× memory); a smaller model.
- **Consequences:** Small storage and fast search. The quality impact is measured explicitly in ablation A-emb.

### ADR-004: Start with Postgres full-text search; switch to real BM25 only if needed
- **Context:** `ts_rank_cd` is not BM25, and hybrid quality depends on the sparse signal.
- **Decision:** Start with built-in FTS. If A2 shows sparse recall is the bottleneck, switch to ParadeDB
  `pg_search` (BM25 inside Postgres).
- **Consequences:** Simple start. The switch is a retriever adapter change, measured as its own ablation.

### ADR-005: Reciprocal Rank Fusion for hybrid search
- **Decision:** RRF with k = 60 over dense and sparse lists (and over multi-query lists).
- **Alternatives:** Weighted score fusion (needs normalising scores that aren't comparable, plus tuning);
  learned fusion (overkill).
- **Consequences:** No tuning needed and robust. The reranker does the fine ordering.

### ADR-006: Rerank with a cross-encoder (Cohere Rerank; local BGE reranker as an alternative)
- **Decision:** Rerank 40 fused candidates to the top 8.
- **Consequences:** Expected to be the biggest precision gain, at a cost of ~300–400 ms. Comparing the
  local BGE reranker against Cohere is a cost/latency data point for the README.

### ADR-007: Label ground truth as evidence character spans, not chunk IDs
- **Context:** Chunking strategy is one of the variables being tested.
- **Decision:** The golden set stores `(accession_no, char_start, char_end)`. Relevance is computed by
  overlap (≥ 50% of the span).
- **Consequences:** One golden set works for every chunking strategy. The parser must produce **stable
  character offsets**, so parsed text is stored and treated as the canonical coordinate system. Re-parsing
  with a new parser version requires re-anchoring spans (a script does fuzzy re-matching).

### ADR-008: Pipelines as declarative YAML plus config hashing
- **Decision:** Each pipeline variant is a YAML file validated by Pydantic. Its canonical hash is attached
  to every trace and eval run.
- **Consequences:** An ablation is a diff; results are reproducible. We need discipline: no hidden
  behaviour outside config.

### ADR-009: The eval runner calls the pipeline in-process, not over HTTP
- **Decision:** `evals` imports `ragkit` directly.
- **Consequences:** Fast, deterministic CI with no server to start, and the same code path as production.
  The HTTP layer is covered separately by contract tests.

### ADR-010: Gate on deterministic metrics tightly, on judged metrics loosely
- **Decision:** Tight tolerances for recall, numeric and abstention accuracy (no LLM involved). Looser
  tolerances plus absolute floors for faithfulness and correctness (LLM-judged).
- **Consequences:** Fewer flaky CI failures. Judge trust depends on the calibration κ (see
  [04 §4.5](04-evaluation-design.md)).

### ADR-011: Judge model from a different family than the generator
- **Decision:** Generator and judge come from different providers/families.
- **Consequences:** Less self-preference bias; two provider keys are needed.

### ADR-012: Langfuse for tracing, prompts and eval runs
- **Alternatives:** LangSmith (tied to the LangChain ecosystem, hosted); Arize Phoenix (strong, OTel-native).
- **Decision:** Langfuse (open source, self-hostable, OTel ingestion, datasets + scores). This fills an
  explicit résumé gap.
- **Consequences:** Use Langfuse Cloud's free tier during development to avoid running its self-hosted
  dependencies (ClickHouse, object storage). Self-hosting is documented as an option.

### ADR-013: No LangChain / LlamaIndex in the core pipeline
- **Context:** You've already shown framework usage. This project is about showing you understand the
  mechanics.
- **Decision:** Plain Python + provider SDKs + SQL. Docling is used for parsing; Ragas and DeepEval are
  used only in `evals`.
- **Alternatives:** LlamaIndex (fast to build, but it hides the retrieval internals we want to measure and
  explain).
- **Consequences:** More code, but every stage is transparent, traceable and easy to explain in interviews.
  A LlamaIndex re-implementation can be an optional comparison.

### ADR-014: Azure Container Apps + Azure Postgres Flexible Server for the cloud deployment
- **Decision:** Deploy to Azure with Terraform. It matches your résumé and common JD requirements
  (Azure-heavy GCC/enterprise roles).
- **Alternatives:** AWS ECS + RDS (documented equivalent); Fly.io/Render (cheaper, less résumé value).
- **Consequences:** Scale-to-zero keeps demo cost low. Postgres Flexible Server supports the pgvector
  extension (it must be allow-listed in the server parameters).
