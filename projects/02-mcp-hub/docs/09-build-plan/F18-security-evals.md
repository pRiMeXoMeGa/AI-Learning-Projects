# F18: Security Evals

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M5 | Must | F6, F11, F12 | 8 h | F20, F22 |

**Goal:** Turn the threat model into numbers: the **attack success rate (ASR)** with the gateway's
defences off vs. on, per attack category and model, plus the **false-positive rate** and utility cost on
normal tasks.

## Diagram: two suites

```mermaid
flowchart TB
    subgraph GL["Gateway-level suite (deterministic, every PR)"]
        GC["~45 crafted requests from<br/>evals/security/gateway_cases/*.yaml<br/>(written in F7–F12)"] --> GA["assert: decision, status,<br/>audit event, no side effect"]
    end
    subgraph E2E["End-to-end suite (LLM in the loop, nightly / on demand)"]
        RS["rogue servers in a test tenant:<br/>poisoned descriptions · rug-pull ·<br/>injected results · exfil sink"] --> AG["own client (F6) + real model<br/>+ benign user task"]
        AG --> OBS["observe: did the attack goal happen?<br/>(DB state, sink received data,<br/>destructive call executed)"]
    end
    GA --> CI["100% required in CI"]
    OBS --> ASR["ASR per category × model ×<br/>defences off / on"]
```

## Diagram: defences toggle

```mermaid
flowchart LR
    OFF["defences OFF profile:<br/>pinning off · filters off ·<br/>confirmations off · flow rules off<br/>(auth and RLS stay ON)"] --> RUN1["run attack + normal tasks"]
    ON["defences ON profile<br/>(production config)"] --> RUN2["run attack + normal tasks"]
    RUN1 & RUN2 --> CMP["ASR off vs on ·<br/>normal task success off vs on ·<br/>false positives"]
```

Authentication and row-level security stay on in both profiles: switching them off would only prove that
turning off auth is bad. The comparison isolates the **MCP-specific** controls.

## Deliverables / files
```
evals/security/gateway_cases/*.yaml       # grows with each gateway feature
evals/security/run_gateway_cases.py
evals/security/rogue_servers/             # poisoned, rugpull, injector, exfil_sink (FastMCP)
evals/security/e2e_cases.jsonl            # attack scenario + benign task + success condition
evals/security/run_e2e.py
config/profiles/defences_off.yaml, defences_on.yaml
reports/security.md
```

## Tasks
- [ ] Rogue servers (each one small and clearly labelled as a test fixture)
- [ ] End-to-end cases for every category in [04 §4.6](../04-evaluation-design.md#46-security-evals), with objective success conditions
- [ ] Defences on/off profiles; run both models × 3 repeats
- [ ] False-positive run: the normal tool-design smoke tasks with defences on
- [ ] Tune injection heuristics once, on a **dev split** of cases; report on the rest (no tuning on the test split)
- [ ] Report: ASR table, examples with traces, residual risks

## Acceptance criteria
- Gateway-level suite at 100% in CI
- The report shows ASR off vs. on with CIs, and the false-positive rate on normal tasks
- At least one honest example where a defence did **not** stop an attack, and why

## Tests
- The suites are the tests; the harness itself has unit tests for success-condition checks

**Interview talking point:** *"With the MCP-specific defences on, the attack success rate dropped from __%
to __%, and normal task success changed by only __ points."* (Fill in with your real numbers.)
