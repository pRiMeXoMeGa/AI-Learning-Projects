# Profile vs. Market: Gap Analysis

**Current profile:** Senior GenAI & Full-Stack Engineer, about 5 years (Fractal, LTIMindtree, Analyze Infotech, TCS).
Enterprise clients include PepsiCo and Unilever. M.Tech in Information Security.

**Target roles**
1. GenAI / Applied AI Engineer
2. Agent Engineer
3. AI Full-stack Engineer

---

## 1. What you already have (strong match with 2026 JDs)

| JD requirement | Evidence on your résumé | Strength |
|---|---|---|
| LangGraph multi-agent orchestration | Supervisor–worker graphs, TypedDict state, conditional routing (Fractal) | ✅ Strong |
| MCP servers | FastAPI + FastMCP tool servers over forecasting DBs (PepsiCo) | ✅ Strong, and rare in the market |
| Workflow orchestration, retries, state hand-offs | "Funnel Automation", +50% prediction accuracy (Unilever) | ✅ Strong |
| Structured outputs / Pydantic validation | Extraction pipelines with per-field confidence scores | ✅ Strong |
| RAG | Pinecone + Postgres, chunking, hybrid retrieval, Azure AI Search | ✅ Good |
| Guardrails / Responsible AI | PII tokenisation, toxicity, prompt-injection shielding, Content Safety | ✅ Strong (Information Security M.Tech helps) |
| Human-in-the-loop, checkpoints | HITL wait states, state persistence (LTIMindtree) | ✅ Good |
| Multi-LLM platform | GenAI Playground with 17+ LLMs (Azure OpenAI, Claude, Bedrock) | ✅ Strong |
| Cloud | Azure (Foundry, Functions, Key Vault), AWS Bedrock/SageMaker, Terraform | ✅ Good |
| Backend | FastAPI, Flask, OAuth/JWT, Redis, Kafka, RabbitMQ | ✅ Strong |
| Frontend | React, TypeScript, Redux, Tailwind, WebSocket; conversational UIs | ✅ Strong |
| Leadership | Led a team of 5 engineers; set API contract standards | ✅ Senior signal |

**Positioning:** You already look like an **Agent Engineer with full-stack depth**, which is an uncommon
combination. You don't need beginner projects. You need projects that show **the proof the 2026 market
screens for** and that currently isn't visible in your profile.

## 2. Gaps: what JDs ask for that your résumé doesn't show

| Gap | Why it matters | Roles affected | Priority |
|---|---|---|---|
| **Evals** (Ragas/DeepEval/promptfoo, LLM-as-judge, CI eval gates, trajectory evals) | Called "the 2026 differentiator"; your résumé mentions "prompt testing" but no eval framework or metrics | All three | 🔴 Critical |
| **LLM observability** (Langfuse/LangSmith/Phoenix, OpenTelemetry) | In most senior JDs; not mentioned anywhere on your résumé | All three | 🔴 Critical |
| **Public proof of work** | All your work is client/NDA work, so recruiters can't verify it. Your GitHub needs 3–4 polished repos | All three | 🔴 Critical |
| **Measured RAG quality** (reranking, retrieval metrics, ablations) | You list hybrid retrieval but no reranker and no quality numbers | GenAI | 🟠 High |
| **Next.js + Vercel AI SDK, streaming & generative UI** | Standard stack for AI full-stack roles; you have React/Angular but not Next.js | Full-stack | 🟠 High |
| **Vendor agent SDKs** (OpenAI Agents SDK, Claude Agent SDK, Google ADK) and **A2A** | Agent roles increasingly ask for framework breadth and agent-to-agent interop | Agent | 🟠 High |
| **Remote MCP with OAuth 2.1**, MCP clients, tool-design evaluation | Builds on your MCP strength; takes you from "used MCP" to "MCP expert" | Agent | 🟠 High |
| **Agent sandboxing / code execution** (E2B, Docker), computer/browser-use | "Sandbox security" is an explicit agent-JD requirement; fits your security background | Agent | 🟠 High |
| **Long-term memory & durable execution** (LangGraph store, Temporal) | Long-running agents are a big 2026 theme | Agent | 🟡 Medium |
| **Cost/latency engineering** (semantic cache, prompt caching, model routing, LiteLLM) | "Reason about cost like a senior backend engineer" | GenAI, Full-stack | 🟡 Medium |
| **Realtime / voice agents** | Fast-growing niche for full-stack AI | Full-stack | 🟢 Optional |
| Fine-tuning / serving (LoRA, vLLM) | Rarely required for your 3 target roles | — | 🟢 Optional |

## 3. Résumé fixes (do these before applying)

1. **Typos** (ATS and recruiters notice them): "Pratices" → Practices, "Generative Al" → Generative AI,
   "deisgn" → design, "WebScoket" → WebSocket, "Cretificate" → Certificates, "Techonology" → Technology.
2. **Timeline issues:** The LTIMindtree role (06/2022–06/2024) mentions **GPT-4.5**, which was released in
   2025, and **Azure AI Foundry**, which was called Azure AI Studio until late 2024. An interviewer may ask
   about this. Use the model and product names that existed at the time (e.g. GPT-4 / GPT-4o,
   "Azure AI Studio (now AI Foundry)").
3. **Skills section:** Move Terraform from "LLM" to "Cloud & DevOps". Add a dedicated
   **"Agents & LLMOps"** line (LangGraph, MCP, A2A, evals, Langfuse, OpenTelemetry) as you learn them.
   Remove Angular Level-2 / Bootstrap / Netlify, which dilute an AI profile.
4. **Metrics:** Only 1 of about 17 AI bullets has a number (+50%). Add latency, cost, accuracy, volume or
   adoption figures where you can, e.g. "served N users / N requests per day", "reduced manual review by X%".
5. **Headline:** Pick one headline per application:
   - *GenAI Engineer:* "Senior GenAI Engineer — RAG, Agents, Evals | Azure & AWS"
   - *Agent Engineer:* "Agent Engineer — LangGraph, MCP, Multi-agent Systems"
   - *AI Full-stack:* "AI Full-stack Engineer — FastAPI, React/Next.js, LLM Apps"
6. **Links:** Make sure the Portfolio / LinkedIn / GitHub links go to the pinned project repos below.
7. **Certification:** AZ-900 is entry level. **AI-102 (Azure AI Engineer Associate)** matches your
   Azure experience and is common in Indian GCC and Middle-East enterprise JDs.

## 4. The plan in one line

**Turn your hidden client work into public, measured, open-source proof:** each project in
[03-projects.md](03-projects.md) fills one or more of the gaps above and is mapped to a target role.
