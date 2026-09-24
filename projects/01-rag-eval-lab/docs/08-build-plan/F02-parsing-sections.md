# F2: Parsing & Section Detection

| Milestone | Depends on | Effort | Unblocks |
|---|---|---|---|
| M1 | F1 | 5 h | F3, F11 |

**Goal:** Convert raw 10-K HTML into a **canonical text with stable character offsets**, a heading tree,
tables, and **10-K Item sections** (1, 1A, 7, 7A, 8 …). These offsets are the coordinate system for both
chunks and golden-set evidence spans (ADR-007).

## Diagram: parse pipeline

```mermaid
flowchart LR
    RAW["raw HTML<br/>(blob)"] --> CLEAN["Pre-clean<br/>drop XBRL hidden tags,<br/>page headers/footers"]
    CLEAN --> DOCL["Docling convert<br/>→ DoclingDocument"]
    DOCL --> CANON["Build canonical text<br/>+ element offset map"]
    CANON --> SECT["Section detector<br/>regex + heading tree"]
    DOCL --> TABS["Table extractor<br/>cells → Markdown"]
    SECT --> OUT1["sections rows<br/>(item_code, title, char_start, char_end)"]
    TABS --> OUT2["table elements<br/>(offset, caption, markdown)"]
    CANON --> OUT3["parsed JSON → blob<br/>parsed/{ticker}/{accession}.json"]
    OUT1 & OUT2 & OUT3 --> DONE["documents.status = parsed<br/>enqueue chunk jobs (F3)"]
```

## Diagram: canonical text & offset model

```mermaid
classDiagram
    class ParsedDocument {
        +str document_id
        +str canonical_text
        +str parser_version
        +list~Element~ elements
        +list~Section~ sections
    }
    class Element {
        +str kind  "heading|paragraph|table|list"
        +int char_start
        +int char_end
        +list~str~ heading_path
        +str table_markdown
    }
    class Section {
        +str item_code  "1, 1A, 7, 7A, 8 ..."
        +str title
        +int char_start
        +int char_end
    }
    ParsedDocument "1" --> "*" Element
    ParsedDocument "1" --> "*" Section
```

## Diagram: section detection logic

```mermaid
stateDiagram-v2
    [*] --> ScanHeadings
    ScanHeadings --> MatchItem: heading matches ^ITEM\s+(\d+[A-C]?)\.
    MatchItem --> SkipTOC: first occurrence inside table of contents?
    SkipTOC --> ScanHeadings: yes (skip)
    SkipTOC --> OpenSection: no
    OpenSection --> ScanHeadings: close previous section at this offset
    ScanHeadings --> [*]: end of document → close last section
```

## Deliverables / files
```
src/ragkit/ingestion/parse.py        # docling wrapper → ParsedDocument
src/ragkit/ingestion/sections.py     # 10-K item detector (TOC-aware)
src/ragkit/ingestion/tables.py       # table → markdown with caption + header row
src/worker/jobs/parse.py
notebooks/spike_docling.ipynb        # the spike (step 1)
```

## Tasks
- [ ] **Spike first (1 h):** run Docling on 3 filings (PEP, CL, HSY); inspect table quality. Decide go/fallback.
- [ ] Canonical text builder: concatenate elements with `\n\n`; record `(char_start, char_end)` per element
- [ ] Section detector: must skip the table-of-contents duplicates, and handle "Item 7." vs "ITEM 7 —" variants
- [ ] Table serialiser: Markdown with header row, caption and the preceding sentence as context
- [ ] Store `parser_version` (Docling version + our code version) on the document
- [ ] Pin the Docling version in `pyproject.toml`

## Acceptance criteria
- For ≥ 95% of filings, Items 1, 1A, 7, 7A and 8 are detected with plausible lengths (a report lists outliers)
- `canonical_text[el.char_start:el.char_end]` equals the element text for 100% of elements (invariant test)
- Parsing the same file twice gives byte-identical canonical text (determinism)
- 5 manually checked tables per company group render correctly as Markdown

## Tests
- Unit: section regex on 15 heading variants; TOC skipping; offset invariant (hypothesis)
- Golden-file test: 1 small committed 10-K excerpt → expected sections JSON

**Interview talking point:** *"All ground truth is anchored to a deterministic canonical text, so a
chunking change never invalidates the eval set."*
