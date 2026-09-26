# F17: Tool-Design Evals

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M5 | Must | F6, F9, F13 | 9 h full · **6 h core (T1, T2, T4)** | F20, F22 |

**Goal:** Measure how **tool granularity, description style and output schemas** change agent task
success, reliability and token cost, for **two model families**, using the design in
[04 §4.3–4.5](../04-evaluation-design.md).

## Diagram: harness

```mermaid
flowchart TB
    SNAP[("frozen DB snapshot")] --> EXP["expected answers computed by code<br/>(returns, XIRR, final watchlist state)"]
    TASKS["tasks.jsonl (60)"] --> RUN
    VAR["toolset variants T1–T5<br/>(served by india-mf-mcp via a variant flag)"] --> RUN
    MOD["models.yaml: claude-* · openai-*"] --> RUN
    RUN["runner: task × variant × model × 3 repeats<br/>(own client F6, scripted user simulator,<br/>LLM response cache)"] --> TRJ[("trajectories + final DB state")]
    TRJ --> SC["scoring: success · pass^3 · selection P/R ·<br/>arg validity · calls · tokens (incl. tools/list) · latency"]
    EXP --> SC
    SC --> ST["paired bootstrap per variant pair<br/>(Project 1 stats code)"]
    ST --> REP["reports/tool-design.md<br/>+ charts + per-type tables"]
```

## Diagram: how variants are served

```mermaid
flowchart LR
    V{"TOOLSET_VARIANT"} -->|T1| A["12 focused tools,<br/>rich descriptions, output schemas"]
    V -->|T2| B["4 coarse workflow tools<br/>with a 'mode' argument"]
    V -->|T3| C["20 fine-grained tools"]
    V -->|T4| D["T1 with one-line descriptions"]
    V -->|T5| E["T1 without output schemas"]
    A & B & C & D & E --> SAME["same maths and data underneath<br/>(only the tool surface changes)"]
```

## Deliverables / files
```
evals/tool_design/tasks.jsonl           # 60 tasks with types and expectations
evals/tool_design/expected.py           # computes expected values from the snapshot
evals/tool_design/simulator.py          # scripted answers for input_required
evals/tool_design/run.py                # matrix runner, concurrency, budget guard
evals/tool_design/score.py              # metrics (reuses Project 1 trajectory metrics)
evals/tool_design/report.py             # tables, charts
servers/india-mf-mcp/src/india_mf/variants/  # T2–T5 tool surfaces
reports/tool-design.md
```

## Tasks
- [ ] Freeze the snapshot; write `expected.py` so answers are never hand-typed
- [ ] Write the 60 tasks (7 types, [04 §4.3](../04-evaluation-design.md)); ≥ 50% hand-written
- [ ] Variant surfaces T2–T5 behind a flag (core plan: T1, T2, T4 only)
- [ ] Runner with cache, budget guard and resume; smoke set of 15 tasks
- [ ] Scoring incl. final-state checks (e.g. watchlist contents in the DB after the task)
- [ ] Report: overall and per task type, per model, with CIs; token cost of `tools/list` per variant

## Acceptance criteria
- Full matrix (or core subset) runs end to end; every run has a Langfuse trace
- The report states at least three findings, each backed by a significant paired difference (or honestly says "no difference")

## Tests
- Unit: expected-value computation, scoring on fixture trajectories
- A fake-LLM run of the whole pipeline in CI (no API cost)

**Interview talking point:** *"I measured tool design instead of guessing: for example, what minimal
descriptions or coarse tools do to multi-step success and tokens per task, for two model families."*
(Replace with your real findings.)
