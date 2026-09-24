# F5: Hybrid Retrieval

| Milestone | Depends on | Effort | Unblocks |
|---|---|---|---|
| Phase A (M1): dense only · Phase B (M3): sparse, RRF, filters, multi-query | F4 (F7 for multi-query) | 5 h | F6, F8 |

**Goal:** Given one or more queries and optional filters, return a fused and deduplicated candidate list
from **dense (pgvector)** and **sparse (Postgres FTS)** retrieval, with every intermediate score kept for
debugging and evaluation.

## Diagram: retrieval flow

```mermaid
flowchart TB
    IN["RetrievalRequest<br/>queries[1..3], filters, index_version, cfg"] --> EMB["embed queries<br/>(input_type='query', cached)"]
    IN --> FB{"filters present?"}
    FB --> WH["build WHERE clause<br/>ticker IN · fiscal_year IN · section_item IN"]

    subgraph PerQuery["for each query (asyncio.gather)"]
        direction LR
        D["Dense SQL<br/>ORDER BY embedding <=> $q<br/>LIMIT 50<br/>SET hnsw.ef_search=100<br/>SET hnsw.iterative_scan=relaxed_order"]
        S["Sparse SQL<br/>WHERE tsv @@ websearch_to_tsquery($q)<br/>ORDER BY ts_rank_cd DESC LIMIT 50"]
    end
    EMB --> D
    WH --> D & S
    IN --> S
    D & S --> RRF["Reciprocal Rank Fusion<br/>score = Σ 1/(60 + rank)"]
    RRF --> SOFT{"results < 5 and<br/>section filter used?"}
    SOFT -- yes --> RETRY["retry without section filter<br/>(soft filter)"] --> RRF
    SOFT -- no --> DEDUP["dedupe overlapping char ranges<br/>keep top-40"]
    DEDUP --> OUT["Candidates[]<br/>{chunk, dense_rank, sparse_rank, rrf_score}"]
```

## Diagram: RRF worked example

```mermaid
flowchart LR
    subgraph Dense
        d1["1. chunk A"]
        d2["2. chunk B"]
        d3["3. chunk C"]
    end
    subgraph Sparse
        s1["1. chunk C"]
        s2["2. chunk D"]
        s3["3. chunk A"]
    end
    Dense & Sparse --> F["A: 1/61 + 1/63 = 0.0323<br/>C: 1/63 + 1/61 = 0.0323<br/>B: 1/62 = 0.0161<br/>D: 1/62 = 0.0161"]
    F --> R["Fused: A, C, B, D<br/>(ties broken by dense rank)"]
```

## Deliverables / files
```
src/ragkit/retrieval/dense.py
src/ragkit/retrieval/sparse.py
src/ragkit/retrieval/fusion.py       # rrf(), pure function
src/ragkit/retrieval/filters.py      # Filters model → SQL fragments (parameterised)
src/ragkit/retrieval/retriever.py    # HybridRetriever orchestrates per cfg
```

## Tasks
- Phase A
  - [ ] Dense retrieval with a parameterised SQL filter, `ef_search` from config
  - [ ] `Candidate` model with every score field (null when a stage is disabled)
- Phase B
  - [ ] Sparse retrieval with `websearch_to_tsquery('english', q)`
  - [ ] `rrf()` pure function, multi-list, deterministic tie-break
  - [ ] Soft section filter fallback; overlap dedupe
  - [ ] Multi-query: fuse all per-query lists in one RRF
  - [ ] Config switches `dense.enabled`, `sparse.enabled`, `fusion.top_n`

## Acceptance criteria
- Retrieval p95 (dense + sparse, 1 query) < 150 ms on the full corpus, local Docker
- A filtered dense query returns `k` rows even with restrictive filters (iterative scan works)
- Disabling a stage in YAML changes results without code changes
- **All SQL is parameterised** (no string interpolation of user input)

## Tests
- Unit: `rrf()` against the worked example; tie-break; empty lists
- Integration: seeded mini-corpus. An exact-term query (e.g. a line-item name) is found by sparse and not by dense, proving the value of hybrid

**Interview talking point:** *"RRF uses ranks, not scores, so cosine similarity and ts_rank never need
normalising."*
