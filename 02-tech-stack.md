# Tech Stack for GenAI Engineer · Agent Engineer · AI Full-stack Engineer

**Columns**
- **G / A / F**: how important the item is for GenAI Engineer / Agent Engineer / AI Full-stack Engineer JDs.
  🔴 Must · 🟠 Should · 🟢 Nice · — not relevant.
- **You**: status based on your résumé.
  ✅ shown on résumé · ⚠️ partial or not shown publicly · ❌ gap to learn.

The ❌ and ⚠️ rows with 🔴 or 🟠 ratings are your learning list. Each one maps to a project in
[03-projects.md](03-projects.md).

---

## 1. Languages & foundations
| Tech | G | A | F | You | Notes |
|---|---|---|---|---|---|
| Python (async, typing, `uv`) | 🔴 | 🔴 | 🔴 | ✅ | |
| Pydantic v2, structured outputs | 🔴 | 🔴 | 🔴 | ✅ | Strong: extraction pipelines with confidence scores |
| TypeScript | 🟢 | 🟠 | 🔴 | ✅ | |
| SQL / Postgres | 🔴 | 🟠 | 🔴 | ✅ | |
| pytest, Jest, Cypress, TDD | 🟠 | 🟠 | 🟠 | ✅ | |

## 2. LLM platforms & models
| Tech | G | A | F | You | Notes |
|---|---|---|---|---|---|
| OpenAI, Anthropic Claude, Gemini APIs (tool use, streaming, batch, prompt caching) | 🔴 | 🔴 | 🔴 | ⚠️ | Used via the Playground; add prompt caching & batch APIs |
| Azure OpenAI / AI Foundry | 🔴 | 🟠 | 🟠 | ✅ | |
| AWS Bedrock (incl. Bedrock Agents / AgentCore) | 🔴 | 🟠 | 🟠 | ⚠️ | Know Bedrock models; add its agent services |
| Google Vertex AI / Gemini | 🟢 | 🟢 | 🟢 | ❌ | Low priority |
| Embedding models & **rerankers** (Cohere, BGE, Voyage) | 🔴 | 🟠 | 🟢 | ⚠️ | Rerankers not on résumé |
| Open-weight models via Ollama/vLLM | 🟢 | 🟢 | 🟢 | ❌ | Optional; useful for local dev and cost comparisons |

## 3. RAG & retrieval
| Tech | G | A | F | You | Notes |
|---|---|---|---|---|---|
| Chunking, metadata, hybrid search | 🔴 | 🟠 | 🟠 | ✅ | |
| Re-ranking, query rewriting, HyDE, parent-doc retrieval | 🔴 | 🟠 | 🟢 | ❌ | |
| Vector DBs: Pinecone, pgvector, Qdrant, Azure AI Search | 🔴 | 🟠 | 🔴 | ✅ | Add pgvector publicly (full-stack default) |
| Agentic RAG (retrieval as a tool, iterative retrieval) | 🟠 | 🔴 | 🟢 | ⚠️ | |
| ACL-aware / multi-tenant retrieval | 🟠 | 🟠 | 🟠 | ❌ | |
| Document parsing: Azure Document Intelligence, Docling | 🟠 | 🟢 | 🟢 | ✅ | |
| GraphRAG (Neo4j, LightRAG) | 🟢 | 🟢 | — | ❌ | Optional |

## 4. Agents & orchestration
| Tech | G | A | F | You | Notes |
|---|---|---|---|---|---|
| **LangGraph** (state, routing, subgraphs) | 🟠 | 🔴 | 🟠 | ✅ | Strong |
| LangGraph persistence: checkpointers, interrupts, **long-term memory store** | 🟠 | 🔴 | 🟠 | ⚠️ | HITL on résumé; make memory & time-travel visible |
| **MCP servers** (FastMCP) | 🟠 | 🔴 | 🟠 | ✅ | Strong and rare |
| **Remote MCP**: streamable HTTP, **OAuth 2.1**, elicitation, sampling, MCP clients | 🟢 | 🔴 | 🟠 | ❌ | Next step up from your current MCP work |
| **A2A protocol** | 🟢 | 🟠 | — | ❌ | |
| **OpenAI Agents SDK**, **Claude Agent SDK**, **Google ADK** | 🟢 | 🔴 | 🟠 | ❌ | JDs ask for framework breadth |
| AutoGen / Microsoft Agent Framework, CrewAI, Semantic Kernel | 🟢 | 🟠 | — | ⚠️ | AutoGen on résumé |
| Durable execution: Temporal (or LangGraph Platform) | 🟢 | 🟠 | 🟢 | ❌ | You've used similar patterns ("Funnel Automation") |
| **Sandboxed code execution** (E2B, Docker, gVisor) | 🟢 | 🔴 | 🟠 | ❌ | Your InfoSec background is an advantage here |
| Browser/computer-use agents (Playwright, Browser Use) | — | 🟠 | 🟢 | ❌ | |

