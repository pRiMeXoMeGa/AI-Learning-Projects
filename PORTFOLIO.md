# Portfolio at a Glance

**Mohd Aqib** · Senior GenAI & Agent Engineer · LangGraph · MCP · RAG · Python/FastAPI · React/TypeScript · Azure/AWS

Eight public projects that show, with **measured results**, the production skills behind my client work
(PepsiCo and Unilever demand forecasting, agentic workflows, MCP tool servers and guardrails). Everything is
built on public data, so every design, number and line of code can be discussed without NDA limits.

> **Status (September 2026):** every project is fully designed (architecture, evaluation plan, threat
> model, build plan). Builds run from October 2026 to August 2027 in the order below. Each row links to
> its design, and the results fill in as each project ships.

## The eight projects

| # | Project | What it is | Skills it proves | Roles | Ships |
|---|---|---|---|---|---|
| 1 | [**RAG Eval Lab**](projects/01-rag-eval-lab/README.md) | RAG over SEC 10-K filings where every retrieval technique is measured, plus an agentic mode | Evals (Ragas, DeepEval, LLM-as-judge with calibrated judges), CI quality gates, hybrid search + reranking, pgvector, Langfuse/OTel, trajectory evals, Azure + Terraform | GenAI, Agent | Nov 2026 |
| 2 | [**MCP Hub**](projects/02-mcp-hub/README.md) | An open-source MCP server (`india-mf-mcp`) and a secure MCP gateway on the 2026-07-28 spec | Remote MCP, OAuth 2.1 (Keycloak, token exchange), Cedar policies, tool-poisoning and injection defences, MCP Apps, Python + TypeScript SDKs, tool-design evals | Agent, Full-stack | Jan 2027 |
| 3 | [**Agent Reliability Harness** (OpsDesk)](projects/03-agent-reliability-harness/README.md) | One incident-triage agent built four ways and scored on a published benchmark (OpsDesk-50) | LangGraph, OpenAI Agents SDK, Claude Agent SDK, pass^k reliability, human-in-the-loop approvals, crash/resume + idempotency, injection evals, calibration | Agent | Mar 2027 |
| 4 | [**A2A Agent Mesh**](projects/04-a2a-agent-mesh/README.md) | Four agents from four frameworks cooperating over A2A 1.0 | A2A protocol, Google ADK + Gemini, signed Agent Cards, delegated auth (token exchange), cross-agent approvals, fault injection, MCP-vs-A2A measured | Agent | Apr 2027 |
| 5 | [**ClauseDesk**](projects/06-ai-saas-nextjs/README.md) | A multi-tenant contract-review SaaS with cited answers and clause extraction | Next.js 16, Vercel AI SDK 7 (agents, tool approvals, resumable streams), durable workflows, Postgres RLS, Stripe usage billing, E2E tests on preview deploys, CUAD extraction evals | Full-stack, GenAI | May 2027 |
| 6 | [**Analyst**](projects/05-sandboxed-data-analyst/README.md) | A data-analyst agent that writes and runs code in isolated sandboxes and renders safe charts | Sandboxed code execution (E2B Firecracker, gVisor), MCP sandbox broker, output-handling security, generative UI, code-agent benchmarks (DABstep) | Agent, Full-stack | Jun 2027 |
| 7 | [**Switchboard**](projects/07-llm-gateway-router/README.md) | A supply-chain-hardened LLM gateway that the other projects route through | Cost engineering, prompt/exact/semantic caching (with poisoning tests), learned model routing, budgets, fallbacks, circuit breakers, FinOps dashboards, build-vs-buy analysis | GenAI | Jul 2027 |
| 🏆 | [**Cadence**](projects/08-demand-planning-copilot/README.md) | A demand-planning copilot for CPG (capstone): models make the forecast, agents make bounded, evidence-cited adjustments and order proposals that a planner approves | Forecasting (LightGBM, Chronos-2, hierarchical reconciliation) on M5, Forecast Value Added, replenishment simulation, multi-agent LangGraph, and all of the above integrated | All three | Aug 2027 |

