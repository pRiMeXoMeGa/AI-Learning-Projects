# 6. Non-Functional Design

## 6.1 Interactive latency budget (one slice review)

| Step | Budget (p95) |
|---|---|
| Supervisor routing (small model) | ≤ 1.5 s |
| Context search (knowledge-mcp, P1 pipeline) | ≤ 2 s |
| Forecast slice + analogs (forecast service, cached run) | ≤ 1 s |
| Forecast Reviewer reasoning + proposal (main model, streamed) | ≤ 25 s |
| Revision engine validate + apply + re-reconcile slice | ≤ 1 s |
| Chart render (typed UI part) | ≤ 0.5 s |
| **Total, one slice** | **≤ 45 s** (first token ≤ 2 s) |

Scenario re-forecasts (≤ 200 series) use Chronos-2 on CPU with a warm model: target ≤ 5 s. If that fails
in the spike, scenarios fall back to LightGBM with changed features, which is milliseconds.

## 6.2 Batch sizing

| Job | Scope | Estimate (to verify in the first spike) |
|---|---|---|
| Nightly demo forecast | FreshRetailNet-50K subset (≈ 2,000 series) or one M5 state locally | ≤ 30 min on 4 vCPU |
| M5 statistical + LightGBM backtests | 30,490 series × 5 origins | A few hours on one machine; offline |
| M5 foundation-model backtests | Chronos-2 on 30,490 series × 5 origins | Rented GPU for a few hours, or a stratified 3,000-series sample on CPU |

The decision (GPU vs sample) is made from measured throughput in week 1 and recorded in an ADR update. The
same sample is used for every model if sampling is chosen, so comparisons stay fair.

## 6.3 Cost of running it

| Item | Estimate |
|---|---|
| Agent service, MCP servers, forecast service (Container Apps, scale to zero) | ≈ $20–40/month while demoing |
| Nightly job (Container Apps Job) | ≈ $2–5/month |
| Postgres + Blob (existing) | ≈ $0–5 extra |
| Vercel (web) | Free/hobby tier |
| GPU for foundation-model backtests (optional, one-off) | ≈ $10–30 |
| Evaluation model spend (PlanBench k=3 × 2 designs, RAG, red team) | ≈ $40–80, via P7 budgets |
| **Per planning session** | Target median ≤ $0.40 (small model for routing, main model for reviewing) |

## 6.4 Observability

- **Traces:** one Langfuse trace per session with P3's span names: supervisor, specialist, tool calls,
  revision-engine checks, approvals, cost from P7 headers.
- **Metrics:** sessions, revisions proposed/accepted/rejected (by rule), approvals and edit rate, orders,
  FVA over time (AI vs human), nightly job duration, forecast service latency.
- **Dashboards:** "Planning health" (FVA trend, harmful rate, approval edit rate) and "Cost" (per session,
  per org), next to P7's gateway dashboards.

## 6.5 Failure modes

| Failure | Behaviour |
|---|---|
| Nightly job fails | Yesterday's run stays current, flagged with its age; alert |
| Forecast service down | Read-only mode: last run, no scenarios, no new revisions |
| Model hub unreachable | No effect at run time (weights baked into the image) |
| LLM provider down | P7 fallbacks; if all fail, the workspace still shows forecasts and exceptions without agents |
| Agent crash mid-session | Resume from checkpoint (P3); approvals and orders are idempotent |
| Supplier agent down | PO stays submitted; retries; inbox shows the status |
| Sandbox down | Analyst works from tool data only |

## 6.6 Testing strategy

| Layer | Tests |
|---|---|
| Revision engine | Property tests: bounds, windows, quantile monotonicity after scaling, hierarchy sums after reconciliation |
| Order maths | Property tests: non-negative, case-pack rounding, service level monotonic in α |
| Forecast service | Backtest smoke; model-card checks; reproducible runs from seeds and data hashes |
| MCP tools | Contract tests; Cedar policy tests per role; RLS tests |
| Agents | PlanBench smoke; trajectory assertions; chaos resume (P3) |
| Web | Playwright: approval flow, diff rendering, no external requests from charts (P5 pattern) |
| Security | Red-team suite R1–R8 |
