# 2. High-Level Architecture

## 2.1 Design principles

1. **The server renders; the client streams.** Pages are Server Components that read data directly.
   Client components exist only where interaction is needed (chat, viewer highlights, tables).
2. **Tenant isolation lives in the database.** RLS is the last line of defence, so a missed `where
   org_id = …` in app code still can't leak data.
3. **Anything longer than a request is a workflow.** Ingestion and playbook runs are durable workflows
   with steps, retries and resume. They never run as a fire-and-forget promise inside a request.
4. **Check before you spend.** Quotas and rate limits are checked before any model call or workflow
   start, and usage is recorded from the provider's own token counts.
5. **Structured over free text.** Extraction uses schemas, tool results are typed, and the UI renders
   typed parts.
6. **Python where it's strongest, TypeScript where the user is.** Document AI (parsing, retrieval)
   stays in Project 1's Python service. Product logic, agent orchestration and UI live in Next.js.
7. **Every PR is a working copy of production:** a preview deployment, a database branch and an E2E run.

## 2.2 System context (C4 level 1)

```mermaid
flowchart TB
    USER(["👩‍💼 Reviewer / owner / viewer"])
    DEV(["👩‍💻 Developer"])
    subgraph SYS["ClauseDesk"]
        S["Next.js app + workflows + AI service"]
    end
    STRIPE["Stripe (test mode)"]
    LLM["Anthropic · OpenAI"]
    GOOG["Google OAuth"]
    MAIL["Email (Resend)"]
    LF["Langfuse"]
    GH["GitHub (PRs → previews)"]
    USER -->|"HTTPS"| S
    S -->|"checkout · portal · meter events"| STRIPE
    STRIPE -->|"webhooks"| S
    S --> LLM
    S --> GOOG
    S --> MAIL
    S -.->|"OTLP"| LF
    DEV --> GH -->|"preview deploy + DB branch"| S
```

## 2.3 Containers (C4 level 2)

```mermaid
flowchart TB
    subgraph Vercel["Vercel"]
        subgraph Next["Next.js 16 app"]
            RSC["Server Components<br/>dashboard · library · viewer ·<br/>register · admin · billing"]
            SA["Server Actions<br/>mutations (auth + org checked)"]
            RH["Route handlers<br/>/api/chat · /api/chat/[id]/stream ·<br/>/api/stripe/webhook · /api/upload"]
            AG["AI SDK 7 agent<br/>(ToolLoopAgent)"]
            PX["proxy.ts<br/>session gate · org routing"]
        end
        WFS["Workflow SDK runtime<br/>ingest · playbook-run"]
    end
    subgraph Azure["Azure Container Apps"]
        PY["ai-service (FastAPI, Project 1)<br/>parse · chunk · embed · search · rerank"]
    end
    PG[("Neon Postgres + pgvector<br/>RLS · branch per PR")]
    RD[("Upstash Redis<br/>resumable streams · rate limits")]
    BL[("Vercel Blob<br/>originals · page images")]
    ST["Stripe"]
    LLM["Model APIs"]
    PX --> RSC & RH
    RSC --> PG
    SA --> PG
    RH --> AG --> LLM
    AG -->|"tools (HTTP, service JWT)"| PY
    AG --> RD
    SA -->|"start"| WFS
    WFS --> PY & LLM & PG & BL
    PY --> PG
    RH --> ST
    RH --> BL
```

| Container | Responsibility |
|---|---|
| **Next.js app** | UI, auth, org switching, mutations, chat agent, Stripe integration, usage checks |
| **proxy.ts** | Redirect signed-out users; resolve the active org from the URL (`/o/[slug]/…`); security headers |
| **Workflow runtime** | Durable ingestion and playbook runs (steps, retries, sleep, progress streaming) |
| **ai-service** | Project 1's document AI behind an internal API; enforces the org from a signed service token |
| **Neon Postgres** | All relational data, embeddings (pgvector) and RLS; a database branch per preview |
| **Upstash Redis** | Resumable stream buffers; sliding-window rate limits |
| **Vercel Blob** | Uploaded files and rendered page images (private; signed, short-lived URLs) |

## 2.4 Route map

```mermaid
flowchart LR
    ROOT["/"] --> AUTH["/sign-in · /sign-up · /invite/[token]"]
    ROOT --> ORG["/o/[org]"]
    ORG --> DOCS["/o/[org]/documents<br/>/o/[org]/documents/[id]"]
    ORG --> CHAT["/o/[org]/chat · /o/[org]/chat/[id]"]
    ORG --> REG["/o/[org]/register"]
    ORG --> PB["/o/[org]/playbooks · /runs/[id]"]
    ORG --> SET["/o/[org]/settings/members ·<br/>billing · usage · audit"]
```

## 2.5 Data flow A: chat with citations and a resumable stream