*(Projects are numbered by design order; ClauseDesk is built before Analyst because Analyst reuses its web shell.)*

## Skills coverage

| Skill area | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 🏆 |
|---|---|---|---|---|---|---|---|---|
| RAG and retrieval quality | ● | | | | ● | | | ● |
| Evals (golden sets, LLM-as-judge, CI gates) | ● | ● | ● | ● | ● | ● | ● | ● |
| Agent frameworks (LangGraph, OpenAI, Claude, ADK) | ● | | ● | ● | ● | ● | | ● |
| MCP and A2A | ● | ● | ● | ● | | ● | | ● |
| Human-in-the-loop and approvals | | ● | ● | ● | ● | | | ● |
| AI security (injection, poisoning, sandboxing, auth) | ● | ● | ● | ● | ● | ● | ● | ● |
| Full-stack (React/Next.js, streaming UI, SaaS) | | ● | ● | | ● | ● | | ● |
| Cost, latency and observability | ● | ● | ● | ● | ● | | ● | ● |
| Cloud and delivery (Azure, Terraform, CI/CD) | ● | ● | ● | ● | ● | ● | ● | ● |
| Forecasting and planning (my domain) | | | | | | | | ● |

## Résumé bullets

The blanks (__) are filled with measured results as each project ships. No number goes in until it has
been measured.

1. **RAG Eval Lab:** "Raised RAG faithfulness from __ to __ and recall@5 from __ to __ on SEC 10-K filings via
   hybrid search and reranking, measured with calibrated LLM judges; CI eval gates block quality regressions."
2. **MCP Hub:** "Built an OAuth 2.1-secured MCP gateway (MCP 2026-07-28, Keycloak, Cedar policies) aggregating
   __ servers / __ tools with tool-poisoning and injection defences; published an open-source MCP server with
   __ installs."
3. **Agent Reliability Harness:** "Built an agent reliability harness (50 incident scenarios, pass^k, safety and
   HITL metrics) comparing the same agent in LangGraph, OpenAI Agents SDK and Claude Agent SDK; guards and
   approval backstops cut unsafe actions from __% to __% and failed runs from __% to __%."
4. **A2A Agent Mesh:** "Built a cross-framework A2A 1.0 agent mesh (Google ADK, LangGraph, OpenAI Agents SDK,
   Claude Agent SDK) with signed Agent Cards, per-hop token exchange and cross-agent human approval; passed the
   A2A TCK and blocked __/14 attack classes."
5. **ClauseDesk:** "Built and deployed a multi-tenant AI SaaS (Next.js 16, AI SDK 7, Postgres RLS + pgvector)
   with durable extraction workflows, resumable streaming, tool approvals and Stripe usage billing; __ F1 on
   CUAD clause extraction, 0 cross-tenant leaks."
6. **Analyst:** "Built a sandboxed data-analyst agent (AI SDK 7, MCP sandbox broker, E2B Firecracker + gVisor)
   with safe generative-UI charts; __% on a 50-question benchmark, __% on DABstep; blocked __/21 escape and
   abuse attacks."
7. **Switchboard:** "Built an LLM gateway with budgets, fallbacks, prompt/exact/semantic caching and learned
   routing; cut cost per 1k requests by __% at < __ points of quality loss, with a measured semantic-cache
   false-hit rate of __% and poisoning defences."
8. **Cadence:** "Built an open-source multi-agent demand-planning copilot (LangGraph, MCP, A2A, Next.js,
   Chronos-2) on the M5 dataset; agent forecast adjustments added __ points of Forecast Value Added with a __%
   harmful-adjustment rate and improved simulated fill rate by __ points, at $__ per planning session."

## How to read this repo in 10 minutes

1. This page.
2. The capstone's [Start Here](projects/08-demand-planning-copilot/docs/00-start-here.md): the project closest
   to my day job.
3. Any project's **Evaluation Design** (doc 04): how each claim above will be measured.
4. The [roadmap](04-roadmap.md) for the build order and dates.
