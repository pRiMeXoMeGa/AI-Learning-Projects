# F5: Ensemble, Reconciliation & Forecast Service

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M1 | Must | F3, F4 | 5 h | F6, F8, F9, F11 |

**Goal:** A skill-weighted ensemble, MinT reconciliation across the hierarchy, and the FastAPI forecast
service with slices, scenarios, analogs and the nightly job ([03 §3.2](../03-low-level-design.md#32-forecast-service)).

## Diagram: nightly run

```mermaid
flowchart LR
    DATA["latest snapshot"] --> MODS["ets · lgbm · chronos2"]
    MODS --> ENS["ensemble<br/>(weights per level from validation)"]
    ENS --> REC["MinT reconciliation<br/>(hierarchicalforecast)"]
    REC --> RUN[("forecast_run + forecast<br/>(q10 · q50 · q90 · q95)")]
    RUN --> EXC["exceptions: errors · events ·<br/>model disagreement · low cover"]
```

## Deliverables / files
```
forecast/ensemble.py            # weights fitted on validation, frozen
forecast/reconcile.py           # MinT shrink; quantile handling per spike result
forecast/service/app.py         # /forecast/run, /slice, /scenario, /analogs
forecast/service/analogs.py     # past windows with similar event type / price change and their realised uplift
forecast/jobs/nightly.py        # Container Apps Job entry point
forecast/exceptions.py          # ranked exception list per org
```

## Tasks
- [ ] Ensemble weights per hierarchy level from validation skill
- [ ] Reconciliation on the median; quantiles per the S3 spike result; sums checked in tests
- [ ] Slice, scenario (never overwrites a run) and analog endpoints
- [ ] Exceptions: biggest recent errors, upcoming events, low projected cover, high model disagreement
- [ ] Nightly job for the demo org

## Acceptance criteria
- Reconciled forecasts add up at every level (tolerance 1e-6)
- Slice ≤ 1 s, scenario ≤ 5 s p95 for 200 series

## Tests
- Hypothesis: reconciliation sums; quantile monotonicity after reconciliation; API contract tests

**Interview talking point:** *"Planners notice immediately when store forecasts don't add up to the region.
Reconciliation fixes that, and the report shows whether it also helped accuracy."*
