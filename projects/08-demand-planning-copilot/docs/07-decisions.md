# 7. Architecture Decision Records

Format: **Context → Decision → Alternatives → Consequences.** Status is *Proposed* until validated while
building.

---

### ADR-001: Models produce forecast numbers; agents propose typed revision actions
- **Context:** LLMs can produce plausible numbers, but they aren't calibrated over thousands of series and
  their changes are hard to audit. 2026 "last-mile forecasting" work puts LLM agents on top of a
  forecasting backbone with explicit, constrained revision actions and a trace.
- **Decision:** The forecast service is the only source of numbers. Agents emit `scale`, `shift`,
  `override`, `cap/floor` or `no_change` actions with evidence; code validates, applies and reconciles.
- **Alternatives:** LLM-as-forecaster (numbers in text); LLM picks a model only (no context use).
- **Consequences:** Every change is bounded, attributable and scorable with FVA. It also turns prompt
  injection into a bounded-authorization problem ([05 §5.4](05-security-threat-model.md#54-why-the-llm-never-writes-numbers-is-also-a-security-control)).

### ADR-002: Trusted forecast service; LLM-written code only in the sandbox
- **Decision:** Our own tested forecasting code runs as a normal service. The Analyst's ad-hoc code runs
  in the P5 sandbox via MCP.
- **Alternatives:** Run all forecasting in the sandbox (P5's original capstone hook idea).
- **Consequences:** Fast, reproducible nightly runs; the sandbox is used where it matters (untrusted code).
  P5's `run_python` still gets real use from the Analyst.

### ADR-003: Model portfolio: baselines + LightGBM + Chronos-2, ensembled and reconciled
- **Context:** Strong M5 results came from gradient-boosted global models. In 2025–26 benchmarks
  (fev-bench, GIFT-Eval), Chronos-2 leads pretrained models and supports covariates through group
  attention.
- **Decision:** `snaive`, `ets/theta`, `croston_tsb`, `lgbm`, `chronos2`, and a skill-weighted `ensemble`;
  TimesFM 2.5 as a comparison (Should). **TiRex is excluded** because its licence (NXAI Community License)
  isn't a standard open-source licence; revisit if that changes.
- **Consequences:** An honest comparison of classical, ML and foundation models on a famous dataset; the
  report can say where each wins (e.g. intermittent series).

### ADR-004: M5 for evaluation, FreshRetailNet-50K for the public demo
- **Context:** M5 is the best-known retail benchmark, but it's distributed under Kaggle competition terms
  that limit redistribution. FreshRetailNet-50K (2025) is CC BY 4.0, has promotions, weather and
  **stockout labels**.
- **Decision:** A canonical schema with two loaders. M5 is downloaded by each user with their own Kaggle
  account for evaluation; the repo stores provenance only. The public demo uses a FreshRetailNet-50K
  subset with attribution.
- **Consequences:** Headline accuracy numbers are M5 (comparable to the literature); the demo is legally
  clean. The FreshRetailNet stockout labels also give a censored-demand sensitivity check.

### ADR-005: Hierarchical reconciliation (MinT) on by default
- **Decision:** Reconcile across item → dept → category → store → state → total.
- **Consequences:** Numbers add up across levels (planners notice when they don't); revisions re-reconcile
  the affected branch. Accuracy with and without reconciliation is reported.

### ADR-006: Supervisor + specialists, compared against a single agent
- **Context:** Multi-agent designs cost more tokens and add coordination failures; P3 measured exactly
  this kind of trade-off.
- **Decision:** A LangGraph supervisor with three specialists, **and** a single-agent baseline with the same
  tools, both evaluated on PlanBench-60.
- **Consequences:** The design choice is evidence-based. If the single agent wins, the report says so and
  the product ships it.

### ADR-007: All tools via MCP behind the P2 gateway, with Cedar policies by role
- **Decision:** sales, forecast, inventory and memory servers plus P1's knowledge tool and P5's sandbox,
  reached only through the gateway. The org claim comes from the token.
- **Consequences:** Role-based tool access (a viewer can't propose orders) and one audit point, reused
  from P2.

### ADR-008: Approvals as signed tokens checked in the tool; idempotent orders
- **Decision:** P3's JWS approval tokens, bound to the proposal and its lines hash, verified inside
  `submit_order` and `accept_revision`. Idempotency keys on every order.
- **Consequences:** An agent can't bypass approval by prompt; crash-resume can't double-order.

### ADR-009: Supplier as an A2A agent in a separate org (Should)
- **Decision:** A small supplier agent (P4 pattern: signed Agent Card, token exchange) confirms, part-fills
  or delays POs.
- **Consequences:** A realistic cross-organization flow and a reuse of P4; the core loop works without it.

### ADR-010: Evaluate with FVA and a replenishment simulation, not accuracy alone
- **Decision:** PlanBench-60 for FVA (including controls, stale and injected documents) and a simulator
  for fill rate and cost.
- **Consequences:** Answers the questions a planning lead asks. The simulator's limits (censored sales,
  assumed lead times) are stated in the report.

### ADR-011: The web workspace reuses P6's shell and P5's safe charts
- **Decision:** Next.js 16 + AI SDK 7 typed UI parts; forecast charts as validated Vega-Lite specs (P5);
  Better Auth orgs and RLS (P6).
- **Consequences:** Little new UI plumbing; the effort goes into the diff, approval and FVA views.

### ADR-012: All model calls through Switchboard (P7); semantic cache off
- **Decision:** One virtual key per org and environment. Prompt caching and exact caching on; the
  semantic cache **off**, because planning answers depend on data that changes daily.
- **Consequences:** Cost per session comes straight from the ledger; budgets cap evaluation spend.

### ADR-013: Memory only from planner-confirmed facts
- **Decision:** Implement P3's (deferred) memory service with a strict write policy: no automatic writes
  from documents or tool output; provenance and scope on every item.
- **Consequences:** Slower memory growth, but closes memory poisoning. The UI lets planners see and delete
  what's remembered.
