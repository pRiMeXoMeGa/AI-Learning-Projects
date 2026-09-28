# F0: Foundation & Spikes

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M1 | Must | — | 3.5 h | Everything |

**Goal:** A monorepo with the web app (using Project 6's shell packages), the broker, the runner and the
sandbox image, plus CI. Four spikes settle the risky assumptions.

## Diagram: repository

```mermaid
flowchart TB
    subgraph Repo["analyst (monorepo)"]
        W["apps/web (Next.js 16, P6 packages)"]
        B["services/broker (Python, FastMCP)"]
        R["services/runner (Python, on gVisor host)"]
        I["sandbox-image (Dockerfile, harness)"]
        D["datasets/ · evals/ · redteam/ · infra/"]
    end
    CI["GitHub Actions: lint · types · pytest ·<br/>vitest · image build + Trivy"] --> Repo
```

## Diagram: spikes (≈ 1.5 h total, results in `docs/notes/spikes.md`)

```mermaid
flowchart TB
    S1["S1 · E2B sandbox with internet disabled:<br/>curl, DNS, 169.254.169.254 all fail?"] --> D1{"yes?"}
    D1 -- no --> F1["gVisor becomes default;<br/>report the finding"]
    S2["S2 · runsc on an Azure D4s VM:<br/>pandas + DuckDB on a 1M-row Parquet"] --> D2{"works, acceptable speed?"}
    D2 -- no --> F2["tune platform / VM size;<br/>document gaps"]
    S3["S3 · vega-embed + vega-interpreter<br/>under CSP without unsafe-eval"] --> D3{"renders?"}
    D3 -- no --> F3["smaller chart subset; PNG fallback"]
    S4["S4 · @ai-sdk/mcp client ↔ FastMCP 4<br/>(2026-07-28, auth)"] --> D4{"ok?"}
    D4 -- no --> F4["HTTP tool wrappers, same schemas"]
```

## Tasks
- [ ] Monorepo (pnpm + uv); import Project 6's shell packages (auth, UI, AI Elements)
- [ ] CI: web and Python checks; image build + Trivy + Syft SBOM (scanner actions pinned by commit SHA; scan job has no secrets)
- [ ] Spikes S1–S4 with written decisions

## Acceptance criteria
- CI green on an empty PR; spike notes committed with a decision per spike

**Interview talking point:** *"Before building anything, I checked the one assumption the whole design
rests on: that a sandbox really can't reach the network. I tested DNS and the cloud metadata endpoint
too, not just 'curl google.com'."*
