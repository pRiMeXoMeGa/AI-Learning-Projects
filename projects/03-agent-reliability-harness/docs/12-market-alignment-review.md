# 12. Market Alignment Review (September 2026)

> **Question answered:** Does every approach in Project 3 match how agent frameworks, agent reliability
> evaluation, human-in-the-loop and durable execution are built and hired for in 2026, and what's missing?
>
> **Method:** ~14 web searches across framework releases (OpenAI Agents SDK, Claude Agent SDK, LangGraph),
> 2026 agent-reliability research, agent and SRE benchmarks, durable-execution platforms, prompt-injection
> defence research, OpenTelemetry status and job-market guides (sources at the end). Each area was rated;
> recommended changes were **applied to the design docs**, and optional ones are listed as a backlog.

## 12.1 Verdict

**The design is well aligned. Its central bet, "reliability, not accuracy", has become the mainstream
2026 research position.** The evidence supports its core choices:

1. **pass^k over pass@1.** Microsoft's **Thinkingbox-bench** (August 2026: 507 policy-bound business
   workflows, isolated MCP tool sessions, grading on terminal backend state) reports the strongest models at
   **65% pass@1 but only 25% pass^20**. That's the same method as OpsDesk: MCP environment, state-based
   grading, repeated runs.
2. **Four reliability dimensions.** The design already follows Rabanser et al. ("Towards a Science of AI
   Agent Reliability", **ICML 2026**): consistency, robustness, predictability and safety. The paper's
   headline findings (outcome consistency of only 30–75%, predictability the weakest dimension, large drops
   under semantics-preserving rephrasing) are exactly what the harness measures.
3. **The three SDKs are the right three.** 2026 comparisons settle into lanes: LangGraph for stateful
   orchestration, checkpoints and approval gates (the most production mileage); the OpenAI Agents SDK for
   simplicity (and, since April 2026, built-in HITL, sessions and sandboxed "harness" agents); the Claude
   Agent SDK for MCP-native work with hooks and permission callbacks.
4. **HITL is standard practice.** LangChain's 2026 State of Agent Engineering report (as cited in 2026
   durable-execution write-ups) found about **60%** of production agent systems added human intervention
   points. Framework-native approvals plus an
   environment backstop is the right design.
5. **A light simulator is the right call.** SRE benchmarks keep multiplying (**ITBench-AA** by Artificial
   Analysis and IBM, where frontier models score below 50%; **SREGym**; **InfraBench**), but they all test
   root-cause skill on real infrastructure. None compares frameworks on safety, HITL and crash recovery,
   which is this project's gap.

**But six areas needed updating:**

