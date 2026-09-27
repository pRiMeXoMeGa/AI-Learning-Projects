# 11. Glossary

Plain-English definitions of the terms used in these docs, and where each one shows up in this project.
RAG and evaluation terms are in the [Project 1 glossary](../../01-rag-eval-lab/docs/10-glossary.md).

## Next.js and React

| Term | Meaning | In this project |
|---|---|---|
| **App Router** | Next.js routing based on the `app/` folder, with layouts and nested routes | All pages |
| **Server Component** | A React component that runs only on the server and can read data directly | Dashboard, library, register |
| **Client Component** | A component that runs in the browser (`"use client"`) for interactivity | Chat, viewer, table |
| **Server Action** | A server function called from a form or button; it's a public HTTP endpoint under the hood | All mutations (each checks auth) |
| **Route handler** | An API endpoint in `app/api/…/route.ts` | Chat stream, webhooks, upload |
| **`proxy.ts`** | Next.js 16's request interceptor (formerly middleware) | Session gate, org routing, headers |
| **Cache Components / `"use cache"`** | Next.js 16's explicit caching for components and functions | Library lists (org in the key) |
| **`revalidateTag`** | Invalidates cached data with a given tag | After uploads and edits |
| **`after()`** | Runs work after a response is sent | Keeps generating a resumable stream |
| **Turbopack** | The default Next.js bundler in 16 | Dev and build |
| **Preview deployment** | A full deployment of a pull request at its own URL | E2E tests on every PR |

## AI SDK

| Term | Meaning | In this project |
|---|---|---|
| **AI SDK 7** | Vercel's TypeScript toolkit for LLM apps (June 2026 release) | Chat, extraction |
| **`ToolLoopAgent`** | An AI SDK agent that calls tools in a loop until done | Chat agent |
| **`useChat`** | React hook that manages a streaming chat | Chat page |
| **UI message parts** | Typed pieces of a message (text, tool call, data) that render as components | Citation cards, register tables |
| **Tool approval** | A tool call waits for the user to approve or reject it | Register edits |
| **Structured output** | The model returns JSON matching a schema | Extraction |
| **Resumable stream** | A stream stored in Redis so a reconnecting client can pick it up | Refresh mid-answer |
| **AI Elements** | shadcn-based React components for AI chat UIs | Chat UI |
| **Mock provider** | A fake model that replays recorded output, for tests | E2E |

## SaaS and data

| Term | Meaning | In this project |
|---|---|---|
| **Multi-tenant** | One app serving many customers, each isolated | Organizations |
| **Organization / tenant** | A customer workspace with members | `organization` table |
| **RBAC** | Role-based access control | owner, admin, member, viewer |
| **RLS** (row-level security) | Postgres policies that filter rows per session | Every tenant table |
| **Better Auth** | Open-source TypeScript auth library | Sign-in, orgs, invitations |
| **Drizzle ORM** | Typed SQL toolkit for TypeScript | Schema, queries, migrations |
| **Neon branch** | A copy-on-write copy of a Postgres database | One per preview |
| **Durable workflow** | Code whose progress is saved step by step, so it survives crashes and deploys | Ingestion, playbook runs |
| **Workflow SDK** | Vercel's open-source durable workflow library (`"use workflow"`, `"use step"`) | Workflows |
| **Idempotency key** | An ID that makes a repeated request safe | Usage events, Stripe meter events |
| **Usage meter** | Stripe object that aggregates usage events into billable quantities | AI credits |
| **Credits** | The app's unit of usage (from tokens and pages) | Quotas and billing |
| **Webhook** | A server-to-server callback when something happens | Stripe events |
| **Core Web Vitals** | Google's page-experience metrics: LCP, INP, CLS | Performance budgets |

## Contract domain

| Term | Meaning | In this project |
|---|---|---|
| **CUAD** | Contract Understanding Atticus Dataset: 510 contracts, 41 clause types, expert labels | Demo + evals |
| **Clause** | A provision in a contract (e.g. governing law, cap on liability) | Clause types |
| **Playbook** | A reusable set of clause questions with schemas and risk rules | "Vendor review" |
| **Clause register** | A table of extracted clause values across contracts | Register page |
| **Auto-renewal** | A contract renews unless someone gives notice in time | Risk rule example |
| **Cap on liability** | A limit on how much one party must pay if things go wrong | Risk rule example |
