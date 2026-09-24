# F4: Embedding & Indexing

| Milestone | Depends on | Effort | Unblocks |
|---|---|---|---|
| M1 | F3 | 4 h | F5 |

**Goal:** Embed chunks in batches with caching, write vectors as `halfvec(1024)`, and build the HNSW and
GIN indexes for each index version.

## Diagram: embedding job

```mermaid
flowchart TB
    Q["chunks WHERE index_version = v<br/>AND embedding IS NULL AND kind != 'parent'"] --> B["batch 128 chunks<br/>(≤ provider token limit)"]
    B --> TXT["text = context_header + '\n' + content"]
    TXT --> C{"Redis cache hit?<br/>key = sha256(model‖dim‖'document'‖text)"}
    C -- hit --> V
    C -- miss --> API["Embedder.embed(texts, input_type='document')<br/>retry w/ backoff on 429"]
    API --> PUT["cache.set"] --> V["vectors (1024-d, L2-normalised)"]
    V --> UP["COPY / batched UPDATE<br/>chunks.embedding = halfvec"]
    UP --> MORE{"more batches?"}
    MORE -- yes --> B
    MORE -- no --> IDX["CREATE INDEX IF NOT EXISTS<br/>HNSW partial index for v<br/>ANALYZE chunks"]
    IDX --> READY["index_versions.status = ready"]
```

## Diagram: embedder adapters

```mermaid
classDiagram
    class Embedder {
        <<interface>>
        +model_id: str
        +dim: int
        +embed(texts, input_type) list~vector~
    }
    class OpenAIEmbedder {
        text-embedding-3-large
        dimensions=1024 (Matryoshka)
    }
    class AzureOpenAIEmbedder
    class BGEM3Embedder {
        local sentence-transformers
        1024-d
    }
    class CachedEmbedder {
        decorator: Redis cache
    }
    class TracedEmbedder {
        decorator: span + tokens + cost
    }
    Embedder <|.. OpenAIEmbedder
    Embedder <|.. AzureOpenAIEmbedder
    Embedder <|.. BGEM3Embedder
    CachedEmbedder o-- Embedder
    TracedEmbedder o-- Embedder
```

## Deliverables / files
```
src/ragkit/providers/embeddings.py   # protocol + adapters + decorators
src/ragkit/storage/chunks_repo.py    # bulk write, index management
src/worker/jobs/embed.py
migrations/0002_hnsw_indexes.py      # helper to create partial HNSW per version
```

## Tasks
- [ ] `Embedder` protocol; OpenAI/Azure adapter with `dimensions=1024`; BGE-M3 local adapter
- [ ] Cache + tracing decorators (tracing is a no-op until F10)
- [ ] Batch writer using `psycopg` COPY or `executemany`; `halfvec` conversion
- [ ] Per-version partial HNSW index (`m=16, ef_construction=64`) + GIN on `tsv`
- [ ] Embedding cost report: total tokens per version

## Acceptance criteria
- Second run of the embed job: 100% cache hits, 0 provider calls
- `EXPLAIN` of a dense query uses the HNSW index for that version
- Embedding a full index version (~12k chunks) finishes in < 15 min

## Tests
- Unit: batching respects the token limit; cache key includes the model and dimension
- Integration: testcontainers Postgres, fake embedder → cosine query returns the expected nearest chunk

**Interview talking point:** *"1024-dimensional Matryoshka embeddings stored as halfvec: half the memory,
within pgvector's HNSW limits, and the recall cost is measured in ablation A-emb."*
