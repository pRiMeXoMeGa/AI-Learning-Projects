# 7. Architecture Decision Records

Format: **Context → Decision → Alternatives → Consequences.** Status is *Proposed* until validated while
building.

---

### ADR-001: Contract review on public CUAD data
- **Context:** The product needs a real B2B shape, public data and a way to measure quality.
- **Decision:** ClauseDesk, a contract-review SaaS, with CUAD contracts as the demo and eval data.
- **Alternatives:**
  - Policy Q&A over company handbooks (weaker structured-output story).
  - Invoices/receipts (well covered by OCR tools).
  - A generic "chat with PDFs" app (no differentiation).
- **Consequences:**
  - Extraction quality is measured against expert labels.
  - A legal disclaimer is required.
  - CUAD needs a small value-normalization layer for structured fields.

### ADR-002: Next.js 16 App Router with Server Components, Server Actions and Cache Components
- **Context:** Next.js is the biggest full-stack gap on the profile, and 16.x is current (Turbopack
  default, `"use cache"`, `proxy.ts`).
- **Decision:**
  - The App Router throughout.
  - Server Components for data pages.
  - Server Actions for mutations (each one checks auth).
  - Route handlers for streaming and webhooks.
  - `proxy.ts` for the session gate.
- **Alternatives:** Remix/React Router 7 (good, less demand in JDs); a Vite SPA + FastAPI (no server
  rendering story).
- **Consequences:** The key modern Next.js concepts are exercised. There are caching pitfalls to guard
  against (org in every cache key).

### ADR-003: AI SDK 7 for chat, tools, approvals and structured outputs
- **Decision:**
  - A `ToolLoopAgent` in the chat route.
  - `useChat` with typed message parts.
  - Structured outputs for extraction.
  - Tool approval for write tools.
  - Resumable streams with `resumable-stream` + Redis.
- **Alternatives:** Calling provider SDKs directly (more code, no UI integration); LangChain.js (less
  common in Next.js JDs).
- **Consequences:** Typed end-to-end streaming UI. v7 is recent (June 2026), so a few APIs may still
  shift; versions are pinned.

### ADR-004: Durable workflows with the Workflow SDK
- **Context:** Ingestion and playbook runs take minutes and must survive deploys and failures. Plain
  promises in serverless functions don't.
- **Decision:**
  - The Workflow SDK (`"use workflow"` / `"use step"`) for ingestion and playbook runs.
  - The managed runtime on Vercel.
  - Postgres World documented for self-hosting, since it needs a long-lived worker.
