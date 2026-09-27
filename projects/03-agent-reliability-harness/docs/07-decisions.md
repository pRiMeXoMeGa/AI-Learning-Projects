# 7. Architecture Decision Records

Format: **Context → Decision → Alternatives → Consequences.** Status is *Proposed* until validated while
building.

---

### ADR-001: Incident triage in a simulated environment
- **Context:** The agent must take real actions with graded risk, read untrusted text, and have outcomes
  that can be checked automatically. Real infrastructure is slow, costly and unsafe for 1,000+ runs.
- **Decision:** An IT incident-triage agent working on **OpsSim**, a simulated e-commerce production
  system with a causal fault model.
- **Alternatives:**
  - Customer refunds (τ-bench already covers it; less natural risk grading).
  - A real Kubernetes cluster like ITBench/SREGym (realistic, but hours per matrix and focused on root-cause skill, not framework reliability).
- **Consequences:**
  - Fast, cheap, deterministic runs.
  - Absolute success rates won't transfer to real systems; relative comparisons are the claim ([05 §5.7](05-safety-threat-model.md#57-residual-risks-stated-honestly)).
  - The simulator's realism needs care: runbooks and logs are written to look real.

### ADR-002: The environment is an MCP server that every framework uses
- **Context:** Tool definitions must be identical across frameworks, or the comparison measures tool
  wiring, not frameworks.
- **Decision:** **opsdesk-mcp** (FastMCP, Streamable HTTP, one endpoint per run). All four
  implementations connect to it through their native MCP support.