```mermaid
sequenceDiagram
    autonumber
    participant B as Browser (useChat, resume)
    participant R as /api/chat
    participant Q as Quota + rate limit
    participant A as Agent (AI SDK 7)
    participant P as ai-service
    participant X as Redis (resumable-stream)
    participant D as Postgres
    B->>R: POST message (chatId)
    R->>Q: check org credits + user rate
    Q-->>R: ok
    R->>A: stream(messages, tools)
    A->>P: search_contracts(query) [service JWT: org, user]
    P-->>A: passages + spans
    A-->>R: UI message stream (text + tool parts)
    R->>X: publish stream (streamId)
    R->>D: chat.activeStreamId = streamId
    R-->>B: SSE
    Note over B: user refreshes the page
    B->>R: GET /api/chat/[id]/stream (auth + org checked)
    R->>X: resume streamId
    X-->>B: remaining parts
    A-->>R: onFinish(usage)
    R->>D: save messages, usage rows, clear activeStreamId
```

The generation runs in `after()`, so tokens keep being produced after the first response closes. The
resume GET **checks the session and org** before re-attaching. The reference implementation leaves this
out, and it is attack T3 in the threat model.

## 2.6 Data flow B: playbook run as a durable workflow

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant SA as Server Action
    participant W as playbookRun workflow
    participant P as ai-service
    participant M as Model
    participant D as Postgres
    U->>SA: run "Vendor review" on 40 docs
    SA->>SA: auth, role ≥ member, credits estimate ≤ remaining
    SA->>W: start(runId, docIds, playbook)
    loop each document (step, retried on failure)
        W->>P: candidate spans per clause type
        W->>M: extract (structured output + citations)
        W->>D: upsert register rows (idempotent key: run, doc, clause)
        W-->>U: progress event (n/40)
    end
    W->>D: risk flags, run summary, usage
    W-->>U: done
```

Each document is one **step**:
- A crash or deploy resumes at the next unfinished document.
- Upserts keyed by `(run_id, document_id, clause_type)` make retries safe.
- Cancel sets a flag that the next step checks.

## 2.7 Data flow C: tool approval in chat

```mermaid
sequenceDiagram
    autonumber
    participant B as Browser
    participant A as Agent
    participant D as Postgres
    A-->>B: tool call update_register_entry(entry 812, notice_days: 60) — needs approval
    B->>B: approval card: exact change (old → new), who, which contract
    B->>A: approve (signed approval, per AI SDK 7)
    A->>D: update (RLS) + audit event
    A-->>B: result part → updated row card
```

## 2.8 Data flow D: sign-up → org → subscription → metering

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant APP as Next.js
    participant BA as Better Auth
    participant S as Stripe
    participant D as Postgres
    U->>APP: sign up
    APP->>BA: create user, org (owner), session.activeOrganizationId
    APP->>S: create customer (org)
    U->>APP: upgrade to Pro
    APP->>S: Checkout session (subscription + metered price)
    S-->>APP: webhook checkout.session.completed (signature verified)
    APP->>D: org.plan = pro, credits reset
    loop every model call / page ingested
        APP->>D: usage_event (idempotent id)
        APP->>S: meter event (batched every minute, idempotency key)
    end
    S-->>APP: invoice.paid / payment_failed webhooks → plan state
```

## 2.9 Deployment view

```mermaid
flowchart TB
    subgraph PR["Every pull request"]
        PV["Vercel preview deploy"]
        NB["Neon branch (copy of seeded staging DB)"]
        E2E["Playwright E2E + Lighthouse +<br/>eval smoke against the preview"]
        PV --> NB
        E2E --> PV
    end
    subgraph PROD["main"]
        VP["Vercel production"]
        NM["Neon main branch"]
        ACA["ai-service on Azure Container Apps<br/>(one staging, one prod revision)"]
    end
    PR -->|"merge"| PROD
```

The ai-service is shared by previews (staging revision). Previews use their own Neon branch, and the
service token carries which database branch to use. If that proves awkward, previews fall back to a
staging database ([ADR-010](07-decisions.md)).

## 2.10 Proposed repository layout

```
clausedesk/
├─ apps/web/                       # Next.js 16
│  ├─ app/(marketing)/  app/(auth)/  app/o/[org]/…
│  ├─ app/api/chat/route.ts  app/api/chat/[id]/stream/route.ts  app/api/stripe/webhook/route.ts
│  ├─ proxy.ts
│  ├─ lib/ai/          # agent, tools, UI part types, prompts
│  ├─ lib/db/          # Drizzle schema, RLS helper (withOrg), migrations
│  ├─ lib/auth/        # Better Auth config (organization plugin)
│  ├─ lib/billing/     # Stripe, meters, quotas, rate limits
│  ├─ workflows/       # ingest.ts, playbook-run.ts
│  └─ components/      # AI Elements + shadcn/ui; register table; viewer
├─ services/ai/                    # Project 1 code as FastAPI (Python)
├─ evals/                          # CUAD extraction, chat citations, isolation, billing accuracy
├─ e2e/                            # Playwright
├─ infra/                          # Terraform (ai-service on Azure), Neon/Vercel config
└─ docs/
```
