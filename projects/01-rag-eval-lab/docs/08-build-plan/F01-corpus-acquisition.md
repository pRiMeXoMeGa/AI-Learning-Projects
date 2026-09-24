# F1: Corpus Acquisition (SEC EDGAR)

| Milestone | Depends on | Effort | Unblocks |
|---|---|---|---|
| M1 (2 companies) → M3 (20 companies) | F0 | 3 h | F2 |

**Goal:** Download the primary 10-K HTML document for each (company, fiscal year) **idempotently**, in
line with SEC fair-access rules, and record filing metadata.

## Diagram: fetch flow

```mermaid
flowchart TB
    START(["rag-lab ingest fetch --tickers PEP,KO --years 2022-2024"]) --> MAP["Resolve ticker → CIK<br/>(company_tickers.json, cached)"]
    MAP --> SUB["GET data.sec.gov/submissions/CIK##########.json"]
    SUB --> FILT["Filter form == '10-K'<br/>match fiscal year (reportDate)"]
    FILT --> LOOP{{"for each filing"}}
    LOOP --> EXIST{"accession_no already<br/>in documents?"}
    EXIST -- "yes" --> HEAD["Download HTML, compute sha256"]
    HEAD --> SAME{"hash unchanged?"}
    SAME -- "yes" --> SKIP["skip (idempotent)"]
    SAME -- "no" --> SAVE
    EXIST -- "no" --> DL["Download primaryDocument HTML"]
    DL --> SAVE["Store raw → blob<br/>raw/{ticker}/{accession}.html"]
    SAVE --> ROW["Upsert documents row<br/>status = fetched"]
    ROW --> ENQ["Enqueue parse job (F2)"]
```

## Diagram: rate limiting & politeness

```mermaid
sequenceDiagram
    participant F as Fetcher
    participant TB as Token bucket (8 req/s)
    participant SEC as sec.gov
    F->>TB: acquire()
    TB-->>F: ok
    F->>SEC: GET (User-Agent: "RAG-Eval-Lab your-email@example.com")
    alt 200
        SEC-->>F: HTML
    else 429 / 503
        SEC-->>F: error
        F->>F: exponential backoff (1s, 2s, 4s, max 5 tries)
    end
```

## Deliverables / files
```
src/ragkit/ingestion/edgar.py        # EdgarClient: cik lookup, submissions, download
src/ragkit/ingestion/ratelimit.py    # async token bucket
src/ragkit/storage/blob.py           # BlobStore protocol: LocalBlobStore, AzureBlobStore (later)
src/worker/jobs/fetch.py             # arq job
configs/corpus.yaml                  # the 20 tickers + fiscal years
```

## Tasks
- [ ] `configs/corpus.yaml` with 20 tickers; **check each has 3 consecutive 10-Ks** (see requirements)
- [ ] `EdgarClient` with httpx AsyncClient, `User-Agent` from settings, token bucket ≤ 8 req/s
- [ ] Map fiscal year using `reportDate` (fiscal years don't equal calendar years for many CPG companies)
- [ ] Only allow `*.sec.gov` hosts (SSRF guard)
- [ ] Blob layout `raw/{ticker}/{accession}.html`; sha256 stored on `documents`
- [ ] CLI `rag-lab ingest fetch`; arq job wrapper
- [ ] Phase A: PEP + KO, 3 years. Phase B: all 20.

## Acceptance criteria
- Running fetch twice makes **0 downloads and 0 DB writes** on the second run
- 60 `documents` rows with `status=fetched` (Phase B); any missing (ticker, year) combination is listed in a report
- No request exceeds the rate limit (checked with a log counter)

## Tests
- Unit: fiscal-year mapping on fixture JSON; allow-list rejects non-SEC hosts
- Integration: `respx` mocked EDGAR → idempotency test

**Interview talking point:** *"Ingestion is idempotent on accession number plus content hash, so re-runs
are free and safe."*