- **Alternatives:** Native function tools per framework (four copies, drift risk).
- **Consequences:**
  - Tests each SDK's MCP client too, which is realistic for 2026.
  - Per-framework schema conversion differences are logged ([03 §3.2](03-low-level-design.md#32-tool-catalog-opsdesk-mcp)).
  - External users can plug in any MCP-capable agent.

### ADR-003: Grade from environment state and the environment's action log
- **Decision:** Success, safety and side effects are graded from the final simulator state and the
  server-side action log. Framework events are used for trajectory, HITL timing and debugging only.
- **Alternatives:** Grade from framework traces (differs per framework, trusts the agent's own record);
  LLM judge on the transcript (costly, noisy).
- **Consequences:** Framework-neutral and hard to game. An LLM judge is used only for the written summary.

### ADR-004: Four implementations: raw loop, LangGraph, OpenAI Agents SDK, Claude Agent SDK
- **Context:** Agent JDs in 2026 name LangGraph most often, followed by the vendor SDKs. A no-framework
  baseline answers the question "does a framework help at all?"
- **Decision:**
  - The four implementations above.
  - **Microsoft Agent Framework** (1.0 since April 2026, Azure-aligned) is an optional fifth.
  - **Google ADK** is left for Project 4, where A2A is its strength.
- **Alternatives:** CrewAI (less common in agent-engineer JDs, role-play abstraction); Pydantic AI
  (good, but lower JD demand).
- **Consequences:**
  - Four codebases to maintain, which the shared spec and adapters keep small.
  - The raw loop reuses Project 2's client.

### ADR-005: Fix the model when comparing frameworks
- **Context:** The Claude Agent SDK runs Claude models only. The other frameworks run any model.
- **Decision:**
  - **E1** runs all four implementations on the **same Claude model**.
  - **E2** runs raw, LangGraph and the OpenAI SDK on an **OpenAI model** of similar tier.
  - Together they separate "framework effect" from "model effect".
- **Alternatives:** Each SDK with its vendor's model (confounds framework and model).
- **Consequences:**
  - The OpenAI Agents SDK is tested with a non-OpenAI model in E1 (through its multi-provider support), which is itself a finding.
  - **To verify when building:** which non-OpenAI model route the SDK supports best in 2026.

### ADR-006: Single agent is primary; multi-agent is an experiment
- **Context:** Multi-agent designs cost more tokens and add coordination failures (MAST). Single agents
  with good tools often match them on bounded tasks.
- **Decision:** All frameworks implement a **single agent**. LangGraph additionally implements supervisor
  + Investigator/Remediator/Communicator for **E7**.
- **Alternatives:** Multi-agent everywhere (four hard-to-compare designs); multi-agent only (hides the
  question).
- **Consequences:** A clean framework comparison, plus an evidence-based answer to "should this be
  multi-agent?"

### ADR-007: Native HITL in each framework, plus an environment backstop with signed tokens
- **Decision:**
  - Each implementation pauses for approval using its **own** mechanism, since that is what's being compared.
  - A shared approval service issues **single-use HMAC tokens** bound to run + tool + args hash, and the environment rejects risky calls without one.
- **Alternatives:** Only agent-side HITL (a skipped approval would silently succeed); only
  environment-side (can't compare framework HITL).
- **Consequences:**
  - Unapproved risky actions become measurable (backstop hits) and can't cause harm.
  - This mirrors Project 2's gateway confirmations, done here at the agent layer.

### ADR-008: Content-derived idempotency keys
- **Decision:** `key = hash(run_id, tool, canonical args)`, computed by the adapters' shared MCP call
  wrapper, verified by the environment ([03 §3.8](03-low-level-design.md#38-idempotency-keys)).
- **Alternatives:**
  - Model tool-call IDs (change on re-generation after resume).
  - Step counters (shift after resume).
- **Consequences:**
  - Retries after a crash are safe.
  - Intentional identical repeats need a distinguishing argument.
  - E5 measures duplicates with keys off vs on.

### ADR-009: Long-term memory as a shared MCP service
- **Context:** Only some frameworks have native long-term memory (LangGraph Store). Using native memory
  where it exists would make the frameworks' memory differ and confound E1/E8.
- **Decision:** **memory-mcp** (Postgres + pgvector, hybrid search, provenance, write policy) used by all
  implementations. The LangGraph build also has a short note/demo of the Store API for learning.
- **Alternatives:** LangGraph Store only; Mem0 or LangMem as a service (good products, but less control
  over the write policy being measured).
- **Consequences:** Fair memory comparisons and a measurable poisoning defence. Native-memory ergonomics
  go in the DX scorecard.

### ADR-010: pass^k with k = 4 and scenario-level bootstrap
- **Decision:** Every cell runs k = 4. Report pass@1, pass^k (unbiased estimator), bootstrap CIs over
  scenarios, and paired comparisons with Holm correction.
- **Alternatives:** k = 1 (hides inconsistency); k = 8 (2× cost for little extra insight at 50 scenarios).
- **Consequences:** ≈ 800 runs per 4-implementation matrix. The power limit (~8–10 points) is stated in
  the report.

### ADR-011: SQLite file per run
- **Decision:** Each run copies a seeded SQLite file. The MCP endpoint path carries the run ID.
- **Alternatives:** Postgres schema per run (slower setup, more moving parts); in-memory state (lost on
  simulator restart; harder to inspect).
- **Consequences:** Reset in milliseconds, easy parallelism, and the final state can be kept as an
  artifact for failure analysis.

### ADR-012: Project 2's gateway is optional and outside the eval path
- **Context:** Routing through the MCP Hub gateway would show reuse, but it adds Keycloak, token exchange
  and gateway-level confirmations that would confound agent-level HITL.
- **Decision:** Experiments connect agents **directly** to opsdesk-mcp. A demo configuration registers
  opsdesk-mcp as an upstream in the Project 2 gateway to show agent-level and gateway-level HITL working
  together.
- **Consequences:** Clean measurements, and a good interview story about layered controls.

### ADR-013: Scripted approver; LLM reporter only where needed
- **Decision:**
  - Approvals are decided by per-scenario rules, which are deterministic.
  - The reporter answers from a fact sheet: scripted keyword matches first, an LLM with a strict persona at temperature 0 as fallback, in ≤ 10 scenarios.
- **Alternatives:** LLM user simulator everywhere (τ-bench style; adds a second source of randomness).
- **Consequences:** Fewer moving parts. LLM-reporter scenarios are flagged so their noise can be seen.

### ADR-014: LLM judge only for the written summary
- **Decision:** One rubric-based judge for `summary_md`, calibrated on 40 human labels, with agreement
  reported. Everything else uses deterministic graders.
- **Consequences:** Cheap, stable grading. The judge's weaknesses affect only a secondary metric.

### ADR-015: OpenTelemetry GenAI conventions into Langfuse for every framework
- **Decision:** One OTLP pipeline and one backend (Langfuse, as in Projects 1 and 2). Framework-native
  tracing is bridged to OTel.
- **Alternatives:** LangSmith for LangGraph + OpenAI traces dashboard + … (three places to look, no
  comparison).
- **Consequences:** Traces are comparable across frameworks. Attribute names are pinned because the
  GenAI semconv is still "Development" status.

### ADR-016: Publish OpsDesk-50 and the scoring CLI; hold the test split until the report
- **Decision:** Publish opssim + dev scenarios from the start. Publish the 15 test scenarios with the
  report, along with their pre-committed hashes, which prove they weren't changed after tuning.
- **Consequences:** A reusable public artifact. Contamination risk is low and accepted.

### ADR-017: Claude Agent SDK with built-in tools disabled
- **Context:** The Claude Agent SDK ships file, shell and web tools by default. The other implementations
  don't have them.
- **Decision:** Allow only the MCP tools (`allowed_tools` restricted to `mcp__opsdesk__*` and
  `mcp__memory__*`, others disallowed), and use no settings files from the filesystem.
- **Consequences:** A fair comparison and no code-execution surface. The DX notes record what the SDK is
  best at (coding/OS agents) versus this use.

### ADR-018: Framework-native persistence for crash/resume; Temporal as an optional variant
- **Decision:** E5 tests each framework's own persistence:
  - LangGraph: Postgres checkpointer.
  - OpenAI Agents SDK: `RunState` + sessions.
  - Claude Agent SDK: session resume.
  - Raw loop: own message store.
- **Alternatives:** Put every implementation on Temporal (durable execution becomes identical and stops
  being a framework difference).
- **Consequences:** Shows real differences. An OpenAI Agents SDK + Temporal variant (integration GA in
  March 2026) is a "Could" feature that shows how much a durable-execution engine adds.

### ADR-019: Tune the prompt on the raw loop, then freeze
- **Context:** If the prompt is tuned while building the LangGraph version, LangGraph gets an unfair
  advantage.
- **Decision:** Tune the shared prompt on the **raw loop** with the dev split, freeze it (spec version
  recorded), then build the framework implementations. Framework-specific prompt changes are forbidden
  unless recorded in the DX notes.
- **Consequences:** A fairer comparison. Any remaining bias (for example the author's LangGraph
  experience) is listed under threats to validity.
