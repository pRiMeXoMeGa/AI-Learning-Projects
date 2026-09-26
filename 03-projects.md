# Projects for GenAI Engineer · Agent Engineer · AI Full-stack Engineer

These projects are chosen for your profile. You've already shipped LangGraph multi-agent systems, MCP
servers, RAG, guardrails and React UIs **for clients**, so a beginner "build a RAG chatbot" project won't
help. Each project here does three things:

1. **Fills a gap** from [00-profile-gap-analysis.md](00-profile-gap-analysis.md): evals, observability,
   remote MCP/A2A, agent SDKs, sandboxing, Next.js/Vercel AI SDK, cost engineering.
2. **Turns hidden client experience into public proof**: open-source repos with metrics.
3. **Maps to a target role** so you can pin the right repos for each application.

> ⚠️ Build everything from scratch on **public data**. Don't reuse PepsiCo, Unilever, Fractal or
> LTIMindtree code, prompts or data.

## Overview

| # | Project | Role(s) | Gaps it fills | Time |
|---|---|---|---|---|
| 1 | **RAG Eval Lab**: measured, eval-gated RAG + agentic mode | GenAI, Agent | Evals, CI gates, reranking, Langfuse, agentic RAG + trajectory evals | 7 wks (full plan) |
| 2 | **MCP Hub**: remote MCP servers + gateway | Agent, Full-stack | Remote MCP, OAuth 2.1, MCP client, tool-design evals, MCP security | 8 wks (core plan) |
| 3 | **Agent Reliability Harness**: one agent, three frameworks | Agent | Trajectory evals, agent SDKs, memory, tracing | 3–4 wks |
| 4 | **A2A Agent Mesh** | Agent | A2A, cross-framework interop | 1–2 wks |
| 5 | **Sandboxed Data-Analyst Agent** with generative UI | Agent, Full-stack | Sandboxing, code-execution security, generative UI | 3 wks |
| 6 | **Full-stack AI SaaS on Next.js** | Full-stack | Next.js, Vercel AI SDK, resumable streams, billing, multi-tenancy | 3–4 wks |
| 7 | **LLM Gateway & Cost Router** | GenAI | Semantic cache, routing, cost/latency metrics, OTel | 2 wks |
| 8 | *(optional)* **Realtime Voice Agent** | Full-stack | Realtime/voice UX | 2 wks |
| 🏆 | **Capstone: Demand-Planning Copilot for CPG** | All three | Everything, in your own domain | 5–6 wks |

**Suggested order:** 1 → 2 → 3 → 6 → 5 → 4 → 7 → Capstone. Projects 1 and 3 close the biggest gap
(evals + observability) first. The **capstone reuses components from projects 1–7**, so nothing you build
is thrown away.

**GitHub pins by role**
- GenAI Engineer: Capstone, 1, 7, 3
- Agent Engineer: Capstone, 3, 2, 5
- AI Full-stack Engineer: Capstone, 6, 5, 2

---

## 1. RAG Eval Lab: measured, eval-gated RAG  *(GenAI)*

📐 **Detailed system design:** [projects/01-rag-eval-lab](projects/01-rag-eval-lab/README.md)

**Why this project:** Your résumé shows RAG but no quality numbers. Hiring managers want to see that
you can *measure and improve* retrieval.

**Data:** A public corpus with hard questions, e.g. SEC 10-K filings of 20 CPG companies, or RBI/SEBI
circulars. These give you tables, cross-document questions and numbers.

**Stack:** FastAPI (SSE), Docling, Postgres + **pgvector** (with Pinecone for comparison), BM25,
**Cohere/BGE reranker**, query rewriting, **Ragas + DeepEval**, **Langfuse**, **LangGraph** (agent mode
only), GitHub Actions, Docker.

**Milestones**
1. Build a 150-question **golden set** (synthetic generation + manual review) with question types:
   factoid, multi-hop, table/numeric, unanswerable.
2. Naive baseline, then measure context precision/recall, faithfulness and answer correctness.
3. Add one technique at a time (hybrid → reranker → query rewriting → parent-document chunking →
   contextual chunk headers) and **record each step's effect** in an ablation table.
4. Calibrate an LLM-as-judge against 50 human labels and report agreement.
5. **CI gate:** a PR fails if faithfulness or recall drops more than a threshold.
6. Langfuse traces with cost and latency per query; p50/p95 dashboard.
7. **Agentic mode:** a read-only LangGraph research agent (search, read, calculate tools, hard budgets)
   that reuses the same retrieval and answer generator; **trajectory evals** compare agent vs. pipeline
   per question type, and an `auto` router sends only the types where the agent wins.

**Résumé bullet templates:**
- "Raised RAG faithfulness from __ to __ and recall@5 from __ to __ via hybrid search + reranking; added
  CI eval gates in GitHub Actions that block quality regressions."
- "Built an agentic RAG mode (LangGraph) with trajectory evals; improved cross-company answer correctness
  by __ points and routed only complex questions to the agent, keeping average cost at __× the pipeline."

---

## 2. MCP Hub: remote MCP servers + gateway  *(Agent, Full-stack)*

