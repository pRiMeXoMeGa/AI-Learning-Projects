# AI Learning Projects

A market-driven plan for a senior GenAI and full-stack engineer (about 5 years of experience) targeting
these roles:

1. **GenAI / Applied AI Engineer**
2. **Agent Engineer**
3. **AI Full-stack Engineer**

| File | What's inside |
|---|---|
| [00-profile-gap-analysis.md](00-profile-gap-analysis.md) | Current strengths vs. 2026 JDs, gaps, résumé fixes |
| [01-market-analysis.md](01-market-analysis.md) | What JDs ask for, with a breakdown for each target role |
| [02-tech-stack.md](02-tech-stack.md) | Full tech stack rated for each role, with your current status (✅ / ⚠️ / ❌) |
| [03-projects.md](03-projects.md) | 7 projects + 1 optional + a CPG demand-planning capstone, mapped to roles and gaps |
| [04-roadmap.md](04-roadmap.md) | ~11-month plan (week by week for Projects 1–7 and the capstone), certification, interview prep, keywords |

## Project design docs

| Project | Status |
|---|---|
| [1. RAG Eval Lab](projects/01-rag-eval-lab/README.md) | 🟡 System design, tech stack, build plan and agentic mode done; guides added; **checked against the 2026 market** (review + updates) |
| [2. MCP Hub](projects/02-mcp-hub/README.md) | 🟡 System design, tech stack and build plan done (MCP 2026-07-28); core plan chosen (8 weeks); **checked against the 2026 market** |
| [3. Agent Reliability Harness](projects/03-agent-reliability-harness/README.md) | 🟡 System design, tech stack and build plan done (incident-triage agent × 4 implementations, OpsDesk-50 benchmark); core plan chosen (8 weeks) |
| [4. A2A Agent Mesh](projects/04-a2a-agent-mesh/README.md) | 🟡 System design done (A2A 1.0 mesh: ADK Commander + 3 agents from 3 frameworks, signed cards, cross-agent approvals, MCP-vs-A2A comparison), tech stack and build plan done; core plan chosen (4 weeks) |
| [5. Sandboxed Data-Analyst Agent](projects/05-sandboxed-data-analyst/README.md) | 🟡 System design done (Analyst: MCP sandbox broker on E2B + gVisor, safe generative-UI charts, escape suite, AnalystBench-50 + DABstep), tech stack and build plan done; core plan chosen (4 weeks) |
| [6. AI SaaS on Next.js](projects/06-ai-saas-nextjs/README.md) | 🟡 System design done (ClauseDesk: Next.js 16, AI SDK 7, durable workflows, RLS multi-tenancy, Stripe usage billing, CUAD evals), tech stack and build plan done; core plan chosen (6 weeks) |
| [7. LLM Gateway & Cost Router](projects/07-llm-gateway-router/README.md) | 🟡 System design done (Switchboard: official-SDK gateway, budgets, fallbacks, prompt/exact/scoped semantic caching with poisoning tests, five routing policies on Pareto curves, build-vs-buy), tech stack and build plan done; core plan chosen (4 weeks) |
| [🏆 Capstone: Demand-Planning Copilot](projects/08-demand-planning-copilot/README.md) | 🟡 System design done (Cadence: models make the numbers, agents propose bounded evidence-cited revisions and orders with approvals, FVA on PlanBench-60, replenishment simulation, reuses P1–P7), tech stack and build plan done; core plan chosen (6 weeks, fits the roadmap) |

## TL;DR
- **Your strengths are already in demand:** LangGraph multi-agent systems, MCP servers, guardrails,
  multi-LLM platforms, FastAPI + React/TypeScript, and team leadership.
- **What's missing is public, measured proof.** Specifically:
  - evals (Ragas/DeepEval, CI gates, trajectory evals)
  - observability (Langfuse/OTel)
  - remote MCP/A2A and vendor agent SDKs
  - sandboxed agents
  - Next.js + Vercel AI SDK
- **Plan:** Projects 1 → 2 → 3 → 6 → 5 → 4 → 7, then the **Demand-Planning Copilot** capstone on the
  public M5 dataset. It rebuilds your CPG forecasting experience as an open-source portfolio piece.
