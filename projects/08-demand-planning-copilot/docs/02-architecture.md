# 2. High-Level Architecture

## 2.1 Design principles

1. **Models make numbers, agents make proposals.** Forecast numbers come only from the forecast service.
   Agents propose typed revision actions and orders; code validates and applies them.
2. **Every change carries its evidence.** A revision action without a citation (a document chunk, a past
   analog, a data query) is rejected.
3. **Money needs a human.** Orders and large adjustments need a signed approval. The approval is checked
   by the tool, not by the prompt.
4. **Trusted code runs as services; LLM-written code runs in the sandbox.** The forecast service is our
   own, tested code. Anything an agent writes on the fly goes to the P5 sandbox.
5. **Measure like a planning team.** Accuracy, FVA and simulated service levels, not just "the answer
   looked right".
6. **Reuse the platform.** MCP through P2's gateway, RAG from P1, agent reliability patterns from P3, A2A
   from P4, the sandbox from P5, the web shell from P6, and all model calls through P7.

## 2.2 System context (C4 level 1)

```mermaid
flowchart LR
    PL(["Planner / category manager /<br/>supply planner"]) -->|"browser"| CAD["Cadence"]
    CAD -->|"LLM calls"| SW["Switchboard (P7)<br/>→ Anthropic / OpenAI"]
    CAD -->|"MCP"| GWY["MCP gateway (P2)<br/>OAuth · Cedar"]
    CAD -->|"code runs"| SBX["Sandbox broker (P5)"]
    CAD -->|"A2A"| SUP["Supplier agent (P4 pattern)"]
    CAD -->|"traces"| LF["Langfuse"]
    DATA[("public data:<br/>M5 · FreshRetailNet-50K")] --> CAD
```

## 2.3 Containers (C4 level 2)

```mermaid
flowchart TB
    subgraph WEB["Web (Vercel)"]
        UI["Next.js 16 workspace<br/>(P6 shell · AI SDK 7 UI)"]
        BFF["route handlers:<br/>auth · session · approval inbox"]
    end
    subgraph AGT["Agent service (Container Apps)"]
        SUPV["LangGraph supervisor"]
        SPEC["Forecast Reviewer · Analyst · Planner"]
        APPLY["revision engine<br/>(validate · apply · reconcile)"]
        CKP[("checkpoints (Postgres)")]
    end
    subgraph TOOLS["MCP servers (behind P2 gateway)"]
        T1["sales-mcp (read)"]
        T2["forecast-mcp"]
        T3["inventory-mcp (orders need approval)"]
        T4["knowledge-mcp (P1 RAG)"]
        T5["memory-mcp"]
    end
    subgraph FC["Forecast service"]
        API["FastAPI: forecast · scenario · backtest"]
        JOB["nightly job (Container Apps Job)"]
        MOD["stats · LightGBM · Chronos-2 · ensemble ·<br/>reconciliation"]
    end
    subgraph STATE["State"]
        PG[("Postgres + pgvector:<br/>canonical data · forecasts · revisions ·<br/>approvals · orders · audit")]
        BLOB[("Blob: Parquet snapshots,<br/>model artifacts")]
    end
    ERP["simulated ERP + inventory"]
    UI --> BFF --> SUPV --> SPEC
    SPEC -->|"MCP"| T1 & T2 & T3 & T4 & T5
    SPEC --> APPLY --> T2
    T2 --> API --> MOD
    JOB --> MOD
    T1 & T2 & T3 --> PG
    MOD --> BLOB
    T3 --> ERP
```

| Container | Responsibility |
|---|---|
| **Workspace** | Exceptions, charts, diffs, scenario sliders, approval inbox, timeline; streams agent output as typed UI parts |
| **Agent service** | LangGraph graph with checkpoints; the revision engine that validates and applies actions; approval token checks happen in tools |
| **MCP servers** | The only way agents touch data or act; each has a narrow tool set and Cedar policies per role |
| **Forecast service** | Model fitting, prediction, scenarios, reconciliation, backtests; no LLM inside |
| **Simulated ERP** | Inventory positions, open orders, receipts; runs the replenishment simulation day by day |
| **Supplier agent** | An A2A agent in a separate "supplier" org that confirms, part-fills or delays orders |

## 2.4 The weekly planning loop

```mermaid
flowchart LR
    N["nightly: actuals in →<br/>forecasts out (all series)"] --> X["exceptions ranked"]
    X --> R["review a slice:<br/>context → revision actions"]
    R --> A{"above threshold?"}
    A -- yes --> AP["planner approval"]
    A -- no --> APPLY["apply + reconcile"]
    AP --> APPLY
    APPLY --> O["order proposals<br/>(quantiles → order-up-to)"]
    O --> OA["planner approval (always)"]
    OA --> ERP["ERP + supplier (A2A)"]
    ERP --> ACT["actuals arrive"]
    ACT --> FVA["FVA scored per adjustment"]
    FVA -.-> N
```

