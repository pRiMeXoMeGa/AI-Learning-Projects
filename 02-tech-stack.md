# The 2026 AI / GenAI Engineering Tech Stack

Priority legend (based on JD frequency in [01-market-analysis.md](01-market-analysis.md)):

- 🔴 **Must** — shows up in most GenAI/AI Engineer JDs; missing it gets you filtered out
- 🟠 **Should** — common in senior JDs; a strong differentiator
- 🟢 **Nice** — niche or role-specific; learn when a project needs it

---

## 1. Languages & engineering foundations
| Tech | Priority | Notes |
|---|---|---|
| Python (async, typing, packaging with `uv`) | 🔴 | ~71% of postings. Know `asyncio`, generators, type hints |
| Pydantic v2 | 🔴 | Structured outputs, validation, settings |
| SQL (Postgres) | 🔴 | ~17% of postings; also your vector store via pgvector |
| TypeScript | 🟠 | Full-stack AI roles, Vercel AI SDK, MCP servers in TS |
| Git, testing (pytest), clean architecture | 🔴 | Expected at 4+ YOE |
| Go / Rust | 🟢 | Platform/inference-infra roles only |

## 2. LLM providers & model families
| Tech | Priority | Notes |
|---|---|---|
| OpenAI API, Anthropic Claude API, Google Gemini API | 🔴 | Tool calling, structured outputs, streaming, batch APIs, prompt caching |
| Azure OpenAI / **Azure AI Foundry** | 🔴 (India/enterprise) | Very common in Indian GCC & consulting JDs |
| **AWS Bedrock** | 🔴 | AWS ~33% of postings |
| Google **Vertex AI** | 🟠 | |
| Open-weight models: Llama, Qwen, Mistral, DeepSeek, Gemma | 🟠 | Self-hosting, fine-tuning, cost control |
| Embedding models (OpenAI, Cohere, Voyage, BGE, E5, Jina) | 🔴 | Know how to pick & evaluate them |
| Rerankers (Cohere Rerank, BGE-reranker, cross-encoders) | 🟠 | Biggest cheap quality win in RAG |

## 3. Prompt & context engineering
| Tech | Priority | Notes |
|---|---|---|
| Prompt patterns: few-shot, CoT, ReAct, self-critique | 🔴 | Table stakes |
| **Structured outputs** / JSON schema, function calling | 🔴 | Every JD mentions tool/function calling |
| Context engineering (context window budgeting, compaction, memory) | 🟠 | The 2026 evolution of "prompt engineering" |
| DSPy (programmatic prompt optimisation) | 🟢 | Differentiator for research-leaning roles |
| Instructor / Outlines | 🟢 | Constrained generation |

## 4. RAG & retrieval
| Tech | Priority | Notes |
|---|---|---|
| RAG pipeline design: chunking, metadata, query rewriting, HyDE, citations | 🔴 | ~in every GenAI JD |
| **Hybrid search** (BM25 + dense) + re-ranking | 🔴 | Explicitly called out in senior JDs |
| Vector DBs: **pgvector**, **Qdrant**, Pinecone, Weaviate, Milvus, FAISS | 🔴 | Know at least pgvector + one managed DB |
| Elasticsearch / OpenSearch / Azure AI Search | 🟠 | Enterprise search integration |
| LlamaIndex | 🟠 | ~4% of postings; strong for ingestion/RAG |
| Document parsing: Docling, Unstructured, LlamaParse, Azure Document Intelligence | 🟠 | Real-world PDFs/tables/scans |
| **GraphRAG** / knowledge graphs (Neo4j) | 🟢 | Growing; use when relations matter |
| Agentic RAG (retrieval as a tool, iterative retrieval) | 🟠 | |

## 5. Agents & orchestration
| Tech | Priority | Notes |
|---|---|---|
| **LangGraph** | 🔴 | The dominant orchestration ask in 2026 JDs |
| LangChain (core abstractions) | 🔴 | ~11% of postings; know it, don't depend on it blindly |
| **MCP (Model Context Protocol)** — servers & clients | 🔴 | "Universal" agent↔tool protocol; explicit in senior JDs |
| **A2A (Agent2Agent) protocol** | 🟠 | Agent↔agent interop, emerging fast |
| OpenAI Agents SDK, Claude Agent SDK, Google ADK | 🟠 | Vendor-native agent frameworks |
| CrewAI, AutoGen / Microsoft Agent Framework, Semantic Kernel | 🟠 | Semantic Kernel common in Azure shops |
| Pydantic AI, smolagents | 🟢 | |
| Agent patterns: planner-executor, supervisor, human-in-the-loop, reflection | 🔴 | |
| Durable execution: Temporal, LangGraph checkpointers | 🟠 | Long-running agents |
| Sandboxed code execution: E2B, Docker, Firecracker | 🟠 | "Sandbox security" in agent JDs |
| Browser/computer-use agents (Playwright, Browser Use) | 🟢 | |