📐 **Detailed system design:** [projects/02-mcp-hub](projects/02-mcp-hub/README.md). The design targets the
**MCP 2026-07-28 specification** (stateless core, multi round-trip requests for elicitation, CIMD client
registration, no token passthrough). The domain chosen is **Indian mutual-fund data (AMFI)**; the
milestones below are refined there.

**Why this project:** You've built internal FastMCP servers. This project takes you from "built MCP
tools" to **MCP platform expert**, which is the strongest differentiator in agent JDs, and the result is
public.

**Stack:** FastMCP (Python) + MCP TypeScript SDK, streamable HTTP, **OAuth 2.1** (Auth0/Keycloak/Entra),
Postgres, Redis, Docker, MCP Inspector, pytest.

**Milestones**
1. **Open-source MCP server** for a useful public API or dataset, e.g. Indian mutual-fund NAVs,
   data.gov.in, or UAE open data. Publish it to the MCP registry and PyPI/npm.
2. Remote deployment with **OAuth 2.1**, scoped tools per user, rate limits, audit log of every call.
3. Use **elicitation** (asking the user for input mid-call; since 2026-07-28 done through multi
   round-trip requests) and a confirmation flow for write tools.
4. **MCP gateway:** one endpoint that aggregates several MCP servers, with a tool registry, per-tenant
   allow-lists and policy checks. Your security background shows here.
5. Write your own **MCP client** in a raw tool-calling loop (no framework), for both the Claude and
   OpenAI APIs.
6. **Tool-design evals:** compare tool granularity and description wording, and measure agent task
   success and token usage.
7. Threat-model write-up: tool poisoning, prompt injection via tool results, confused-deputy attacks.

**Résumé bullet template:** "Built an OAuth-secured MCP gateway aggregating __ servers / __ tools with
per-tenant policies; open-source MCP server with __ installs/stars."

---

## 3. Agent Reliability Harness: one agent, three frameworks  *(Agent)*

**Why this project:** Agent JDs ask for **trajectory evals, tracing and framework breadth**. You know
LangGraph deeply; this shows you can judge frameworks objectively.

> **How this differs from Project 1's agent mode:** Project 1's agent is read-only and single-agent, and
> is judged against a fixed pipeline. Here the agent **acts** (tools with side effects, HITL approval,
> crash/resume, memory) and the comparison is **between frameworks**. The trajectory-metric code from
> Project 1 (F19) can be reused as a starting point.

**Task:** An "IT incident triage" or "customer-refund" agent: read the ticket, query the MCP tools from
project 2, decide, ask a human for approval, act.

