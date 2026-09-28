# 6. Non-Functional Design

## 6.1 Latency budget (gateway overhead, cache miss)

| Step | Budget (p95) |
|---|---|
| Parse + key lookup (Redis cache of key metadata) | ≤ 2 ms |
| Rate limit + budget reserve (one Redis Lua script) | ≤ 2 ms |
| Exact cache lookup | ≤ 2 ms |
| Semantic cache (only if eligible): embedding + pgvector top-1 | ≤ 35 ms |
| Router (ONNX classifier, in-process) | ≤ 2 ms |
| Request translation | ≤ 1 ms |
| Stream relay per chunk | ≤ 0.2 ms |
| **Total without semantic cache** | **≤ ~10 ms** (target 15 ms) |

The semantic cache is the expensive step, which is why it's opt-in per app. The embedding call can use a
small local model (ONNX) instead of an API to keep it ≤ 10 ms, and experiment §4.3 compares both.

## 6.2 Throughput and scaling

- Python with async I/O (uvicorn, httpx/official async SDKs). Streaming relays don't block the event loop.
- Target: 500 req/s per instance with a mock provider. Scale out with stateless replicas (Redis and
  Postgres hold the state).
- Ledger writes are batched (≤ 200 ms or 500 rows) off the request path. If the ledger queue is full, the
  gateway **keeps serving** but spills to a local file and alerts. Unlike Project 2's audit log, a missing
  ledger row is recoverable from provider usage exports, so availability wins here.

## 6.3 Cost of running it

| Item | Estimate |
|---|---|
| Container Apps (2 small replicas) | ≈ $15–30/month, or scale to zero for the demo |
| Redis (small tier) | ≈ $15/month, or Upstash free tier |
| Postgres (reuse the existing flexible server) | $0 extra |
| Evaluation runs (replay W1–W3 across C0–C6, router sweeps) | ≈ $30–60 in model spend (cheap tiers where possible) |

## 6.4 Observability

**Metrics** (Prometheus, OTel GenAI naming where it exists):
- `gen_ai.client.token.usage` (by model, app, type: input/cached/output)
- `sb_cost_usd_total` (by app, team, model)
- `sb_cache_requests_total` (by layer: exact/semantic, result: hit/miss)
- `sb_route_decisions_total` (by policy, decision)
- `sb_fallbacks_total`, `sb_breaker_state` (by provider/model)
- latency histograms: `gen_ai.server.time_to_first_token`, total duration, gateway overhead

**Dashboards (Grafana):**
- **Spend:** cost per request, per app/team, daily burn vs budget.
- **Performance:** TTFT, p50/p95, overhead.
- **Cache:** hit rates per layer, saved $, false-hit alarms (from verify-on-hit "no" rates).
- **Routing:** share per tier, escalation rate, quality monitors.
- **Reliability:** breaker states, fallback rate, provider error codes.

**Traces:** one span per request with the cache and route decisions as attributes, sent to Langfuse.

## 6.5 Failure modes

| Failure | Behaviour |
|---|---|
| Redis down | Exact cache off (miss); rate limits and budgets **fail closed** (configurable per app); breaker state per instance only |
| Postgres down | Semantic cache off; ledger spills to disk; key metadata served from cache for ≤ 5 min |
| Embedding model down | Semantic cache off |
| Router model fails to load | Fall back to the `rules` policy |
| One provider down | Breakers + fallbacks |
| All providers down | `503` quickly (breakers open) instead of hanging |

## 6.6 Testing strategy

| Layer | Tests |
|---|---|
| Translation | Contract tests per provider with recorded responses (VCR-style cassettes) |
| Cost | Property tests on cost maths; reconciliation against provider usage |
| Caches | Key canonicalization properties; isolation tests; poisoning tests |
| Router | Offline Pareto evaluation; inference latency test; fallback when the model is missing |
| Reliability | Fault-proxy tests (Toxiproxy, as in Project 4) |
| Load | k6 with a mock provider |
| Supply chain | Hash-locked installs, SHA-pinned actions, `.pth` check, image scan |