## 5. Evaluation, safety & observability  ← your biggest gap
| Tech | G | A | F | You | Notes |
|---|---|---|---|---|---|
| **Ragas / DeepEval / promptfoo** | 🔴 | 🔴 | 🟠 | ❌ | |
| LLM-as-judge (calibrated), golden datasets | 🔴 | 🔴 | 🟠 | ❌ | |
| **Trajectory / tool-use evals** for agents | 🟠 | 🔴 | 🟢 | ❌ | |
| **Eval gates in CI** (GitHub Actions) | 🔴 | 🔴 | 🟠 | ❌ | |
| **Langfuse / LangSmith / Arize Phoenix** tracing | 🔴 | 🔴 | 🟠 | ❌ | |
| OpenTelemetry (GenAI semantic conventions) | 🟠 | 🟠 | 🟠 | ❌ | |
| Guardrails: PII, toxicity, prompt-injection, Content Safety | 🔴 | 🔴 | 🟠 | ✅ | Strong |
| Red-teaming (promptfoo redteam, garak, PyRIT) | 🟠 | 🟠 | 🟢 | ⚠️ | You harden against attacks but don't run automated red-team suites yet |
| Prompt versioning & management | 🟠 | 🟠 | 🟢 | ✅ | Semantic-versioned prompts |

## 6. Cost & performance engineering
| Tech | G | A | F | You | Notes |
|---|---|---|---|---|---|
| Provider prompt caching, Redis exact & **semantic cache** | 🟠 | 🟠 | 🟠 | ⚠️ | Redis known; semantic cache not shown |
| Model routing / fallbacks (LiteLLM, Portkey) | 🟠 | 🟠 | 🟢 | ⚠️ | 17-LLM Playground is close; add routing + cost metrics |
| Token/cost budgets, latency (TTFT, p95) measurement | 🔴 | 🔴 | 🟠 | ❌ | Not quantified on résumé |

## 7. Backend & infrastructure
| Tech | G | A | F | You | Notes |
|---|---|---|---|---|---|
| FastAPI (SSE streaming, background tasks, WebSockets) | 🔴 | 🔴 | 🔴 | ✅ | |
| Node.js backend / tRPC / Next.js route handlers | — | 🟢 | 🟠 | ⚠️ | |
| Kafka / RabbitMQ / Celery / Redis queues | 🟠 | 🟠 | 🟠 | ✅ | |
| OAuth2/OIDC, JWT, secrets (Key Vault) | 🟠 | 🔴 | 🔴 | ✅ | |
| Docker, Kubernetes, Terraform | 🟠 | 🟠 | 🟠 | ✅ | |
| GitHub Actions / GitLab CI | 🔴 | 🔴 | 🔴 | ✅ | |
| Databricks / Snowflake | 🟠 | 🟢 | — | ❌ | Common in CPG/enterprise GenAI JDs; optional |

## 8. Frontend & product (AI Full-stack)
| Tech | G | A | F | You | Notes |
|---|---|---|---|---|---|
| React, Redux, Tailwind | 🟢 | 🟢 | 🔴 | ✅ | |
| **Next.js (App Router, Server Components, Server Actions)** | — | 🟢 | 🔴 | ❌ | Biggest full-stack gap |
| **Vercel AI SDK** (useChat, streamText, tool UIs, generative UI) | — | 🟢 | 🔴 | ❌ | |
| Agent UX: tool-call timelines, approvals, citations, artifacts | 🟢 | 🟠 | 🔴 | ⚠️ | You've built conversational UIs; add agent-specific UX |
| shadcn/ui, TanStack Query, Zustand | — | — | 🟠 | ❌ | Quick to learn with your React background |
| Auth.js / Clerk, Stripe usage billing | — | — | 🟠 | ❌ | |
| Realtime voice (LiveKit / Pipecat / Realtime APIs) | 🟢 | 🟢 | 🟠 | ❌ | Optional |

## 9. System-design topics for interviews
- **GenAI Engineer:** RAG at scale (ingestion, freshness, ACLs, multi-tenancy), an eval pipeline, cost modelling
- **Agent Engineer:** an agent platform (tool registry, MCP gateway, permissions, state, HITL, observability),
  failure modes (loops, tool misuse, injection), trajectory evals
- **AI Full-stack Engineer:** a streaming chat/agent product end to end (auth, persistence, resumable streams,
  rate limits, billing), generative UI architecture
