# F3: Chunking & Index Versions

| Milestone | Depends on | Effort | Unblocks |
|---|---|---|---|
| Phase A (M1): `fixed-512` · Phase B (M3): structural, tables, contextual headers, parent-child | F2 | 5 h | F4, F14 |

**Goal:** Pluggable chunking strategies that turn a `ParsedDocument` into chunks with **exact character
offsets**, stored under an **index version** so several strategies can be compared side by side.

## Diagram: strategy plug-in architecture

```mermaid
classDiagram
    class Chunker {
        <<interface>>
        +strategy_id: str
        +chunk(doc: ParsedDocument, params) list~Chunk~
    }
    class FixedTokenChunker {
        size=512, overlap=64
    }
    class StructuralChunker {
        target=512, max=768
        never crosses section
    }
    class TableChunker {
        1 table = 1 chunk
        split by row groups
    }
    class ParentChildChunker {
        child=256, parent≤2048
    }
    class ContextHeaderDecorator {
        prepends company·FY·item·heading_path
    }
    Chunker <|.. FixedTokenChunker
    Chunker <|.. StructuralChunker
    Chunker <|.. TableChunker
    Chunker <|.. ParentChildChunker
    ContextHeaderDecorator o-- Chunker : wraps
    class Chunk {
        +uuid id
        +str kind
        +int char_start
        +int char_end
        +str section_item
        +str heading_path
        +str context_header
        +str content
        +int token_count
        +uuid parent_id
    }
    Chunker ..> Chunk : produces
```

## Diagram: structural chunking algorithm

```mermaid
flowchart TB
    S["for each section"] --> E["walk elements in order"]
    E --> T{"element is table?"}
    T -- yes --> TC["flush buffer<br/>→ TableChunker emits table chunk(s)"]
    TC --> E
    T -- no --> FIT{"buffer tokens + element<br/>≤ target (512)?"}
    FIT -- yes --> ADD["append element to buffer"] --> E
    FIT -- no --> BIG{"element alone > max (768)?"}
    BIG -- yes --> SPLIT["flush buffer; split element<br/>on sentence boundaries"] --> E
    BIG -- no --> FLUSH["flush buffer as chunk<br/>carry last paragraph as overlap"] --> ADD
    E -->|"section end"| END["flush remaining buffer"]
```

## Diagram: index versions (side-by-side ablations)

```mermaid
flowchart LR
    P[("parsed docs<br/>(parse once)")] --> V1["v1: fixed-512<br/>× emb-large-1024"]
    P --> V2["v2: structural-512 + tables<br/>× emb-large-1024"]
    P --> V3["v3: structural-512 + ctx<br/>× emb-large-1024"]
    P --> V4["v4: parent-child + ctx<br/>× emb-large-1024"]
    P --> V5["v5: structural-512 + ctx<br/>× bge-m3"]
    V1 & V2 & V3 & V4 & V5 --> CH[("chunks table<br/>partitioned by index_version")]
```

## Deliverables / files
```
src/ragkit/chunking/base.py          # Chunker protocol, Chunk model, registry
src/ragkit/chunking/fixed.py
src/ragkit/chunking/structural.py
src/ragkit/chunking/tables.py
src/ragkit/chunking/parent_child.py
src/ragkit/chunking/context_header.py
src/worker/jobs/chunk.py             # (document_id, index_version) → chunks
configs/index_versions.yaml
```

## Tasks
- Phase A
  - [ ] Chunker protocol, registry, `FixedTokenChunker` (tiktoken)
  - [ ] `index_versions` row creation from YAML; chunk ID = hash(index_version, doc, start, end)
- Phase B
  - [ ] `StructuralChunker` (algorithm above), `TableChunker`, `ContextHeaderDecorator`
  - [ ] `ParentChildChunker` (parents stored with `kind='parent'`, not embedded)
  - [ ] Chunk-statistics report per version: count, token histogram, % tables

## Acceptance criteria
- **Offset invariant:** `canonical_text[c.char_start:c.char_end] == c.content` (without the header) for every chunk
- **Coverage:** every non-whitespace character of every section is in ≥ 1 chunk (excluding the TOC)
- No chunk crosses a section boundary (structural strategies)
- Re-chunking the same version is idempotent (same chunk IDs)

## Tests
- Property tests (hypothesis): coverage, offset invariant, max-token bound
- Unit: table split repeats the header row; header template renders correctly

**Interview talking point:** *"Chunking is a measured variable: five index versions live side by side and
the ablation table shows the effect of each, by question type."*