## 2.5 The agent graph

```mermaid
flowchart TB
    START(["planner request"]) --> SUP{"supervisor:<br/>classify intent"}
    SUP -->|"review / adjust"| REV["Forecast Reviewer"]
    SUP -->|"why / explain"| ANA["Analyst"]
    SUP -->|"order"| PLN["Planner"]
    REV --> VAL["revision engine:<br/>validate + preview"]
    VAL --> GATE{"needs approval?"}
    GATE -- yes --> INT["interrupt → approval inbox"]
    GATE -- no --> SUP
    INT --> SUP
    ANA --> SUP
    PLN --> INT2["interrupt → order approval"]
    INT2 --> SUP
    SUP -->|"done"| END(["summary + revision trace"])
```

The supervisor routes, the specialists each have a small tool set, and every action that changes a plan
or spends money goes through an interrupt. The same tools are given to a **single-agent baseline**, so
the evaluation can say whether the multi-agent design earns its extra cost ([ADR-006](07-decisions.md)).

## 2.6 Data flow A: a context-driven adjustment

```mermaid
sequenceDiagram
    autonumber
    participant P as Planner
    participant R as Forecast Reviewer
    participant K as knowledge-mcp (P1 RAG)
    participant F as forecast-mcp
    participant E as Revision engine
    P->>R: "Anything to change for CA_1 foods, next 4 weeks?"
    R->>F: get_forecast(slice, run_id)
    R->>K: search("promo CA_1 foods", date window)
    K-->>R: promo plan §2.3 (price cut, 40 SKUs, days 8–14) + citation
    R->>F: find_analogs(similar past promos)
    F-->>R: past uplift distribution (+15% to +22%)
    R->>E: propose scale(+18%, 40 series, days 8–14, evidence=[§2.3, analogs])
    E->>E: validate bounds + window + evidence, apply, re-reconcile
    E-->>P: before/after chart + diff (approval needed: +18% < 25%, so no)
    P->>E: accept
    E->>F: save revision (run_id, action, author=agent, accepted_by=planner)
```

## 2.7 Data flow B: an order with approval and a supplier

```mermaid
sequenceDiagram
    autonumber
    participant PL as Planner agent
    participant I as inventory-mcp
    participant U as Planner (human)
    participant ERP as Simulated ERP
    participant S as Supplier agent (A2A)
    PL->>I: get_position(series) → on-hand, open orders, lead time
    PL->>PL: order-up-to from adjusted quantiles (95% service level)
    PL->>U: proposal: 120 cases SKU_A, 80 cases SKU_B (interrupt)
    U-->>PL: approve (edits SKU_B to 60) → signed approval token
    PL->>I: submit_order(lines, approval_token, idempotency_key)
    I->>I: verify token (signature, scope, amounts, expiry)
    I->>ERP: create PO
    ERP->>S: A2A message/send (PO)
    S-->>ERP: confirmed SKU_A, part-filled SKU_B (50), lead time 3 days
    ERP-->>U: status in the approval inbox
```

## 2.8 Deployment view

```mermaid
flowchart TB
    subgraph Vercel
        W["Next.js workspace"]
    end
    subgraph Azure["Azure Container Apps (shared env)"]
        AGS["agent service"]
        MCPS["MCP servers"]
        FSV["forecast service"]
        JOBS["nightly forecast job ·<br/>simulation job"]
        SUPA["supplier agent"]
    end
    PGF[("Postgres Flexible + pgvector")]
    BL[("Blob")]
    P2["P2 gateway"]
    P5["P5 sandbox broker"]
    P7["P7 Switchboard"]
    W --> AGS --> P2 --> MCPS
    AGS --> P5 & P7
    MCPS --> FSV
    FSV & JOBS --> PGF & BL
```

GPU is not assumed. Chronos-2 runs on CPU for the demo scope; full-M5 foundation-model backtests run on a
rented GPU or a stratified sample (decided in the first spike, [06 §6.2](06-non-functional.md#62-batch-sizing)).

## 2.9 Proposed repository layout

```
cadence/
├─ data/               # loaders (M5, FreshRetailNet-50K), canonical schema, provenance records (no raw M5)
├─ docs-corpus/        # fictional promo plans, playbooks, meeting notes (+ ground-truth links to events)
├─ forecast/           # models, ensemble, reconciliation, backtests, scenario API, nightly job
├─ sim/                # inventory + ERP simulator, order policies, supplier agent (A2A)
├─ mcp/                # sales, forecast, inventory, memory servers (knowledge = P1, sandbox = P5)
├─ agents/             # LangGraph supervisor + specialists, single-agent baseline, revision engine
├─ web/                # Next.js workspace (P6 shell packages)
├─ evals/              # backtest reports, PlanBench-60, FVA, replenishment, trajectory, RAG, red team
├─ infra/              # Terraform, compose
└─ docs/
```
