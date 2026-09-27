# F12: Experiments & DABstep

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M4 | Must | F11 | 3 h (4 with X3, X4) | F15 |

**Goal:**
- Answer the design questions with numbers: X1 (Python+SQL vs SQL-only) and X2 (stateful vs stateless).
- Get one external number from **DABstep**.

## Diagram: experiment matrix

```mermaid
flowchart TB
    AB["AnalystBench-50 × k=3"] --> X1a["X1: tools = python+sql"]
    AB --> X1b["X1: tools = sql only"]
    AB --> X2a["X2: stateful kernel"]
    AB --> X2b["X2: stateless runs"]
    X1a & X1b & X2a & X2b --> P["paired comparison per question<br/>(bootstrap CIs)"]
    DAB["DABstep dev tasks (local scoring)"] --> FZ["freeze agent"] --> SUB["one leaderboard submission"]
```

## Deliverables / files
```
experiments/x1_tools.yaml   experiments/x2_state.yaml
evals/dabstep/run.py        # dev set locally; submission file for the leaderboard
reports/experiments.md      reports/dabstep.md
```

## Tasks
- [ ] X1 and X2 on AnalystBench (reuse F11 runs as one arm where possible)
- [ ] DABstep: load tasks and context docs, adapt the agent's output format, run dev locally
- [ ] Freeze the agent version (tag) → one leaderboard submission → record the score with the model named
- [ ] *(Full plan)* X3 execution budget 3 vs 6; X4 model A vs B

## Acceptance criteria
- Experiments report with paired differences and CIs; DABstep dev score and leaderboard entry

**Interview talking point:** *"Code execution is a risk, so I measured what it buys: Python plus SQL
versus SQL-only on the same 50 questions. The difference was __ points, mostly on __."* (Fill in.)
