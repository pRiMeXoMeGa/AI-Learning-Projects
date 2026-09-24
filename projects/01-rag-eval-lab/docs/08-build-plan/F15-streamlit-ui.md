# F15: Streamlit UI

| Milestone | Depends on | Effort | Unblocks |
|---|---|---|---|
| M5 | F9, F12 | 4 h | Demo video, screenshots |

**Goal:** An internal tool with three pages: **Ask** (streamed answer, citations, "why this answer"),
**Eval runs** (browse and compare), **Golden set** (read-only browser).

## Diagram: page map

```mermaid
flowchart TB
    APP["Streamlit app"] --> ASK["Ask"]
    APP --> RUNS["Eval runs"]
    APP --> GOLD["Golden set"]

    ASK --> A1["question box + filters + pipeline selector"]
    ASK --> A2["streamed answer with [n] links"]
    ASK --> A3["citations panel → passage with highlighted span"]
    ASK --> A4["'why this answer?' table:<br/>dense rank · sparse rank · RRF · rerank score"]
    ASK --> A5["👍/👎 feedback → F17"]

    RUNS --> R1["runs list (pipeline, golden version, key metrics)"]
    RUNS --> R2["compare two runs:<br/>metric Δ with CI · regressed questions"]
    RUNS --> R3["item drill-down → Langfuse trace link"]

    GOLD --> G1["filter by type / split / company"]
```

## Diagram: data access

```mermaid
sequenceDiagram
    participant UI as Streamlit
    participant API as FastAPI
    participant DB as Postgres
    UI->>API: POST /v1/query (SSE, debug=true)
    API-->>UI: meta · retrieval · token… · citations · done
    UI->>API: GET /v1/eval/runs, /compare/{a}/{b}
    API->>DB: read eval_runs / eval_results
    API-->>UI: JSON
    UI->>API: POST /v1/feedback
```

## Deliverables / files
```
ui/app.py, ui/pages/1_Ask.py, 2_Eval_Runs.py, 3_Golden_Set.py
ui/client.py                         # SSE client (httpx-sse)
```

## Tasks
- [ ] SSE client; incremental rendering with `st.write_stream`
- [ ] Citation panel: fetch the passage by chunk ID; highlight the cited span
- [ ] "Why this answer" table from the `retrieval` debug event
- [ ] Runs list + comparison view + trace links
- [ ] Golden-set browser (read-only)

## Acceptance criteria
- The full demo flow works end to end locally and on the deployed URL
- The UI talks only to the API (no direct DB access)

## Tests
- Smoke test: the app imports and renders with a mocked API (Streamlit AppTest)

**Interview talking point:** *"The 'why this answer' panel exposes every ranking signal, which makes
debugging retrieval a UI task instead of a notebook task."*