- **Alternatives:**
  - Inngest or Trigger.dev (mature, but an extra vendor).
  - Temporal (heavier; already optional in Project 3).
  - A queue plus cron (you'd rebuild retries and resume yourself).
- **Consequences:**
  - Durable steps with the same code locally and on Vercel.
  - Some coupling to the Workflow SDK. The self-host path exists but isn't serverless.

### ADR-005: Better Auth with the organization plugin
- **Context:** In 2026, Auth.js is in maintenance mode (maintained by the Better Auth team). A B2B SaaS
  needs organizations, roles and invitations.
- **Decision:** Better Auth, with email/password, Google OAuth, email verification and the organization
  plugin (roles, invitations, active organization on the session). Passkeys are optional.
- **Alternatives:** Clerk (fastest setup, hosted user data, cost at scale); Auth.js (maintenance only);
  Supabase Auth (ties the database choice).
- **Consequences:**
  - User data stays in our Postgres.
  - Auth pages must be built (with shadcn blocks).
  - Keeping it patched is our job.

### ADR-006: Tenant isolation with Postgres RLS plus app checks
- **Decision:**
  - `org_id` on every tenant table, with `FORCE ROW LEVEL SECURITY`.
  - A transaction-local `app.org_id`.
  - The app role has no `BYPASSRLS`.
  - The ai-service sets the same variable from a verified service token.
  - The app also checks membership and roles.
- **Alternatives:** Only app-level `where` clauses (one bug leaks data); a schema or database per tenant
  (strong, but heavy for a SaaS with many small tenants).
- **Consequences:** Defence in depth, testable with a dedicated suite. Every query needs a transaction.

### ADR-007: Drizzle ORM on Neon Postgres (pgvector), branch per PR
- **Decision:**
  - Drizzle for the TypeScript schema, migrations and typed queries.
  - Neon for serverless Postgres with **database branches for preview deployments**.
  - pgvector and full-text search, shared with the ai-service.
- **Alternatives:** Prisma (heavier client; RLS session variables are more awkward); Supabase (good,
  but its auth/RLS conventions overlap with Better Auth).
- **Consequences:** Real preview environments with their own data. The same database as Project 1's
  design (Postgres + pgvector).

### ADR-008: Keep document AI in Project 1's Python service
- **Context:** Parsing (Docling), hybrid retrieval and reranking already exist and are evaluated in Python.
- **Decision:** A FastAPI ai-service with internal endpoints, called by the Next.js tools and
  workflows with a short-lived service JWT.
- **Alternatives:** Rewrite retrieval in TypeScript (duplicate work, and the Project 1 eval evidence
  would be lost).
- **Consequences:**
  - A realistic polyglot architecture: a TypeScript product plus a Python AI service.
  - Two deployables.
  - The service boundary must enforce tenancy (ADR-006).

### ADR-009: Stripe subscriptions plus usage meters, with our own credit ledger
- **Context:** In 2026, AI SaaS pricing is usually hybrid (a subscription with included credits, then
  usage). Stripe supports meters, credits and even LLM-token billing.
- **Decision:**
  - Record usage in our own `usage_event` ledger from provider token counts.
  - Enforce quotas ourselves before each call.
  - Report batched meter events to Stripe with idempotency keys.
  - Stripe Checkout and Customer Portal for plans.
- **Alternatives:** Stripe's LLM-proxy/gateway-based token billing (less code, but usage is metered
  outside the app, and quotas are harder to enforce up front); flat subscriptions only (breaks under
  spiky AI use).
- **Consequences:** Accurate, explainable usage pages and pre-call enforcement. Reconciliation is our
  job (tested in §4.6).

### ADR-010: Vercel for the web app, Azure Container Apps for the ai-service
- **Decision:** Vercel (preview deployments, Workflow runtime, Blob). The ai-service stays on Azure
  Container Apps with Projects 1–4's Terraform. Previews share a staging ai-service that selects the
  preview's Neon branch from the service token.
- **Alternatives:**
  - Everything on Azure (the Next.js standalone build + Postgres World worker loses previews and the managed workflow runtime).
  - Everything on Vercel (the Python service with Docling is heavy for functions).
- **Consequences:**
  - Two clouds, which is common in practice.
  - Staging isolation for previews is weaker (a residual risk).
  - If branch switching in the ai-service proves awkward, previews use a shared staging database.

### ADR-011: AI Elements + shadcn/ui + Tailwind for the UI
- **Decision:** AI Elements (shadcn-based chat components owned in the repo) for chat, tool parts and
  approvals. shadcn/ui for everything else. TanStack Table for the register.
- **Alternatives:** A custom chat UI from scratch (slower); a component library with its own theme
  system (MUI; heavier, less common with Next.js + AI SDK).
- **Consequences:** A fast, consistent, accessible UI whose code can be changed.

### ADR-012: Mocked models in E2E, real models in evals
- **Decision:** Playwright runs against previews with recorded model streams (AI SDK mock providers).
  Real models run only in the eval suites and a nightly smoke test.
- **Consequences:** E2E is fast, free and deterministic. Quality is measured separately, where it can be
  measured properly.

### ADR-013: Two model providers, no gateway yet
- **Decision:** The Anthropic and OpenAI provider packages directly. The model picker in chat falls back
  to the other provider on error.
- **Alternatives:** Vercel AI Gateway or LiteLLM now (that's Project 7's topic, where it's measured).
- **Consequences:** Project 7 can later put its gateway in front with a one-line provider change, which
  makes a nice before/after.
