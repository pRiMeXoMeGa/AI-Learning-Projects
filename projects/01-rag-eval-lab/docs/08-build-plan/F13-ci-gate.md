# F13: CI Eval Gate

| Milestone | Depends on | Effort | Unblocks |
|---|---|---|---|
| M4 | F12 (+ golden v1 from F11) | 4 h | F16 (deploy only when green) |

**Goal:** A GitHub Actions check that runs the smoke eval on every relevant PR, compares it with the
`main` baseline using [`gate.yaml`](../04-evaluation-design.md#48-ci-gate), **fails on regressions**, and
posts a sticky PR comment with the diff.

## Diagram: workflows

```mermaid
flowchart TB
    subgraph PRW["eval-smoke.yml (pull_request, paths: src/** configs/** data/golden/**)"]
        direction TB
        C1["checkout + uv sync (cached)"] --> C2["services: postgres(pgvector) + redis"]
        C2 --> C3["restore eval cache<br/>(actions/cache key: golden_version)"]
        C3 --> C4["restore index snapshot<br/>(pg_dump of smoke-needed index version)"]
        C4 --> C5["rag-lab eval run --split smoke"]
        C5 --> C6["download baseline results.json<br/>(latest nightly artifact on main)"]
        C6 --> C7["rag-lab eval gate --baseline … --rules configs/gate.yaml"]
        C7 --> C8{"pass?"}
        C8 -->|yes| OK["✅ check green + sticky comment"]
        C8 -->|no| NO["❌ check red + comment:<br/>metric table + top-5 regressed Qs + trace links"]
    end
    subgraph NW["eval-nightly.yml (schedule + push to main)"]
        N1["full eval (150)"] --> N2["upload results.json artifact<br/>= new baseline"] --> N3["update reports/latest.md"]
    end
```

## Diagram: gate decision logic

```mermaid
flowchart TB
    IN["current vs baseline<br/>(same golden_version & judge version?)"] --> SAME{"versions match?"}
    SAME -- no --> WARN["⚠️ neutral: baseline invalid<br/>→ requires a nightly run on main"]
    SAME -- yes --> LOOP["for each metric in gate.yaml"]
    LOOP --> T1{"max_drop / max_increase_pct<br/>exceeded?"}
    T1 -- yes --> FAILM["record failure"]
    T1 -- no --> T2{"min_abs floor breached?"}
    T2 -- yes --> FAILM
    T2 -- no --> PASSM["ok"]
    FAILM & PASSM --> HARD{"hard_fail_on rule hit?<br/>(citation_invalid_rate > 2%)"}
    HARD --> RESULT["exit 1 if any failure, else 0"]
```

## Diagram: security for fork PRs

```mermaid
flowchart LR
    PR{"PR from fork?"} -- yes --> DET["run deterministic retrieval metrics only<br/>(no secrets, fake generator)"]
    PR -- no --> FULL["full smoke eval with provider secrets"]
```

## Deliverables / files
```
.github/workflows/eval-smoke.yml
.github/workflows/eval-nightly.yml
configs/gate.yaml
src/evals/gate.py                    # rules engine (pure) + CLI
scripts/index_snapshot.sh            # build/restore the pg_dump used by CI
```

## Tasks
- [ ] Rules engine (pure function: current, baseline, rules → decision + reasons)
- [ ] Index snapshot so CI doesn't re-embed the corpus (restore a dump for the smoke-relevant documents)
- [ ] Baseline fetch from the latest nightly artifact; version-mismatch handling
- [ ] Sticky PR comment (e.g. `marocchino/sticky-pull-request-comment`)
- [ ] Budget guard + cache restore; timeouts
- [ ] **Demo PR:** switch the default to `fixed-512` / remove the reranker and show the gate blocking it (screenshot for the README)
- [ ] Branch protection: `eval-smoke` required on `main`

## Acceptance criteria
- A PR with no pipeline changes passes with ~0 LLM cost (cache)
- The demo regression PR fails with a readable comment
- The job finishes in < 8 min end to end

## Tests
- Unit: gate rules (drop, increase %, floors, hard-fail, version mismatch)
- Workflow tested with `act`, or on a throwaway branch

**Interview talking point:** *"Prompts and retrieval changes go through CI like code changes. The gate
blocked a PR of mine that silently cut recall by __ points."* (Fill in with your real demo-PR number.)
