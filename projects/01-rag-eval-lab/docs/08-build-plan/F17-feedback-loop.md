# F17: Online Feedback Loop (stretch)

| Milestone | Depends on | Effort | Unblocks |
|---|---|---|---|
| M5 (stretch) | F9, F10 | 2 h | Golden set v2 (F11) |

**Goal:** Close the loop from real usage back into the golden set: user feedback and sampled online judge
scores surface bad answers, which become new golden questions.

## Diagram: the loop

```mermaid
flowchart LR
    U(["user"]) -->|"👍 / 👎 + comment"| FB["POST /v1/feedback"]
    FB --> LS["Langfuse score on trace"]
    CRON["nightly job"] --> SAMP["sample 5% of traces<br/>+ all 👎 traces"]
    SAMP --> OJ["online judges:<br/>faithfulness · citation validity"]
    OJ --> LS
    LS --> TRI["triage queue:<br/>👎 or low score"]
    TRI --> REV["you review in F11 tool"]
    REV -->|"real failure"| NEWQ["new golden question<br/>(with evidence span)"]
    NEWQ --> GV2[("golden v2")]
    GV2 --> GATE["F13 gate now protects<br/>against this failure"]
```

## Diagram: triage decision

```mermaid
flowchart TB
    T["flagged trace"] --> Q1{"answer wrong?"}
    Q1 -- no --> DISM["dismiss (label: false alarm)"]
    Q1 -- yes --> Q2{"evidence exists in corpus?"}
    Q2 -- no --> ABS["golden Q type: unanswerable<br/>(system should have abstained)"]
    Q2 -- yes --> Q3{"was it retrieved?"}
    Q3 -- no --> RET["retrieval failure → golden Q + tag 'retrieval'"]
    Q3 -- yes --> GEN["generation failure → golden Q + tag 'generation'"]
```

## Deliverables / files
```
src/api/routes/feedback.py
src/evals/online.py                  # nightly sampler + judges
.github/workflows/online-eval.yml    # schedule
```

## Tasks
- [ ] Feedback endpoint → Langfuse score (trace ID from the `meta` event)
- [ ] Nightly sampler + online judges; trend chart in Langfuse
- [ ] Triage export → F11 review tool with pre-filled question, answer and trace

## Acceptance criteria
- Feedback appears on the trace within seconds
- At least 5 golden v2 questions come from real or demo usage (shows the loop working)

**Interview talking point:** *"Production failures become regression tests. The eval set grows from real
usage, not only from synthetic data."*