## 6. Evaluation, safety & guardrails
| Tech | Priority | Notes |
|---|---|---|
| **Eval frameworks**: Ragas, DeepEval, promptfoo, OpenAI Evals, Inspect | 🔴 | "The 2026 differentiator" |
| LLM-as-judge, golden datasets, pairwise comparison, eval in CI | 🔴 | |
| Online evals / A/B tests / user feedback loops | 🟠 | |
| Guardrails: NeMo Guardrails, Guardrails AI, Llama Guard, Presidio (PII) | 🟠 | |
| Red-teaming & **prompt-injection defense** (garak, PyRIT, promptfoo redteam) | 🟠 | |
| Responsible AI / governance (EU AI Act, NIST AI RMF, ISO 42001) | 🟢 | Enterprise/regulated industries |

## 7. Observability & LLMOps
| Tech | Priority | Notes |
|---|---|---|
| **Langfuse** / **LangSmith** / Arize Phoenix / W&B Weave | 🔴 | Tracing, prompt versioning, cost tracking |
| **OpenTelemetry** (GenAI semantic conventions) | 🟠 | Called out in staff-level JDs |
| LLM gateway: **LiteLLM**, Portkey, Kong AI gateway | 🟠 | Routing, fallbacks, budgets |
| Caching: Redis (exact + semantic cache), provider prompt caching | 🟠 | Cost/latency |
| Prometheus + Grafana | 🟠 | |
| MLflow (tracking, model registry, LLM evals) | 🟠 | Especially in Databricks shops |

## 8. Model customisation & inference
| Tech | Priority | Notes |
|---|---|---|
| PyTorch | 🟠 | ~28% mention deep learning; PyTorch is the default |
| Hugging Face Transformers, Datasets, PEFT, TRL | 🟠 | |
| **LoRA / QLoRA**, SFT, **DPO**/GRPO | 🟠 | Fine-tuning asks are common in LLM-engineer JDs |
| Unsloth, Axolotl, LLaMA-Factory | 🟢 | Fast fine-tuning |
| Quantization: AWQ, GPTQ, GGUF, FP8 | 🟠 | |
| **vLLM**, SGLang, TGI, NVIDIA Triton / TensorRT-LLM, Ollama | 🟠 | "Inference optimisation" is a top-7 skill |
| Distillation & synthetic data generation | 🟢 | |
| Ray (Serve/Train), GPU basics (CUDA memory, KV cache, batching) | 🟢 | |

## 9. Backend, data & deployment
| Tech | Priority | Notes |
|---|---|---|
| **FastAPI** (streaming SSE/WebSockets, background tasks) | 🔴 | Most-requested backend framework in AI JDs |
| **Docker** | 🔴 | |
| **Kubernetes** (+ Helm, KServe for models) | 🟠 | Explicit in many enterprise JDs |
| CI/CD: GitHub Actions / Azure DevOps | 🔴 | |
| Terraform / IaC | 🟠 | |
| Message queues / event-driven: Kafka, Celery, SQS, Redis Streams | 🟠 | Async ingestion & long agent jobs |
| Data platforms: **Databricks**, Snowflake (Cortex), Spark | 🟠 | Very common in enterprise GenAI JDs |
| Auth: OAuth2/OIDC, Entra ID, IAM, private endpoints | 🟠 | Required for MCP auth & enterprise deploys |

## 10. Front-end & product surface
| Tech | Priority | Notes |
|---|---|---|
| Streamlit / Gradio / Chainlit | 🟠 | Fast demos, internal tools |
| Next.js/React + **Vercel AI SDK** | 🟠 | Full-stack AI roles, streaming & generative UI |

## 11. Multimodal & voice
| Tech | Priority | Notes |
|---|---|---|
| Vision-language models for document/visual QA | 🟠 | Document intelligence is a huge enterprise use case |
| Speech: Whisper/STT, TTS, realtime voice APIs | 🟢 | |
| Voice agent frameworks: LiveKit Agents, Pipecat | 🟢 | Fast-growing niche (support, sales) |
| Image/video generation (diffusion, ComfyUI) | 🟢 | Only for media-focused roles |

## 12. System design knowledge (interviews)
- Designing a RAG system at scale (ingestion, freshness, ACL-aware retrieval, multi-tenancy)
- Designing an agent platform (tool registry, permissions, state, observability, HITL)
- Cost modelling: tokens × price × QPS, caching hit rates, routing to small models
- Latency budgets: TTFT, streaming, parallel tool calls, speculative decoding
- Failure modes: hallucination, prompt injection, tool misuse, runaway loops, data leakage
