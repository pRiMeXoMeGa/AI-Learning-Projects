# F0: Foundation, CI, Previews & Spikes

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M1 | Must | — | 4 h | Everything |

**Goal:** A pnpm monorepo with the Next.js 16 app and the Python ai-service. Every PR gets a Vercel
preview **and** its own Neon database branch, and four spikes settle the risky assumptions.

## Diagram: per-PR pipeline

```mermaid
flowchart LR
    PR["open / push PR"] --> CI["GitHub Actions:<br/>typecheck (TS 7) · lint · Vitest · pytest"]
    PR --> NB["Neon: create branch pr-123<br/>from 'seeded' template → migrate"]
    PR --> VP["Vercel preview build<br/>(env: DATABASE_URL = branch)"]
    NB & VP --> READY["preview ready"]
    READY --> E2E["E2E + isolation + Lighthouse<br/>(grows from F1 on)"]
    CLOSE["PR closed"] --> DEL["delete Neon branch"]
```

## Diagram: spikes (≈ 1 h total, results in `docs/notes/spikes.md`)

```mermaid
flowchart TB
    S1["S1 · withOrg over Neon WebSocket Pool on Vercel:<br/>set_config + query in one transaction; latency"] --> D1{"ok?"}
    D1 -- no --> F1b["pg Pool via pooled endpoint"]
    S2["S2 · AI SDK 7 tool approval inside a resumable stream"] --> D2{"ok?"}
    D2 -- no --> F2b["approval ends turn; execute next request"]
    S3["S3 · Better Auth Stripe plugin: base + metered price, org reference"] --> D3{"ok?"}
    D3 -- no --> F3b["metered item via Stripe SDK"]
    S4["S4 · TypeScript 7 with Next.js 16, ESLint, editor"] --> D4{"ok?"}
    D4 -- no --> F4b["TypeScript 6.x for now"]
```

## Deliverables / files
```
pnpm-workspace.yaml   apps/web (create-next-app, App Router, Turbopack)   services/ai (from Project 1)
apps/web/components/ui (shadcn init)   apps/web/components/ai-elements (AI Elements add)
.github/workflows/ci.yml   .github/workflows/neon-branch.yml   vercel.json (if needed)
.env.example   docs/notes/spikes.md
```

## Tasks
- [ ] Monorepo, Next.js 16.3 app, Tailwind 4, shadcn/ui, AI Elements; strict TS; ESLint + Prettier
- [ ] Vercel project with Git integration; Neon project with a `seeded` template branch
- [ ] Neon branch workflow: create on PR, migrate, inject `DATABASE_URL` into the preview; delete on close
- [ ] CI jobs; Dependabot; secret scanning
- [ ] Spikes S1–S4 with decisions

## Acceptance criteria
- A PR produces a preview URL backed by its own database branch
- Spike notes committed

**Interview talking point:** *"Every pull request gets a full copy of the product: its own URL and its own
database branch, and the end-to-end tests run against it before merge."*
