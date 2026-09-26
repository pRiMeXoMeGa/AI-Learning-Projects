# 10. Glossary

Plain-English definitions of every term used in these docs, and where each one shows up in this
project. Terms are grouped by topic. Use your browser's find (Ctrl/Cmd + F) to jump to one.

## Corpus and ingestion

| Term | Meaning | In this project |
|---|---|---|
| **10-K** | The annual report every US-listed company files with the SEC | The documents being searched: 20 companies × 3 years |
| **SEC EDGAR** | The SEC's public database of company filings | Where filings are downloaded from (F1) |
| **Accession number** | EDGAR's unique ID for one filing | The identity of a document; used to skip re-downloads |
| **Fiscal year (FY)** | A company's 12-month reporting period, which doesn't always match the calendar year | A filter (`fiscal_year`) on every chunk |
| **Item 1A / Item 7…** | Standard 10-K sections (1A = Risk Factors, 7 = Management's Discussion & Analysis) | A filter (`section_item`) and part of each chunk's header |
| **Parsing** | Turning raw HTML into structured text, headings and tables | Done by Docling (F2) |
| **Docling** | An open-source document parser that keeps structure and tables | The parser (F2) |
| **Canonical text** | One fixed text version of a filing that never changes after parsing | All character offsets point into it, so labels stay valid |
| **Character offset / span** | A start and end position in the canonical text, e.g. chars 184,220–184,910 | How chunks and golden-set evidence are located |
| **Chunk** | A small passage (~512 tokens) cut from a document so it fits into search and prompts | The unit that gets embedded, searched and cited (F3) |
| **Chunking strategy** | The rule for cutting chunks: fixed size, by document structure, tables separately… | `fixed-512`, `structural-512`, parent-child (F3) |
| **Overlap** | Chunks share some text at their edges so a sentence isn't cut in half | ~15% in the fixed strategy |
| **Contextual header** | A short line added to each chunk, like "PepsiCo · FY2024 · Item 7 · Gross margin" | Helps search tell companies and years apart (A5) |
| **Parent-child chunks** | Search small chunks, but give the LLM their larger "parent" section | Ablation A7 |
| **Index version** | One combination of chunking strategy × embedding model, stored side by side | e.g. `v2-structural-512-ctx__emb-large-1024` |
| **Idempotent** | Running the same job twice has the same result as running it once | Ingestion skips unchanged filings (content hash) |

## Search and retrieval

| Term | Meaning | In this project |
|---|---|---|
| **RAG** (retrieval-augmented generation) | Find relevant passages first, then let the LLM answer using only them | The whole system |
| **Token** | The unit LLMs read and write in; roughly ¾ of an English word | Chunk sizes, context budgets and costs are all in tokens |
| **Embedding** | A list of numbers (a vector) that represents the meaning of a text; similar meanings give nearby vectors | 1,024 numbers per chunk (F4) |
| **Embedding model** | The model that turns text into embeddings | `text-embedding-3-large` at 1,024 dims; BGE-M3 locally |
| **Vector / dense search** | Finding chunks whose embeddings are closest to the question's embedding | Good at meaning and paraphrases (F5) |
| **Cosine similarity** | A measure of how close two vectors point in the same direction | The distance used in dense search |
| **pgvector** | A Postgres extension that stores vectors and searches them | The vector store (no separate vector DB) |
| **HNSW** | A fast approximate nearest-neighbour index for vectors | The pgvector index type; `ef_search` trades speed for accuracy |
| **halfvec** | pgvector's 16-bit vector type, half the storage of normal floats | Used for all embeddings (ADR-003) |
| **Sparse / keyword search** | Finding chunks that contain the question's exact words | Postgres full-text search; good at names, tickers, line items |
| **BM25 / ts_rank_cd** | Formulas that score how well a text matches keywords | `ts_rank_cd` now, BM25 via `pg_search` if needed (ADR-004) |
| **Hybrid search** | Running dense and sparse search together and merging the results | The default retrieval (A2 onwards) |
| **RRF** (Reciprocal Rank Fusion) | Merge ranked lists by giving each item `1 / (60 + rank)` from each list and summing | How dense and sparse results are combined (F5) |
| **Reranker / cross-encoder** | A model that reads the question and one passage *together* and scores the match; slower but more accurate than embeddings | Re-orders 40 candidates to the best 8 (F6) |
| **top-k** | Keep the k best results | 50 per search, 40 after fusion, 8 after reranking |
| **Query rewrite / multi-query** | An LLM turns the question into one or more better search queries | F7 |
| **Self-query filters** | An LLM extracts filters (company, year, section) from the question itself | F7, applied as a *soft* filter |
| **Metadata filter** | Restricting search to chunks with certain properties | `ticker`, `fiscal_year`, `section_item` |

## Answer generation

| Term | Meaning | In this project |
|---|---|---|
| **Context window / context** | The text given to the LLM along with the question | The numbered sources, capped at ~6,000 tokens |
| **Grounded answer** | An answer based only on the provided sources | Enforced by the prompt and measured by faithfulness |
| **Citation** | A `[n]` marker linking a sentence to source number n | Validated and mapped to exact character ranges (F8) |
| **Abstention** | Saying "not found in the filings" instead of guessing | Triggered by a low rerank score or the LLM's `NOT_FOUND` |
| **Hallucination** | A confident statement not supported by the sources | What faithfulness and citation checks catch |
| **Lost in the middle** | LLMs pay less attention to text in the middle of a long context | Why source order is configurable |
| **Prompt injection** | Text that tries to override the LLM's instructions, e.g. hidden in a document | Sources are wrapped in `<source>` tags and treated as data |
| **SSE** (Server-Sent Events) | A simple way for a server to stream events to a client over HTTP | How answers stream: `meta`, `token`, `citations`, `done` (+ `step` in agent mode) |
| **TTFT** (time to first token) | How long until the first word of the answer appears | Target p95 ≤ 3 s |

## Agents

| Term | Meaning | In this project |
|---|---|---|
| **Agent** | An LLM in a loop that decides which tools to call, looks at the results, and decides what to do next | The research agent in agent mode (F18) |
| **Agentic RAG** | RAG where an agent decides what to search and when it has enough evidence, instead of one fixed search | `mode: agent` |
| **Tool / tool calling** | A function the LLM can ask to run, with arguments in JSON | `search_filings`, `read_chunk_context`, `list_filings`, `calculate`, `finish` |
| **LangGraph** | A framework for building agents as graphs of steps (nodes) that share a state | Used only for the agent loop (ADR-015) |
| **State graph / node / edge** | The agent's steps (nodes), the paths between them (edges), and the shared data they update (state) | plan → act → guard → tools → observe → reflect → answer |
| **Checkpointer** | Saves the agent's state after each step so a run can be inspected or resumed | LangGraph's Postgres checkpointer in the API |
| **Evidence pool** | Every chunk the agent has found so far, deduplicated | What the final answer is written from |
| **Guard / budget** | Hard limits on steps, tool calls, tokens and time, checked before each tool call | 6 steps, 12 calls, 40k tokens, 45 s (ADR-016) |
| **Router (`mode: auto`)** | Sends each question to the pipeline or the agent based on its type | Agent only for comparison and multi-hop questions |
| **Trajectory** | The sequence of tool calls an agent made during a run | Stored per run and scored (F19) |
| **Trajectory eval** | Scoring *how* the agent reached its answer, not only the answer | Search coverage, step efficiency, redundant calls… |
| **Parallel tool calls** | Asking for several tools in one step, which then run at the same time | Searching two companies at once |

## Evaluation

| Term | Meaning | In this project |
|---|---|---|
| **Golden set / golden dataset** | Questions with known correct answers and evidence, checked by a human | 150 questions (+ 20 holdout), versioned in git (F11) |
| **Evidence span** | The exact passage (character range) in a filing that proves an answer | Ground truth for retrieval scoring (ADR-007) |
| **Split: smoke / full / holdout** | Small quick subset / all questions / questions kept aside until the very end | 30 in CI / 150 nightly / 20 once |
| **Recall@k** | Share of the needed evidence that appears in the top k results | Main retrieval metric (recall@8) |
| **Precision@k** | Share of the top k results that are actually relevant | Measures noise in the context |
| **MRR** | Average of 1 / (position of the first relevant result) | Rewards putting the right chunk first |
| **nDCG** | A ranking score that rewards relevant results more when they're higher up | Secondary retrieval metric |
| **Faithfulness** | Share of the answer's claims supported by the retrieved sources | Main hallucination metric (Ragas) |
| **Answer correctness** | Whether the answer contains the key facts and no contradictions | Judged with a rubric (DeepEval GEval) |
| **Deterministic metric** | A metric computed by code, with no LLM involved, so it's the same every run | Recall, numeric accuracy, citation validity; gated tightly |
| **LLM-as-judge** | Using an LLM with a scoring rubric to grade answers | Correctness and citation support |
| **Rubric** | Written scoring rules with example answers for each score | Stored as versioned judge prompts |
| **Calibration / Cohen's κ (kappa)** | Checking the judge against your own labels; κ measures agreement beyond chance (1 = perfect, 0 = chance) | Judge trusted only if κ ≥ 0.6 |
| **Ragas / DeepEval** | Open-source libraries of LLM evaluation metrics | Faithfulness (Ragas), rubric judges (DeepEval) |
| **Ablation** | Changing one thing at a time to measure its effect | A0 → A7, plus AG1 / AG2 for the agent (F14) |
| **Baseline** | The score to beat, usually the simplest version or the `main` branch | A0 for ablations; `main@latest` for the gate |
| **CI eval gate** | An automated check that fails a pull request if quality drops | GitHub Actions + `gate.yaml` (F13) |
| **Regression** | A change that makes a metric worse | What the gate blocks |
| **Online evaluation** | Scoring real usage after deployment, not just the golden set | Sampled traces + 👍/👎 feedback (F17) |

## Statistics

| Term | Meaning | In this project |
|---|---|---|
| **Confidence interval (CI)** | A range that likely contains the true value; narrow = more certain | Every score is shown as mean ± 95% CI |
| **Bootstrap** | Estimating uncertainty by re-sampling the questions many times and recomputing the score | 1,000 resamples |
| **Paired bootstrap** | The same, but on per-question *differences* between two runs on the same questions | Decides if a change is a real improvement |
| **Statistically significant** | The difference is unlikely to be random noise | The CI of the difference excludes 0 |
| **p50 / p95 / p99** | The value that 50% / 95% / 99% of requests stay under | Latency targets use p95 |

## Observability and delivery

| Term | Meaning | In this project |
|---|---|---|
| **Trace / span** | A trace is one request's full record; spans are its timed steps | One trace per query or eval item |
| **Langfuse** | An open-source platform for LLM traces, prompt versions and eval scores | F10 |
| **OpenTelemetry (OTel)** | A vendor-neutral standard for traces and metrics | How spans reach Langfuse |
| **Config hash** | A fingerprint of a pipeline's YAML config | Stored with every run so results are reproducible |
| **uv** | A fast Python package and project manager | Dependency management |
| **Docker Compose** | Runs several containers together from one file | The local stack |
| **testcontainers** | Starts real databases in Docker during tests | Integration tests against real Postgres |
| **Terraform** | Describes cloud infrastructure as code | The Azure deployment (F16) |
| **Azure Container Apps** | Azure's managed service for running containers, with scale to zero | Where the API, worker and UI run |
| **Scale to zero** | No running instances (and almost no cost) when there's no traffic | Keeps the demo cheap |
| **OIDC login** | CI signs in to Azure with short-lived tokens instead of stored passwords | Used by the deploy workflow |