**Stack:** **LangGraph** (Postgres checkpointer, interrupts, long-term memory store),
**OpenAI Agents SDK**, **Claude Agent SDK** (or Google ADK), **Langfuse**/**LangSmith**,
**OpenTelemetry**, DeepEval, pytest.

**Milestones**
1. Build 50 scripted scenarios with expected **tool trajectories** and outcomes, including adversarial
   ones (injected instructions in tickets, missing data, tool failures).
2. LangGraph implementation: supervisor + specialists, HITL approval, crash/resume, time-travel debugging,
   per-user long-term memory.
3. Implement the same agent with the OpenAI Agents SDK and the Claude Agent SDK, using the same MCP tools.
4. Harness scores: task success, trajectory match, steps, tokens, cost, latency, and the rate at which
   prompt injection succeeds.
5. Guardrails: step, cost and time budgets; loop detection; tool allow-lists.
6. Publish a comparison report in the README and as a blog/LinkedIn post.

**Résumé bullet template:** "Built an agent evaluation harness (__ scenarios, trajectory + outcome metrics)
comparing LangGraph, OpenAI Agents SDK and Claude Agent SDK; cut failed runs from __% to __%."

---

## 4. A2A Agent Mesh  *(Agent)*

**Stack:** A2A protocol SDK, agents from project 3 (LangGraph + one vendor SDK), Agent Cards, auth.

**Milestones**
1. Expose two agents built with different frameworks as **A2A servers** with Agent Cards.
2. An orchestrator agent discovers them, delegates tasks, and handles long-running tasks and streaming
   updates.
3. Write up how **MCP (agent↔tool)** and **A2A (agent↔agent)** fit together.

Small project, but it gives you a strong interview story.

---

## 5. Sandboxed Data-Analyst Agent with generative UI  *(Agent, Full-stack)*

**Why this project:** Sandbox security is an explicit agent-JD requirement, and your M.Tech in
Information Security makes this a natural fit. The generative-UI front end also counts toward the
full-stack role.

**Stack:** Claude Agent SDK or LangGraph, **E2B** (or Docker + gVisor) sandbox, pandas/DuckDB,
**Next.js + Vercel AI SDK** (charts rendered as generative UI components), FastAPI.

**Milestones**
1. Loop: the agent writes code, runs it in the sandbox, reads the result, fixes errors.
2. Hard isolation: no network, CPU/memory/time limits, read-only data mounts, destroyed per session.
3. Generative UI: the agent returns chart/table components (not just text) that stream into the page.
4. Benchmark on 50 analysis questions (correctness, iterations, cost).
5. Threat model: sandbox escape, data exfiltration, prompt injection hidden in CSV cells.

---

## 6. Full-stack AI SaaS on Next.js  *(Full-stack)*

**Why this project:** Next.js and the Vercel AI SDK are your biggest full-stack gaps. You already know
React and TypeScript, so this goes quickly.

**Idea:** "Contract/Policy Copilot": upload documents, chat with citations, run extraction workflows.
It reuses the RAG from project 1.

**Stack:** **Next.js (App Router, Server Components, Server Actions)**, **Vercel AI SDK** (`useChat`,
`streamText`, tool calls, structured outputs), shadcn/ui, Auth.js or Clerk, Postgres + pgvector (Drizzle
or Prisma), FastAPI Python service for heavy AI work, Stripe usage-based billing, Vercel + AWS/Azure.

**Milestones**
1. Streaming chat with tool-call timelines, citations and a document-preview side panel.
2. **Resumable streams** (a refresh or reconnect mid-answer doesn't lose it) and persistent chat history.
3. Multi-tenant workspaces, roles, per-tenant data isolation (row-level security).
4. Usage metering (tokens per tenant) and Stripe billing; rate limits.
5. E2E tests with Playwright; deploy with preview environments per PR.

**Résumé bullet template:** "Built and deployed a multi-tenant AI SaaS (Next.js, Vercel AI SDK, FastAPI,
pgvector) with resumable streaming, RLS isolation and usage-based billing."

---

## 7. LLM Gateway & Cost Router  *(GenAI)*

**Why this project:** It builds on your "GenAI Playground (17+ LLMs)" experience and adds the cost/latency
engineering JDs ask for.

**Stack:** LiteLLM (or your own FastAPI gateway), Redis (exact + **semantic cache**), provider prompt
caching, a small classifier for routing, OpenTelemetry → Prometheus/Grafana, Docker/K8s, Terraform.

**Milestones**
1. Unified API with fallbacks, retries, circuit breaker, per-team budgets.
2. Semantic cache: measure hit rate against quality loss using the eval set from project 1.
3. **Cost-aware routing:** easy queries go to a small model, hard ones to a frontier model. Report
   savings against quality.
4. Dashboards: cost per request, TTFT, p95, cache hit rate.

**Résumé bullet template:** "Cut LLM cost per 1k requests by __% at <__% quality loss via semantic
caching + complexity-based model routing."

---

## 8. *(Optional)* Realtime Voice Agent  *(Full-stack)*

LiveKit Agents or Pipecat, streaming STT/TTS or a realtime speech API, MCP tools (calendar/CRM), and a
Next.js front end. Goal: under 800 ms turn latency, barge-in handling, transcript evals.
Do this only if you're targeting voice or customer-support product companies.

---

## 🏆 Capstone: Demand-Planning Copilot for CPG  *(all three roles)*

**Why this project:** It's **your domain**. You've built forecasting agents for PepsiCo and Unilever.
Rebuilding the idea publicly on open data gives you a portfolio piece that looks like your real work,
which you can walk through in any interview without NDA problems.

**Data:** The **M5 Forecasting (Walmart) dataset** (public on Kaggle), plus public holiday/promo calendars.

**Architecture**
- **Data/MCP layer (project 2):** MCP servers over sales, inventory and promotions in Postgres/DuckDB,
  exposed through your OAuth MCP gateway.
- **Agents (projects 3 & 4):** A LangGraph supervisor with specialists: *Data Retriever*,
  *Forecaster* (runs statistical/ML forecasts in the **sandbox** from project 5), *Analyst* (explains
  drivers), *Planner* (proposes order quantities). Planner actions need **human approval**.
  Long-term memory stores planner preferences.
- **Knowledge (project 1):** RAG over category playbooks, promo guidelines and meeting notes, with citations.
- **Quality:** Forecast accuracy (MAPE/WAPE against a statistical baseline), trajectory evals, RAG evals,
  red-team suite, all gated in CI.
- **Ops (project 7):** All LLM calls go through the gateway; Langfuse + OTel traces; cost per planning
  session.
- **UI (project 6):** Next.js + Vercel AI SDK with generative UI: forecast charts, scenario sliders
  ("what if promo +10%?"), an approval inbox, and a tool-call timeline.
- **Deploy:** Docker, Terraform on Azure or AWS, GitHub Actions, public demo with seeded data and a
  3-minute video.

**README must include:** architecture diagram, eval tables, cost per session, latency, threat model,
design trade-offs, and what failed.

**Résumé bullet template:** "Built an open-source multi-agent demand-planning copilot (LangGraph, MCP,
A2A, Next.js) on the M5 dataset; improved WAPE by __% over a baseline and kept cost at $__ per session
with CI-gated evals."

---

## Rules for every project

- **No demo without evals, tracing and a deployment.**
- README: problem → architecture diagram → how to run → **results table** → trade-offs → what's next.
- A 2–3 minute demo video.
- One LinkedIn post per project about a **finding** (e.g. "A reranker beat a 3× larger embedding
  model"), which also builds inbound recruiter interest.
