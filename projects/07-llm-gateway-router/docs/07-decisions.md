# 7. Architecture Decision Records

Format: **Context → Decision → Alternatives → Consequences.** Status is *Proposed* until validated while
building.

---

### ADR-001: Build a small gateway, and benchmark against the ones you'd buy
- **Context:** Mature gateways exist (LiteLLM, Portkey, Kong, Vercel/Cloudflare AI Gateway). The learning
  goal is to understand and measure each cost feature, and interviewers ask "why not just use X?"
- **Decision:** Build **Switchboard** with the features being measured. Compare it with the LiteLLM proxy
  and one managed gateway on overhead and features ([04 §4.6](04-evaluation-design.md#46-overhead-and-load)).
- **Consequences:**
  - Deep understanding and a strong build-vs-buy answer.
  - The README states plainly when you'd buy instead (most teams should).

### ADR-002: Official provider SDKs, no LiteLLM at run time
- **Context:** The gateway sees every key and prompt. LiteLLM's PyPI package was compromised in March 2026.
  More generally, a large translation library is a big supply-chain surface.
- **Decision:**
  - The Anthropic and OpenAI official async SDKs behind small in-house adapters, with contract tests.
  - LiteLLM runs only as a pinned, isolated benchmark target.
- **Alternatives:** LiteLLM as a library (100+ providers, much less code; a larger dependency surface).
- **Consequences:**
  - Only two providers are supported (the ones the projects use). Adding one means an adapter and its contract tests.
  - A smaller, auditable dependency tree.

### ADR-003: OpenAI-compatible API as the front door
- **Decision:** `/v1/chat/completions`, the de facto gateway standard. The AI SDK's
  `openai-compatible` provider, the OpenAI SDKs and LangChain can all point at it. An Anthropic
  pass-through is a Should.
- **Consequences:**
  - One-line integration for Projects 1, 5 and 6.
  - Anthropic-only features (cache controls, some structured-output modes) are mapped or passed through.

### ADR-004: Three cache layers, with the semantic cache opt-in and scoped
- **Decision:**
  - Prompt caching is always helped: stable prefixes and breakpoints.
  - The exact cache is on for deterministic requests.
  - The semantic cache is **off by default** and opt-in per app. It is scoped by tenant, model,
    system-prompt hash and tools hash, has per-app thresholds from the study, uses verify-on-hit for
    high-risk apps, and never caches requests with untrusted context unless the app opts in.
- **Alternatives:** A global semantic cache with one threshold (the common tutorial setup; unsafe, and the
  quality loss is unmeasured).
- **Consequences:** Lower hit rates than naive setups, but measured false-hit rates and poisoning
  resistance. Provider prompt caching usually saves the most for the least risk, and the report will
  likely show that.

### ADR-005: Five routing policies compared on Pareto curves
- **Decision:** `fixed`, `rules`, `routellm-mf`, our `classifier` and `cascade`, all evaluated on the same
  workloads with an oracle upper bound. The shipped policy and threshold per app come from the curves.
- **Alternatives:** A single "smart router" product (NotDiamond, Martian); only rules.
- **Consequences:** An evidence-based routing choice. Training data comes from earlier projects' eval sets,
  which is a nice reuse.

### ADR-006: Router model as a small ONNX classifier in-process
- **Decision:** Logistic regression or a small MLP on embedding + simple features, exported to ONNX and
  loaded in the gateway (≤ 2 ms).
- **Alternatives:** An LLM-as-router (slow, costly); a separate router service (a network hop).
- **Consequences:** Negligible latency. Retraining is offline and versioned with the policy.

### ADR-007: Retries and fallbacks only before the first token
- **Decision:** Retry/fallback happens only before any token has been streamed to the caller. After that,
  errors pass through.
- **Consequences:** No duplicated or spliced answers. Callers handle mid-stream errors, which are rare
  and visible.

### ADR-008: Budgets use reserve-then-reconcile, fail closed
- **Decision:**
  - Reserve the estimated cost atomically in Redis before the call, then settle with the actual usage.
  - If the budget store is unavailable, deny by default (configurable per app).
- **Consequences:** Parallel requests can't overspend. There's a small availability cost when Redis is down.

### ADR-009: Python (FastAPI) for the gateway
- **Context:** Go or Rust gateways are faster. Python matches the rest of the portfolio and the ML routing
  code.
- **Decision:** Async Python (FastAPI + uvicorn + official async SDKs). Overhead is measured, and the target
  is ≤ 15 ms p95.
- **Alternatives:** Go (Kong, custom), Rust (TensorZero-style), TypeScript on the edge.
- **Consequences:** The overhead numbers will show whether Python is enough. The report discusses when you'd
  switch.

### ADR-010: Prometheus + Grafana for cost/performance dashboards, Langfuse for traces
- **Decision:** Metrics in Prometheus (for alerting and dashboards everyone knows), traces in Langfuse (as
  in the other projects).
- **Consequences:** A FinOps-style dashboard that interviewers recognize, and continuity with earlier traces.

### ADR-011: Evaluate on replayed traces from earlier projects
- **Decision:** Record request traces from Projects 1, 3 and 6 (public data only), replay them with a
  realistic repetition model, and score quality with each project's own graders.
- **Alternatives:** Synthetic prompts only (unrealistic repeat rates and difficulty mix).
- **Consequences:** Credible savings numbers for workloads you built and can explain.
