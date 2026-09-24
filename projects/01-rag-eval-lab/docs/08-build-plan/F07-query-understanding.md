# F7: Query Understanding

| Milestone | Depends on | Effort | Unblocks |
|---|---|---|---|
| M3 | F0 (providers) | 4 h | F5 multi-query, ablations A4, A6 |

**Goal:** One cheap LLM call turns the user question into **(a) 1–3 retrieval queries** (rewrite and
decomposition) and **(b) structured filters** (company, fiscal year, section), using a JSON-schema
structured output.

## Diagram: query processor

```mermaid
flowchart TB
    Q(["question + explicit filters?"]) --> SHORT{"rewrite.enabled?"}
    SHORT -- no --> PASS["queries = [question]"]
    SHORT -- yes --> LLM["fast model, temperature 0<br/>structured output → QueryPlan"]
    LLM --> VAL{"valid JSON &<br/>tickers in corpus?"}
    VAL -- no --> PASS
    VAL -- yes --> PLAN["QueryPlan"]
    PLAN --> MERGE["merge filters:<br/>explicit (user) filters WIN over extracted"]
    PASS --> MERGE
    MERGE --> OUT["RetrievalRequest → F5"]
    LLM -. "timeout 3 s" .-> PASS
```

## Diagram: QueryPlan schema

```mermaid
classDiagram
    class QueryPlan {
        +list~str~ queries  "1..3"
        +QueryType type
        +Filters filters
    }
    class QueryType {
        <<enumeration>>
        factoid
        comparison_years
        comparison_companies
        multi_hop
        other
    }
    class Filters {
        +list~str~ ticker "nullable, validated vs corpus"
        +list~int~ fiscal_year "nullable"
        +list~str~ section_item "nullable, e.g. 1A, 7"
    }
    QueryPlan --> QueryType
    QueryPlan --> Filters
```

## Example
```
Q: "Compare Coca-Cola's and PepsiCo's effective tax rates in FY2024."
→ queries:  ["Coca-Cola effective tax rate fiscal 2024",
             "PepsiCo effective tax rate fiscal 2024"]
  type:     comparison_companies
  filters:  {ticker: ["KO","PEP"], fiscal_year: [2024], section_item: null}
```

## Deliverables / files
```
src/ragkit/query/plan.py             # QueryPlan pydantic model
src/ragkit/query/processor.py        # LLM call, validation, merge
configs/prompts/query_plan@v1.txt    # prompt with company-name → ticker list injected
```

## Tasks
- [ ] Prompt with few-shot examples per query type; include the list of corpus companies and tickers
- [ ] Structured output via the provider's JSON-schema mode; Pydantic validation
- [ ] Guardrails: max 3 queries; unknown tickers dropped; fiscal years clamped to the corpus range
- [ ] Separate switches: `rewrite.enabled` and `self_query_filters.enabled` (ablations A4 vs A6)
- [ ] Filter-accuracy metric hook for F12 (compare with `expected_filters`)

## Acceptance criteria
- Filter-extraction accuracy ≥ 90% on golden v0 (exact match on ticker + year)
- p95 latency < 700 ms; on timeout the pipeline still answers (fallback)
- Invalid LLM output never raises to the user

## Tests
- Unit: merge precedence; validation drops unknown tickers
- Recorded-response tests (fixtures) so CI doesn't call the LLM

**Interview talking point:** *"Self-query filters are applied softly, because wrong filter extraction is
worse than no filter."*
