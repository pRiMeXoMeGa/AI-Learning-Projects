# 6. Non-Functional Design

## 6.1 Latency budget: chat time to first token (p95 ≤ 1.5 s)

| Step | Budget |
|---|---|
| proxy.ts session check (cookie + cached session) | ≤ 20 ms |
| Route handler: membership + quota + rate limit (Redis) | ≤ 40 ms |
| Load last N messages (Postgres, indexed) | ≤ 30 ms |
| Model: time to first token (streaming, prompt caching on) | ≤ 1,200 ms |
| Stream setup (Redis publish) | ≤ 20 ms |
| **Total** | **≤ ~1.3 s**, leaving headroom |

**Tool calls add time before the answer text:**
- `search_contracts` on the ai-service: ≤ 400 ms p95.
- The UI shows the tool part (skeleton → citations) immediately, so the user sees progress.

## 6.2 Rendering strategy

| Page | Strategy |
|---|---|
| Marketing, pricing | Static (cached) |
| Dashboard, document library | Server Components; lists with `"use cache"` and tags per org, invalidated by `revalidateTag` after mutations |
| Document viewer | Server-rendered shell + client viewer; page images lazy-loaded |
| Chat | Client component (`useChat`) inside a server-rendered layout; history loaded on the server |
| Register | Server-side pagination and filters; client table for sorting and review |

Cache keys **always include the org id**, and nothing tenant-specific is cached without it (checked in code review and tests).

## 6.3 Cost model (demo)

| Item | Estimate / month |
|---|---|
| Vercel (Hobby for the portfolio demo; Pro if needed) | $0–20 |
| Neon (free tier + branches) | $0 |
| Upstash Redis (free tier) | $0 |
| Vercel Blob | < $1 |
| ai-service on Azure Container Apps (scale to zero) | ≈ $5–10 |
| Model usage (demo capped by Free-plan credits) | ≤ $10 |
| **Total** | **≈ $10–20** |

**Note:** Vercel's Hobby plan is for non-commercial use. It fits a portfolio demo, but a real product would
need Pro.

## 6.4 Observability

- **`@vercel/otel`** in the Next.js app. AI SDK telemetry (`experimental_telemetry` or the v7
  equivalent) is on for every model call, and spans go to **Langfuse**, as in Projects 1–4.
- Span attributes: `org.id` (hashed), `user.id` (hashed), `chat.id`, `route`, `model`, `credits`.
- The ai-service uses the same OTel setup as Project 1, with `traceparent` passed from Next.js.
- **Product metrics** (from the database): weekly active orgs, documents ingested, playbook runs, review
  acceptance rate, credits consumed per org. They are shown on an internal `/admin/metrics` page (owner
  of the platform only).
- The Workflow SDK's run dashboard shows workflow steps and failures.

## 6.5 Failure modes

| Failure | Effect | Handling |
|---|---|---|
| Model provider down | Chat fails | Automatic fallback to the second provider for chat; playbooks retry with backoff |
| ai-service down | No search/ingestion | Chat answers "can't search right now"; workflows retry, then mark the step failed |
| Redis down | No resumable streams / rate limits | Chat streams without resume (degraded); rate limits fail **closed** for expensive calls |
| Neon cold start | Slow first request | Keep-alive ping on production; accept on previews |
| Stripe webhook delay | Plan change late | UI shows "processing"; the webhook handler re-reads state; a daily reconciliation job |
| Deploy mid-workflow | Run interrupted | Durable workflow resumes at the next step |

## 6.6 Scaling path

| Now | Later |
|---|---|
| Neon single region | Read replicas; partition `chunk` by org for large tenants |
| pgvector HNSW in the main DB | Dedicated vector store per large tenant if needed |
| One ai-service revision | Separate ingestion and query services; GPU parsing |
| Upstash free tier | Paid tier or self-hosted Redis |
| Per-org concurrency limits | Queue priorities by plan |

## 6.7 Testing strategy

| Layer | Tools |
|---|---|
| Unit (TS) | Vitest: tools, quota maths, risk-rule evaluator, filter DSL, RLS helper |
| Unit (Python) | pytest (Project 1 tests + new endpoints) |
| Component | React Testing Library for the register table and approval card |
| E2E | Playwright on previews, with mocked models |
| Isolation, billing | Dedicated suites (§4.5, §4.6) |
| Evals | CUAD + chat (§4.2, §4.3) |
| Performance | Lighthouse CI, k6 |