| # | Gap | Why it matters in 2026 | Change applied |
|---|---|---|---|
| 1 | **Robustness (E9) was cut from the core plan** | An August 2026 noise-floor audit found semantics-preserving prompt perturbations cause **11–58× more variance than reruns**; Rabanser et al. found large drops under rephrasing. A framework difference smaller than the rephrasing noise isn't a finding | **E9 back in the core plan**; the report shows every framework difference next to the **paraphrase noise floor** |
| 2 | **LangGraph durability mode was just "recorded"** | LangGraph's default durability mode is `async` (checkpoint written while the next step runs). In a crash, that window is where duplicate actions come from, and it's what 2026 durable-execution write-ups warn about | E5 runs LangGraph in **`sync` and `async`** modes: a direct measurement of the speed vs safety trade-off |
| 3 | **Claude Agent SDK resume relied on local session files** | The SDK now supports a custom **`SessionStore`** (transcripts in a database, with batched or immediate flushes) | Claude sessions stored in **Postgres**, so a killed worker's replacement can resume on any machine; flush mode recorded for E5 |
| 4 | **Adaptive injection attacks were optional** | 2026 research: defences that look strong on static benchmarks fall to adaptive attackers; the field has moved to **deterministic policies outside the model** (CaMeL, FIDES, Progent), evaluated adaptively. Project 2 already made adaptive attacks core | A **slim adaptive attack** (reusing Project 2's attacker) moves into the core E6; the approval backstop is framed as a **deterministic reference monitor** in the threat model |
| 5 | **Grader changes weren't versioned** | τ²-bench's July 2026 grading fix made results from before and after **not comparable**. OpsDesk-50 is being published, so the same thing will happen to it | **Semantic versioning for graders and scenarios**; every result records the grader version; the scoring CLI refuses to compare across major versions |
| 6 | **Predictability lacked discrimination** | Rabanser et al. split predictability into calibration **and discrimination** (can confidence rank successes above failures?) | **AUROC** of `confidence` added next to Brier and ECE |

**Effort impact:** about **+6 h** (§12.4). The core plan goes from ~108 h to **~114 h** and **stays at 8
weeks** at ~14.25 h/week (inside the 12–15 h/week range). The roadmap length doesn't change.

## 12.2 Scorecard

✅ aligned · ⚠️ updated · ➕ added · 🔸 optional backlog · ⛔ considered and rejected (with reason)

| Area | Status | Summary |
|---|---|---|
| Domain: incident triage in a simulator (OpsSim) | ✅ | Differentiated from τ-bench and from real-cluster SRE benchmarks; related work updated |
| Environment as an MCP server; grading from state + action log | ✅ | Same method as Thinkingbox-bench (2026) |
| Four implementations (raw, LangGraph, OpenAI Agents SDK, Claude Agent SDK) | ✅ | The three SDKs match the 2026 "lanes"; raw loop shows what frameworks add |
| Fixed model for the framework comparison | ✅ | Standard practice; E2 covers the model axis |
| OpenAI Agents SDK HITL (`interruptions`, `RunState`) | ✅ ⚠️ | Pin a release after the April 2026 update; re-check the HITL API in the F0 spike; sandbox agents are out of scope (Project 5) |
| Claude Agent SDK (hooks, `can_use_tool`, sessions) | ⚠️ | Postgres `SessionStore` for resume |
| LangGraph (interrupts, checkpointer) | ⚠️ | Durability modes measured in E5 |
| pass^k, bootstrap over scenarios, paired tests, Holm | ✅ | Matches 2026 reliability research |
| Four reliability dimensions | ✅ ⚠️ | Already there; discrimination (AUROC) added |
| Robustness (paraphrase, fault rate) | ⚠️ | Back in core; noise floor reported |
| Failure taxonomy (MAST-based) with human κ | ✅ | Still the standard reference |
| HITL: native approvals + signed-token backstop | ✅ | Matches the 60%-of-production-systems pattern; backstop = reference monitor |
| Crash/resume + idempotency keys | ✅ ⚠️ | Durability modes and session store added |
| Temporal (OpenAI Agents SDK integration, GA March 2026) | ✅ 🔸 | Still optional; Temporal's **LangGraph plugin** (Public Preview, July 2026) added as a backlog variant |
| Injection evals (static + spotlighting) | ⚠️ | Slim adaptive attack in core |
| Grader and scenario versioning | ➕ | Semver, recorded per result |
| OpenTelemetry GenAI conventions | ✅ | Still "Development" in 2026; the design already pins the version |
| Memory (deferred) | ✅ 🔸 | Still deferred; built in the capstone with a strict write policy |
| Multi-agent variant (deferred) | ✅ 🔸 | Still deferred; the capstone runs a single- vs multi-agent comparison |
| Microsoft Agent Framework (optional) | 🔸 | Unchanged; add if target JDs name it |
| Building on an existing eval harness (HAL, Inspect) | ⛔ | See §12.3 |

## 12.3 Considered and rejected (and how to defend it in an interview)

| Technique | Why it's not in this project | Evidence |
|---|---|---|
| **Build the runner on HAL or Inspect** | Both are good general harnesses, but this project needs process-tree kills at chosen points, per-run environment state, and approval services in the loop. Those are the parts that make the comparison new. HAL itself has paused its leaderboard to focus on reliability, the same direction | HAL (ICLR 2026) paper and repo |
| **Run ITBench-AA or SREGym as well** | They need real Kubernetes clusters and measure root-cause skill, not framework reliability; hours per matrix | ITBench-AA, SREGym papers |
| **CaMeL-style dual-LLM agent** | Would redesign all four implementations; the environment backstop already gives the "deterministic policy outside the model" property for risky actions | 2026 injection-defence research |
| **OpenAI sandbox agents / model-native harness** | Designed for agents that edit files and run commands; this agent only calls MCP tools. Sandboxed code execution is Project 5 | OpenAI Agents SDK April 2026 update |
| **LangSmith / hosted deployment for LangGraph** | Unchanged from the design: hosted features would muddy an open-source comparison | ADR-004 |

## 12.4 Changes applied to the design

| # | Change | Where | Effort |
|---|---|---|---|
| C1 | **E9 robustness back in the core plan** (20 scenarios × 2 paraphrases; fault rate 0 / 10 / 25%) | F19, [01](01-requirements.md) FR-34 → Must, build plan | +2 h |
| C2 | **Paraphrase noise floor** shown next to every framework difference; **AUROC** (discrimination) added to predictability | [04 §4.2, §4.4](04-evaluation-design.md), F19 | +0.5 h |
| C3 | **LangGraph durability modes** `sync` vs `async` in E5 | F14, [04 §4.5](04-evaluation-design.md), ADR-018, [08](08-tech-stack.md) | +1 h |
| C4 | **Claude Agent SDK `SessionStore` in Postgres** for resume; flush mode recorded | F11, F14, [08](08-tech-stack.md), ADR-018 | +0.5 h |
| C5 | **Slim adaptive injection attack** in core E6 (reuses Project 2's attacker: up to 3 rewrites, best and worst implementation from E1); full F25 stays optional | F18, F23, [04 §4.5](04-evaluation-design.md), [05](05-safety-threat-model.md) | +1.5 h |
| C6 | **Grader and scenario semver**; version stored in every result; scoring CLI refuses cross-major comparisons | F7, F20, ADR-016 | +0.5 h |
| C7 | OpenAI Agents SDK pinned **after the April 2026 update**; HITL API re-checked in the F0 spike | F0, [08](08-tech-stack.md) | 0 h |
| C8 | Related work: Thinkingbox-bench, ITBench-AA, InfraBench; backstop framed as a reference monitor | [00](00-start-here.md), [01](01-requirements.md), [05](05-safety-threat-model.md) | 0 h |
| | **Total added** | | **≈ +6 h** |

## 12.5 Optional backlog

| ID | Item | Why it's interesting | Effort |
|---|---|---|---|
| P1 | **Temporal LangGraph plugin** variant in E5 (Public Preview) | Compares the same LangGraph agent on its own checkpointer vs on Temporal | ~3 h |
| P2 | Full **F25 adaptive attacks** (5 rewrites, all implementations, spotlighting on/off) | Completes the adaptive picture | ~1.5 h (on top of C5) |
| P3 | Run the four agents on a **small Thinkingbox-bench subset** | An external check that the framework ranking holds outside OpsDesk | ~4 h |
| P4 | **Microsoft Agent Framework** fifth implementation (F23) | If target JDs are Azure-heavy | ~6 h |
| P5 | Deferred core-plan features: memory (F15), multi-agent (F16), Azure demo (F21), inbox polish | As before | ~17 h |

## 12.6 Market keywords this project now covers

LangGraph (interrupts, checkpointers, **durability modes**), **OpenAI Agents SDK** (HITL, `RunState`,
sessions), **Claude Agent SDK** (hooks, permission callbacks, **session store**), MCP, **pass^k**,
agent reliability evaluation (consistency, robustness, predictability, safety), calibration (Brier, ECE,
AUROC), failure taxonomy (MAST), human-in-the-loop approvals, **durable execution** (Temporal), crash
recovery and idempotency, prompt-injection evaluation (static and **adaptive**), OWASP Agentic Top 10,
OpenTelemetry GenAI tracing, Langfuse, benchmark design and versioning.

## 12.7 Sources

**Reliability research and benchmarks**
- [Towards a Science of AI Agent Reliability (Rabanser et al., ICML 2026)](https://arxiv.org/abs/2602.16666) · [authors' summary](https://www.normaltech.ai/p/new-paper-towards-a-science-of-ai)
- [One Success Isn't Reliability: Thinkingbox (Microsoft, 2026)](https://arxiv.org/abs/2608.19741) · [ThinkingBox-Bench dataset](https://huggingface.co/datasets/microsoft/ThinkingBox-Bench)
- [Noise Floor Audit for Agent Benchmarks (2026)](https://arxiv.org/abs/2608.22331) · [τ²-bench run-to-run noise issue](https://github.com/sierra-research/tau2-bench/issues/540)
- [Beyond pass@1: a reliability science framework for long-horizon agents](https://arxiv.org/pdf/2603.29231)
- [τ²-bench (GitHub)](https://github.com/sierra-research/tau2-bench) · [τ²-Bench Airline leaderboard (OpenRouter)](https://openrouter.ai/benchmarks/tau2-bench-airline)
- [Holistic Agent Leaderboard (ICLR 2026)](https://arxiv.org/abs/2510.11977) · [hal-harness](https://github.com/princeton-pli/hal-harness)

**SRE and infrastructure agent benchmarks**
- [ITBench-AA launch (Artificial Analysis)](https://artificialanalysis.ai/articles/itbench-aa-launch) · [SREGym](https://arxiv.org/pdf/2605.07161) · [InfraBench](https://arxiv.org/pdf/2608.11234) · [awesome-LLM-AIOps](https://github.com/Jun-jie-Huang/awesome-LLM-AIOps)

**Frameworks**
- [The best agent framework in 2026: LangGraph vs OpenAI Agents SDK vs Claude Agent SDK](https://medium.com/data-science-collective/the-best-agent-framework-in-2026-langgraph-vs-openai-agents-sdk-vs-claude-agent-sdk-2c64e0b378d9) · [AI agent frameworks 2026 (Let's Data Science)](https://letsdatascience.com/blog/ai-agent-frameworks-compared) · [8 SDKs compared (Morph)](https://www.morphllm.com/ai-agent-framework)
- [OpenAI updates Agents SDK with sandbox and harness (Help Net Security)](https://www.helpnetsecurity.com/2026/04/16/openai-agents-sdk-harness-and-sandbox-update/) · [DevOps.com coverage](https://devops.com/openai-upgrades-its-agents-sdk-with-sandboxing-and-a-new-model-harness/) · [OpenAI Agents SDK docs](https://openai.github.io/openai-agents-python/)
- [Claude Agent SDK Python reference](https://platform.claude.com/docs/en/agent-sdk/python) · [Hooks](https://platform.claude.com/docs/en/agent-sdk/hooks) · [claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)

**Durable execution and HITL**
- [OpenAI Agents SDK + Temporal integration](https://temporal.io/blog/announcing-openai-agents-sdk-integration) · [Temporal LangGraph plugin](https://temporal.io/blog/temporal-langgraph-plugin-durable-execution)
- [Durable execution for AI agents: LangGraph, DBOS, Inngest and Temporal (HackerNoon)](https://hackernoon.com/durable-execution-for-ai-agents-langgraph-dbos-inngest-and-temporal-compared) · [Durable AI agents in 2026 (Reactify)](https://www.reactify-solutions.com/articles/durable-ai-agents-2026) · [Durable execution patterns (Zylos)](https://zylos.ai/research/2026-02-17-durable-execution-ai-agents/)
- [Durable execution in LangGraph (Vadim's blog)](https://vadim.blog/durable-execution-agents-that-survive-failure-and-resume-where-they-left-off) · [LangGraph durable runtime (ZenML)](https://www.zenml.io/blog/langgraph-durable-runtime)

**Prompt injection**
- [Adaptive evaluation of out-of-band defenses against prompt injection (2026)](https://arxiv.org/abs/2606.26479) · [Adaptive attacks break defenses against indirect prompt injection](https://arxiv.org/pdf/2503.00061) · [AgentDojo](https://www.emergentmind.com/topics/agentdojo-benchmark)

**Observability and market**
- [OpenTelemetry GenAI conventions are not stable yet (2026)](https://dev.to/azena-ai/opentelemetrys-genai-semantic-conventions-are-not-stable-yet-heres-what-actually-shipped-in-2026-3mke) · [GenAI observability with OpenTelemetry (OTel blog)](https://opentelemetry.io/blog/2026/genai-observability/)
- [AI evals engineer career guide 2026](https://jobsbyculture.com/blog/ai-evals-engineer-career-guide-2026) · [How to recruit AI agent engineers in 2026](https://www.herohunt.ai/blog/how-to-recruit-ai-agent-engineers-in-2026/) · [SREs now keep AI agents reliable too](https://interviewstack.io/blog/how-ai-is-changing-site-reliability-engineer-2026)
