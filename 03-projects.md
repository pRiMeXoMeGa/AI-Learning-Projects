# Project Recommendations (for a 4-YOE AI Engineer)

Each project is designed so that **together they cover every 🔴/🟠 item in
[02-tech-stack.md](02-tech-stack.md)**, and each one yields a résumé bullet with a *measured*
outcome, which is what senior JDs screen for.

Rule for every project: **no demo without evals, tracing, and a deploy.**

| # | Project | Primary skills | Level | Time |
|---|---|---|---|---|
| 1 | Production-grade Enterprise RAG | Hybrid search, re-ranking, pgvector/Qdrant, Ragas, Langfuse | Core | 3–4 wks |
| 2 | MCP Tool Server Suite | MCP servers/clients, OAuth, tool design | Core | 2 wks |
| 3 | Multi-Agent Workflow with LangGraph | LangGraph, HITL, checkpoints, A2A | Core | 3–4 wks |
| 4 | Eval & Guardrails Platform (CI-gated) | DeepEval/promptfoo, LLM-as-judge, red-teaming | Core | 2–3 wks |
| 5 | LLM Gateway & LLMOps Stack | LiteLLM, semantic cache, OTel, K8s, Terraform | Senior | 3 wks |
| 6 | Fine-tune → Quantize → Serve an SLM | LoRA/QLoRA, DPO, vLLM, benchmarking | Senior | 3–4 wks |
| 7 | Multimodal Document Intelligence | VLMs, Docling, structured extraction, HITL review | Senior | 2–3 wks |
| 8 | Sandboxed Coding / Data-Analyst Agent | Code execution sandbox, security, tool permissions | Senior | 2–3 wks |
| 9 | GraphRAG vs Vector RAG Study | Neo4j, entity extraction, comparative evals | Stretch | 2 wks |
| 10 | Real-time Voice Agent | STT/TTS, LiveKit/Pipecat, latency engineering | Stretch | 2 wks |
| 🏆 | **Capstone: Vertical AI Copilot (SaaS)** | Everything above, multi-tenant, full-stack | Portfolio | 6–8 wks |

---

## 1. Production-grade Enterprise RAG ("DocuMind")

**Problem:** Q&A over a messy corpus (e.g. 2k+ PDFs from SEC filings, RBI circulars, or your
company-style docs) with citations and access control.

**Stack:** Python, FastAPI (SSE streaming), Docling/Unstructured, Postgres + **pgvector** (then
swap to **Qdrant** to compare), BM25 (Postgres FTS or Elasticsearch), BGE/Cohere **reranker**,
OpenAI/Claude/Azure OpenAI, **Langfuse**, **Ragas**, Docker.

**Milestones**
1. Naive RAG baseline → build a 100-question **golden eval set** (with synthetic generation + manual review).
2. Measure faithfulness, answer relevance, context precision/recall with Ragas.
3. Add hybrid search → reranker → query rewriting → parent-document chunking; measure **each** step.
4. Citations with span highlighting; "I don't know" behaviour.
5. Row-level **ACL-aware retrieval** (metadata filters per user/group).
6. Incremental ingestion (only re-embed changed docs) via a queue (Celery/Redis).
7. Tracing + cost/latency dashboard in Langfuse.

**Senior signal:** An ablation table in the README showing which technique moved which metric.
**Résumé bullet:** "Raised RAG faithfulness 0.68→0.91 and cut p95 latency 45% via hybrid search + reranking + caching."

---

## 2. MCP Tool Server Suite

