# F6: Reranking

| Milestone | Depends on | Effort | Unblocks |
|---|---|---|---|
| M3 | F5 | 2 h | F8, ablation A3 |

**Goal:** Re-order the fused candidates with a **cross-encoder** and keep the top-N above a minimum score,
with graceful fallback to the RRF order.

## Diagram: rerank stage

```mermaid
flowchart LR
    C["40 candidates<br/>(RRF order)"] --> TXT["pair: (original question,<br/>context_header + content)"]
    TXT --> RR{"provider"}
    RR -->|cohere| CO["Cohere Rerank API"]
    RR -->|bge-local| BG["BGE-reranker cross-encoder<br/>(sentence-transformers, batch 16)"]
    RR -->|none| PASS["keep RRF order"]
    CO & BG --> SC["relevance scores 0..1"]
    SC --> TH["drop score < min_score (0.15)"]
    TH --> TOP["top_n = 8<br/>+ top_score → abstain signal for F8"]
    CO -. "timeout 2 s / 5xx" .-> FB["fallback: RRF top-8<br/>trace flag degraded=rerank"]
```

## Diagram: why rerank against the original question (multi-query case)

```mermaid
sequenceDiagram
    participant Q as Original question
    participant R1 as Rewrite 1 (FY2023 margin)
    participant R2 as Rewrite 2 (FY2024 margin)
    participant F as RRF union
    participant X as Reranker
    Q->>R1: decompose
    Q->>R2: decompose
    R1->>F: 50 + 50 candidates
    R2->>F: 50 + 50 candidates
    F->>X: top-40 union
    Q->>X: score each against the ORIGINAL question
    Note over X: Keeps chunks relevant to the<br/>whole information need, not one sub-query
```

## Deliverables / files
```
src/ragkit/providers/rerankers.py    # Reranker protocol, CohereReranker, BGEReranker, NoopReranker
src/ragkit/retrieval/rerank.py       # stage logic: threshold, top_n, fallback
```

## Tasks
- [ ] `Reranker` protocol + 3 adapters; Redis cache for eval runs
- [ ] Threshold and top-N from config; expose `top_score` for abstention
- [ ] Timeout + fallback path, trace flag
- [ ] Micro-benchmark: Cohere vs BGE (CPU) latency for 40 docs

## Acceptance criteria
- Rerank p95 < 400 ms (Cohere) for 40 candidates
- A forced provider failure still returns an answer (fallback), flagged in the trace
- Ablation A3 row produced (in F14)

## Tests
- Unit: threshold and top-N logic; fallback on a simulated timeout
- Contract: Cohere adapter parses a recorded response fixture

**Interview talking point:** *"The cross-encoder sees the query and document together, which a bi-encoder
can't. It's usually the cheapest big gain in precision."*
