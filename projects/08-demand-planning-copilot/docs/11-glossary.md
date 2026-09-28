# 11. Glossary

Plain-English definitions of the terms used in these docs, and where each one shows up in this project.

## Demand planning

| Term | Meaning | In this project |
|---|---|---|
| **Demand planning** | Estimating future sales so the business can buy, make and stock the right amounts | The whole project |
| **S&OP** | Sales and operations planning: the regular meeting where demand and supply plans are agreed | The mental model in [00](00-start-here.md) |
| **SKU / series** | A product (at a store): one time series of daily sales | 30,490 in M5 |
| **Hierarchy** | Item → department → category → store → state → total | Reconciliation |
| **Judgmental adjustment / override** | A planner changing the statistical forecast using knowledge the model lacks | Revision actions |
| **FVA (Forecast Value Added)** | How much a step (e.g. an adjustment) improved the error compared with the forecast before it | Main agent metric |
| **Harmful adjustment** | An adjustment that made the forecast worse (FVA < −1 point here) | Reported rate |
| **Exception** | A series that needs attention: big recent error, upcoming event, low stock | Workspace list |
| **Promo uplift** | Extra sales caused by a promotion | `scale` actions, analogs |
| **SNAP days** | Days when US food-assistance benefits are paid; they move grocery demand | M5 calendar |
| **Service level (α)** | The chance of not running out during a replenishment cycle | Order policy |
| **Fill rate** | Share of demand actually served from stock | Simulation metric |
| **Lead time (L) / review period (R)** | Days from order to delivery / days between ordering decisions | Order policy |
| **Order-up-to level** | The stock position to order up to, from the demand distribution over L + R | Planner agent (maths in code) |
| **Censored demand** | Sales that understate demand because the shelf was empty | FreshRetailNet stockout labels |

## Forecasting

| Term | Meaning | In this project |
|---|---|---|
| **Seasonal naive** | "Same as the same weekday last week" | Floor baseline |
| **ETS / Theta** | Classic statistical forecasting methods per series | Baselines |
| **Croston / TSB** | Methods for intermittent (mostly zero) demand | Slow movers |
| **Global model** | One ML model trained across many series | LightGBM |
| **Tweedie loss** | A loss suited to non-negative, zero-heavy sales | LightGBM |
| **Foundation model (TSFM)** | A model pretrained on many time series that forecasts new ones without training | Chronos-2, TimesFM 2.5 |
| **Covariates** | Extra inputs like price, events and SNAP days; "known future" ones are known in advance | Chronos-2, LightGBM |
| **Quantile forecast** | A range: e.g. the 90% quantile is exceeded only 10% of the time | Bands, order policy |
| **Reconciliation (MinT)** | Adjusting forecasts so lower levels add up to higher levels, optimally | ADR-005 |
| **Rolling-origin backtest** | Re-forecasting from several past dates and comparing with what happened | §4.1 |
| **WRMSSE** | The M5 competition's weighted, scaled error across all hierarchy levels | Headline metric |
| **WAPE / MASE** | Total absolute error ÷ total actuals / error scaled by a naive forecast | Planner-friendly metrics |
| **Bias** | Systematic over- or under-forecasting | Reported per model |

## Agents and platform

| Term | Meaning | In this project |
|---|---|---|
| **Last-mile forecasting** | Revising a baseline forecast with business context, as agents do here | ADR-001 |
| **Revision action** | A typed, bounded change (`scale`, `shift`, `override`, `cap/floor`, `no_change`) with evidence | §3.3 |
| **Revision trace** | The record of what changed, why, and on what evidence | Audit + UI |
| **PlanBench-60** | This project's 60 planning scenarios for scoring agent adjustments | §4.2 |
| **Supervisor / specialist** | An agent that routes work / agents with narrow roles and tools | LangGraph graph |
| **Approval token** | A signed, expiring proof that a human approved a specific proposal | Orders, big revisions |
| **Idempotency key** | A unique key so a retried order doesn't create a second PO | `submit_order` |
| **pass^k** | Share of tasks solved in all k repeated runs (reliability) | P3 harness |
