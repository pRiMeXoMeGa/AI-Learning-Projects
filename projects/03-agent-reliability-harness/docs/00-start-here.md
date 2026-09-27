# 0. Start Here: the project in plain English

New to these docs, or preparing for an interview? Read this page first. Unfamiliar terms are explained in
the [glossary](11-glossary.md).

## 0.1 The project in three sentences

1. **OpsSim** is a pretend production system for a small online shop: services, deploys, alerts, logs
   and runbooks. Each run gets a fresh copy, and faults and attacks are planted on purpose.
2. The **Triage Agent** handles incidents in it: it investigates, asks a human before risky actions, fixes,
   and reports. It is built **four times**: plain code, LangGraph, the OpenAI Agents SDK and the Claude
   Agent SDK, with the same prompt and tools.
3. The **harness** runs 50 incident scenarios several times per implementation. From that it measures how
   often each version succeeds **every** time, how often it does something dangerous, whether it asks for
   approval when it should, whether it survives a crash, and what it costs.

## 0.2 Which path should you read?

| You want to… | Read, in this order | Time |
|---|---|---|
| Explain the project in an interview | This page → [Evaluation](04-evaluation-design.md) → [Decisions](07-decisions.md) → [Glossary](11-glossary.md) | 30 min |
| Understand the design | [Requirements](01-requirements.md) → [Architecture](02-architecture.md) → [Low-level design](03-low-level-design.md) | 1 h |
| Understand the safety story | [Safety & threat model](05-safety-threat-model.md) | 15 min |
| Understand cost, speed and reproducibility | [Non-functional design](06-non-functional.md) | 15 min |
| Understand the technology choices | [Tech stack](08-tech-stack.md) | 20 min |
| Start building | This page → [Setup guide](10-setup-guide.md) → [Build plan](09-build-plan/README.md) → the feature page you're on | 45 min |

## 0.3 The mental model: a flight simulator for on-call agents

```mermaid
flowchart LR
    subgraph SIM["🛩️ Flight simulator = OpsSim"]
        SC["scenario card:<br/>'engine fire after takeoff'<br/>= bad deploy, injected log line"]
        COCK["cockpit instruments = MCP tools<br/>metrics · logs · deploys · runbooks"]
        REC["black-box recorder =<br/>environment action log"]
    end
    subgraph PILOTS["👩‍✈️ Four pilots, same training manual"]
        P0["raw loop"]
        P1["LangGraph"]
        P2["OpenAI SDK"]
        P3["Claude SDK"]
    end
    TWR["🗼 Control tower = approval service<br/>'cleared to roll back?'"]
    EXAM["📋 Examiner = harness<br/>each scenario flown 4×"]
    SC --> COCK
    PILOTS -->|"fly with"| COCK
    PILOTS -->|"ask clearance"| TWR
    COCK --> REC --> EXAM
```

| Simulator idea | Real component | Why it matters |
|---|---|---|
| Scenario card | Scenario YAML (fault, attacks, expected outcome) | Every failure is planted deliberately and is checkable |
| Same training manual | Shared agent spec | Differences come from the framework, not the prompt |
| Control tower | Approval service + signed tokens | Risky actions need clearance, and the simulator refuses them without it |
| Black-box recorder | Environment action log | Grading uses what actually happened, not the pilot's story |
| Flying each scenario 4 times | pass^k | One lucky landing doesn't count; consistency does |
| Engine cut mid-flight | Chaos mode (kill + resume) | Tests whether the agent picks up safely without doing things twice |

## 0.4 One run's journey

> **Ticket:** "PagerDuty: checkout-svc latency p95 > 1500 ms for 10 min."

| Step | What happens | Where |
|---|---|---|
| 1. Fresh world | The runner creates run `r_42` from scenario S1-04; a seeded copy of ShopLite is made | OpsSim |
| 2. Investigate | The agent reads the alert, checks latency and error metrics, searches logs (*"PaymentClient timeout… pool exhausted"*), lists deploys: **v42 went out 12 min ago** | Agent → MCP tools |
| 3. Planted trouble | The metrics tool times out once (planted fault); the agent retries | OpsSim fault injector |
| 4. Runbook | The runbook says "latency after deploy → roll back to the last good version" | MCP tool |
| 5. Ask first | Rollback is high-risk, so the agent pauses and requests approval. Its state is saved, so the process could even stop here | Framework HITL → approval service |
| 6. Human edits | The scripted approver approves rollback **to v41** and issues a signed token | Approval service |
| 7. Act safely | The agent calls `rollback_deployment(v41)` with the token and an idempotency key; the simulator checks both | OpsSim |
| 8. Verify | Three simulated minutes later, latency is back to normal; the agent checks the metrics | Agent → MCP tools |
| 9. Communicate | Incident SEV3 created and linked; update posted to `#inc-checkout` | MCP tools |
| 10. Report + grade | The agent returns JSON (status, root cause, confidence 0.85). The harness checks final state, forbidden actions, approvals, trajectory and cost | Harness graders |

Then the same scenario runs **three more times**, and in all four implementations.

## 0.5 What makes this project stand out in interviews

- **Framework judgement with numbers.** "I built the same agent in four frameworks and measured them" is
  rare. Most comparisons are opinion pieces.
- **Reliability, not accuracy.** pass^k, safety severity and calibration follow the 2026 research on agent
  reliability, not just a success rate.
- **Production concerns are measured:** human approval, crash/resume without double actions, memory
  poisoning, budgets. Each one has an on/off experiment.
- **A public benchmark others can use** (OpsDesk-50 + scoring CLI), not only a demo.
- **An honest report:** stated power limits, a test split frozen before tuning, threats to validity.

## 0.6 FAQ

**Why simulate instead of using real infrastructure?**
To run more than 1,000 episodes for tens of dollars, with planted faults and exact grading. Real-cluster
benchmarks (ITBench, SREGym) exist for root-cause skill; this project measures **reliability and
safety across frameworks**, which needs many cheap, repeatable runs. ([ADR-001](07-decisions.md))

**Isn't comparing frameworks unfair if they use different models?**
Yes, so the main comparison fixes the model (a Claude model for all four). A second experiment swaps in an
OpenAI model where possible, to separate framework effects from model effects. ([ADR-005](07-decisions.md))

**Why is there a "raw loop" with no framework?**
It's the control group. If a 200-line loop matches the frameworks, that's a finding too. It usually shows up
in what frameworks give you for free: pause/resume, tracing, persistence.

**What stops the agent from doing something dangerous?**
Three layers:
1. The agent's own approval step.
2. The simulator refuses risky calls without a signed approval token.
3. Budgets and loop limits stop runaway runs.

Everything is simulated anyway, so the goal is to measure how often each layer is needed.
([05](05-safety-threat-model.md))

**How is this different from Project 1's agent?**
Project 1's agent only reads documents and is compared with a fixed pipeline. This one **acts**, waits
for humans, survives crashes, has memory, and is compared **across frameworks**.

**Is this the code?**
Not yet. These are the design documents, the [tech-stack rationale](08-tech-stack.md) and the
[build plan](09-build-plan/README.md), written before any code.
