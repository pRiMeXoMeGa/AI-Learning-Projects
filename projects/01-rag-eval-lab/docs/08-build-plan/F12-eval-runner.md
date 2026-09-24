# F12: Eval Runner & Metrics

| Milestone | Depends on | Effort | Unblocks |
|---|---|---|---|
| M2 | F8, F10, F11 | 8 h | F13, F14, F15 |

**Goal:** `rag-lab eval run --pipeline A3 --golden v1 --split full` runs the pipeline **in-process**
over the golden set, computes all metrics in [04 §4.3–4.6](../04-evaluation-design.md) with bootstrap CIs,
stores results, and links every item to its trace.

## Diagram: runner architecture

```mermaid
flowchart TB
    CLI["rag-lab eval run<br/>--pipeline --golden --split"] --> META["create eval_runs row<br/>git_sha · config_hash · golden_version ·<br/>index_version · model IDs"]
    META --> LOAD["load golden items (split)"]
    LOAD --> POOL["async worker pool<br/>Semaphore(8) + provider token bucket"]
    POOL --> ITEM

    subgraph ITEM["per item"]
        direction TB
        RUN["RAGPipeline.run(question)<br/>(LLM + rerank caches ON)"] --> CAP["capture: candidates (ranked),<br/>answer, citations, usage, latency, trace_id"]
        CAP --> DET["deterministic metrics<br/>recall@k · full-cov · MRR · nDCG · P@8<br/>numeric · abstention · citation validity · filter acc"]
        CAP --> JUD["judged metrics<br/>Ragas faithfulness · relevancy<br/>DeepEval GEval correctness · citation support"]
        DET & JUD --> RES["eval_results row<br/>+ Langfuse scores on trace"]
    end

    ITEM --> AGG["aggregate: mean ± 95% bootstrap CI<br/>overall + by question type"]
    AGG --> SAVE["eval_runs.aggregate_scores<br/>status = done"]
    SAVE --> REP["report.md + results.json<br/>(artifact for CI / F14)"]
```

## Diagram: span-overlap relevance (the core deterministic metric)

```mermaid
flowchart LR
    SPAN["evidence span<br/>chars 184220–184910"] --> OV{"overlap(chunk, span) / len(span) ≥ 0.5<br/>or chunk ⊇ span?"}
    CH1["chunk #3 · 184100–184600"] --> OV
    CH2["chunk #7 · 184600–185200"] --> OV
    OV -->|"#3: 380/690 = 0.55 ✓"| REL["span covered at rank 3"]
    OV -->|"#7: 310/690 = 0.45 ✗ (alone)"| NREL["not sufficient alone"]
```

## Diagram: run comparison (paired bootstrap)

```mermaid
flowchart LR
    A["run A per-item scores"] --> J["join on question_id"]
    B["run B per-item scores"] --> J
    J --> D["Δ_i = B_i − A_i"]
    D --> BS["1,000 bootstrap resamples<br/>of mean Δ"]
    BS --> CI["95% CI of Δ"]
    CI --> V{"CI excludes 0?"}
    V -->|yes| SIG["significant ↑ / ↓"]
    V -->|no| NS["no detectable change"]
    J --> TOPQ["top regressed / improved questions"]
```

## Deliverables / files
```
src/evals/runner.py                  # orchestration, pool, persistence
src/evals/metrics/retrieval.py       # span overlap, recall@k, MRR, nDCG, P@k (pure)
src/evals/metrics/answer.py          # numeric extraction, abstention, citation validity (pure)
src/evals/metrics/judges.py          # Ragas + DeepEval wrappers, judge model from models.yaml
src/evals/stats.py                   # bootstrap, paired bootstrap, Cohen's weighted κ
src/evals/compare.py                 # run diff
src/evals/report.py                  # markdown + json report
src/ragkit/cache/llm_cache.py        # response cache (eval mode only)
```

## Tasks
- [ ] Pure deterministic metric functions with thorough unit tests
- [ ] Judge wrappers: judge model ≠ generator family (assert at startup)
- [ ] LLM, rerank and embedding caches enabled in eval mode; budget guard (estimated tokens > limit → abort)
- [ ] Runner persistence + Langfuse dataset run + scores per trace
- [ ] Bootstrap CIs, per-type breakdown, `compare` command, Markdown report
- [ ] Judge calibration command: `rag-lab eval calibrate --labels …` → κ, Spearman ρ
- [ ] **Record baseline A0 on golden v0** (M2 exit)

## Acceptance criteria
- Re-running the same run with a warm cache costs ≈ 0 tokens and gives identical deterministic metrics
- Smoke run (30 Qs) finishes in < 5 min; full run (150) in < 20 min
- Every `eval_results` row has a working Langfuse trace link
- A0 baseline report committed under `reports/`

## Tests
- Unit: metrics on hand-computed fixtures (the overlap example above); bootstrap with a fixed seed; κ against a known example
- Integration: runner on 3 fake items with fake providers → expected DB rows

**Interview talking point:** *"Retrieval is scored deterministically against evidence spans. The LLM judge
is only used where it has to be, and it's calibrated against my own labels."*
