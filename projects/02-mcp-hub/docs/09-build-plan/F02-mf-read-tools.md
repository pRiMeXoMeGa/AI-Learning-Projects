# F2: MF Read Tools & Maths

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M1 | Must | F1 | 7 h | F3, F5, F15 |

**Goal:** Six read tools with typed input **and output** schemas, correct financial maths, and
descriptions written for models (what, when, when not, units, example).

## Diagram: tools and the code behind them

```mermaid
flowchart LR
    subgraph Tools["FastMCP tools (mf:read)"]
        T1[search_schemes]
        T2[get_scheme]
        T3[get_nav_history]
        T4[compute_returns]
        T5[compare_schemes]
        T6[sip_backtest]
    end
    subgraph Core["pure maths (no I/O)"]
        CAGR["cagr(start, end, years)"]
        ABS["absolute_return"]
        XIRR["xirr(cashflows)<br/>Newton + bisection fallback"]
        DD["max_drawdown(series)"]
        VOL["annualised volatility"]
        NEAR["nav_on_or_before(date)"]
    end
    subgraph Repo["repositories (SQL)"]
        S1["trigram search"]
        S2["NAV range query<br/>(+ weekly/monthly sampling)"]
    end
    T1 --> S1
    T2 & T3 --> S2
    T4 --> NEAR & CAGR & ABS
    T5 --> CAGR & VOL & DD
    T6 --> NEAR & XIRR
```

## Diagram: SIP back-test

```mermaid
flowchart TB
    IN["scheme_code · monthly_amount ·<br/>start · end · day_of_month"] --> DATES["installment dates<br/>(day_of_month, next NAV day if holiday)"]
    DATES --> BUY["units += amount / NAV(date)"]
    BUY --> VAL["value = units × NAV(end)"]
    VAL --> CF["cash flows: −amount on each date,<br/>+value on end date"]
    CF --> X["xirr(cash flows)"]
    X --> OUT["{invested, value, xirr_pct,<br/>installments, as_of}"]
```

## Deliverables / files
```
servers/india-mf-mcp/src/india_mf/maths/returns.py     # cagr, absolute, drawdown, volatility (pure)
servers/india-mf-mcp/src/india_mf/maths/xirr.py        # pure, robust solver
servers/india-mf-mcp/src/india_mf/repo/schemes.py      # search, details
servers/india-mf-mcp/src/india_mf/repo/nav.py          # ranges, sampling, nav_on_or_before
servers/india-mf-mcp/src/india_mf/tools/read.py        # FastMCP tool definitions
servers/india-mf-mcp/src/india_mf/server.py            # server instructions ("data, not advice")
```

## Tasks
- [ ] Pure maths with thorough tests (holidays, missing NAVs, short histories)
- [ ] Output models (Pydantic) → `outputSchema`; return `structuredContent` plus a short text summary
- [ ] Descriptions following the style guide in [03 §3.2](../03-low-level-design.md); annotations (`readOnlyHint`, etc.)
- [ ] `as_of` and `source: "AMFI"` in every result
- [ ] Result limits: max 20 search results, max 500 history points (with `truncated`)
- [ ] Tool errors as `isError` results with helpful messages ("scheme 123 not found; use search_schemes")

## Acceptance criteria
- Returns and XIRR match a spreadsheet calculation on 5 hand-checked examples
- Every tool validates in MCP Inspector with its output schema
- p95 ≤ 300 ms for read tools on the full dataset

## Tests
- Property (hypothesis): XIRR of a constant-growth series equals the growth rate; CAGR round-trips
- Unit: each tool with a fake repository
- Integration: tools against the real schema with seeded data

**Interview talking point:** *"The maths is pure and property-tested, and every tool returns typed
structured content. Later I measured how much those output schemas actually help models (T5 in the
evals)."*