**Problem:** Expose real systems (Postgres, GitHub/Jira, your RAG from #1, a calendar) to any
agent (Claude Desktop, Cursor, your own agent) via **Model Context Protocol**.

**Stack:** MCP Python SDK (FastMCP) and/or TypeScript SDK, streamable HTTP transport, **OAuth 2.1**
auth, Docker, pytest.

**Milestones**
1. Read-only Postgres MCP server (tools + resources + prompts).
2. RAG-as-a-tool: wrap Project #1 as an MCP server.
3. Write-capable tools with confirmation/elicitation and **scoped permissions**.
4. Remote deployment with OAuth; rate limiting; audit logging of every tool call.
5. Build a minimal MCP **client** in your own agent loop (raw API, no framework) to deeply understand tool calling.
6. Tool-design study: compare tool descriptions/granularity and measure agent task success.

**Senior signal:** Security write-up — prompt injection via tool outputs, tool poisoning, least-privilege.

---

## 3. Multi-Agent Workflow with LangGraph ("OpsPilot")

**Problem:** An agentic workflow a business would pay for, e.g. **invoice-to-payment**, **incident
triage** (read alerts → query logs → propose fix → open ticket), or **research-to-report**.

**Stack:** **LangGraph** (supervisor + specialist sub-agents), Postgres checkpointer, MCP tools
from #2, human-in-the-loop interrupts, LangSmith/Langfuse, FastAPI, Streamlit or Next.js UI.

**Milestones**
1. Single ReAct agent with 3–5 tools; measure task success on 30 scripted scenarios.
2. Refactor into supervisor + specialist graph; add **structured state** and typed outputs.
3. **Human approval** before any write action; resume from checkpoint after crash/restart.
4. Short-term + long-term memory (per-user store).
5. Guard against loops (step budgets, cost budgets, timeouts).
6. Re-implement one agent with **OpenAI Agents SDK / Claude Agent SDK** and expose it over **A2A** — compare DX and reliability.

**Senior signal:** Trajectory evals (did the agent call the right tools in the right order?), not just final-answer evals.

---

## 4. Eval & Guardrails Platform (CI-gated)

**Problem:** Make "prompt changes break production" impossible.

**Stack:** **DeepEval** or **promptfoo**, Ragas, LLM-as-judge with calibrated rubrics, GitHub
Actions, **NeMo Guardrails / Llama Guard**, Presidio (PII), garak/PyRIT for red-teaming.

**Milestones**
1. Eval harness that runs against Projects #1 and #3 (datasets versioned in git).
2. **CI gate**: PR fails if faithfulness/task-success drops beyond a threshold.
3. Calibrate LLM-judge against 50 human labels; report agreement (Cohen's kappa).
4. Input/output guardrails: PII redaction, jailbreak & **prompt-injection detection**, topic restriction.
5. Automated red-team suite (indirect injection via retrieved docs & tool outputs).
6. Online eval: sample production traces → judge → dashboard.

**Résumé bullet:** "Built CI eval gates that caught 12 regressions pre-release; blocked 97% of injection attempts in red-team suite."

---

## 5. LLM Gateway & LLMOps Stack

**Problem:** A central platform every team's LLM traffic goes through.

**Stack:** **LiteLLM** (or build your own in FastAPI), Redis (exact + **semantic cache**),
multi-provider routing (OpenAI, Anthropic, Bedrock, Azure OpenAI, self-hosted vLLM),
**OpenTelemetry** → Prometheus/Grafana, per-team budgets, **Kubernetes** + Helm, **Terraform**
(AWS or Azure), GitHub Actions.

**Milestones**
1. Unified API with fallbacks, retries, timeouts, circuit breaker.
2. Semantic cache — measure hit rate vs. answer-quality degradation.
3. **Cost-aware router**: easy queries → small model, hard → frontier model (use a classifier); measure cost savings vs. quality.
4. Per-tenant keys, quotas, spend dashboards.
5. Deploy on managed K8s (EKS/AKS) via Terraform with autoscaling.

**Senior signal:** A cost report: "$ per 1k requests before/after routing + caching."

---

## 6. Fine-tune → Quantize → Serve a Small Language Model

**Problem:** Beat (or match) a frontier API on a narrow task at a fraction of the cost —
e.g. **text-to-SQL** on your schema, **ticket classification + extraction**, or a Hindi/Hinglish
support assistant.

**Stack:** Hugging Face Transformers/PEFT/**TRL**, **Unsloth**, **QLoRA**, synthetic data
generation with a frontier model, **DPO**, AWQ/GGUF quantization, **vLLM** on a cloud GPU
(or Ollama locally), MLflow/W&B.

**Milestones**
1. Baseline: frontier API + few-shot on a held-out test set.
2. Generate & clean a synthetic training set (dedupe, filter with a judge).
3. SFT with QLoRA on a 1–8B open-weight model → evaluate.
4. Preference tuning (DPO) on failure cases → evaluate.
5. Quantize, serve with vLLM (continuous batching), load-test (tokens/sec, TTFT, p95).
6. Cost/quality comparison table vs. API baseline.

**Résumé bullet:** "Fine-tuned 3B model matching frontier-API accuracy (94% vs 95%) at 1/15th the cost per request."

---

## 7. Multimodal Document Intelligence

**Problem:** Extract structured data from invoices, contracts, or medical forms (scans, tables,
handwriting) into a validated schema.

**Stack:** Vision-language models (GPT/Claude/Gemini vision, or Qwen-VL open-weight),
**Docling**/Azure Document Intelligence, Pydantic schemas, confidence scoring, human review UI
(Streamlit), Postgres.

**Milestones**
1. OCR-then-LLM vs direct VLM extraction — compare field-level accuracy.
2. Schema validation + auto-retry on validation errors.
3. Confidence-based routing to a **human review queue**; learn from corrections.
4. Batch processing with a job queue; throughput & cost metrics.

---

## 8. Sandboxed Coding / Data-Analyst Agent

**Problem:** "Upload a CSV / connect a DB, ask questions, get charts" — an agent that writes and
runs code safely.

**Stack:** Agent loop (Claude Agent SDK / OpenAI Agents SDK / LangGraph), **E2B** or Docker-based
sandbox, pandas/DuckDB, network & filesystem restrictions, artifact storage.

**Milestones**
1. Code-gen → execute → observe → fix loop.
2. Hard sandbox: no network, CPU/memory/time limits, read-only mounts.
3. Permission model for dangerous actions; full audit trail.
4. Benchmark on 50 analysis questions (correctness + number of iterations).

**Senior signal:** Threat model document for agentic code execution.

---

## 9. GraphRAG vs Vector RAG — a Comparative Study

**Stack:** Neo4j, LLM entity/relation extraction, Microsoft GraphRAG or LightRAG, the eval
harness from #4.

**Deliverable:** A blog post + repo showing *when* graph retrieval beats vector retrieval
(multi-hop, aggregation questions) and when it doesn't justify the cost. Great for LinkedIn
visibility.

---

## 10. Real-time Voice Agent

**Problem:** Appointment booking / customer-support phone agent.

**Stack:** LiveKit Agents or Pipecat, streaming STT (Whisper/Deepgram), realtime LLM or
STT→LLM→TTS pipeline, tools via MCP (calendar), telephony (Twilio SIP).

**Focus:** End-to-end latency (< 800 ms turn-taking), interruption handling, evals on call transcripts.

---

## 🏆 Capstone: Vertical AI Copilot (SaaS-style)

Combine everything into **one deployable product** in a domain you know from your 4 years
(fintech, healthcare, legal, e-commerce, HR, etc.). Example: **"Compliance Copilot"** for banks.

- Multi-tenant auth (OAuth/OIDC), per-tenant data isolation and ACL-aware RAG (#1)
- Agent workflows with approvals (#3) using MCP tools (#2)
- Document intake (#7), optional fine-tuned classifier (#6)
- All traffic via your gateway (#5), CI eval gates + guardrails (#4)
- Next.js + Vercel AI SDK front-end with streaming & generative UI
- Deployed on AWS or Azure with Terraform, K8s, GitHub Actions
- Public README: architecture diagram, eval results, cost per user, threat model, demo video

This single repo is what you walk through in a senior AI Engineer system-design interview.

---

## How to present these projects

- One repo per project (or a mono-repo) with: **architecture diagram, eval table, cost/latency numbers, trade-offs, what failed**.
- A 2–3 minute demo video for each.
- One LinkedIn post / blog per project, focused on a *finding* ("Reranking beat a bigger embedding model for 1/10th the cost").
- Put the measured outcomes on the résumé, using the JD keywords from [01-market-analysis.md](01-market-analysis.md).
