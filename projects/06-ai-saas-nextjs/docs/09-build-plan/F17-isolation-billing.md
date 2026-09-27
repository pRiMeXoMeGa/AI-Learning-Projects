# F17: Isolation & Billing Suites

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M6 | Must | F2, F14 | 3.5 h | F20 |

**Goal:** Finish the **tenant isolation suite** (every path, 0 leaks) and the **billing reconciliation**
run with a Stripe test clock.

## Diagram: isolation matrix

```mermaid
flowchart TB
    A["user of org A"] --> P1["pages with B ids"]
    A --> P2["Server Actions with B ids"]
    A --> P3["/api/chat · resume with B chat"]
    A --> P4["expired / replayed blob URLs"]
    A --> P5["ai-service with A token + B doc ids"]
    A --> P6["workflow start with B doc ids"]
    A --> P7["chat retrieval that would match B docs"]
    A --> P8["SQL as app_user with org A"]
    P1 & P2 & P3 & P4 & P5 & P6 & P7 & P8 --> R["all must fail / return nothing<br/>(CI gate: 0 leaks)"]
```

## Diagram: billing reconciliation

```mermaid
flowchart LR
    SIM["scripted month:<br/>chats · uploads · 2 playbook runs"] --> LED["usage ledger credits"]
    SIM --> PROV["provider-reported tokens → credits"]
    LED --> CMP1{"± 1%?"}
    PROV --> CMP1
    LED --> MET["Stripe meter totals (test clock → period end)"]
    MET --> CMP2{"equal after retries<br/>and replayed webhooks?"}
    CMP2 --> INV["invoice = base + overage"]
```

## Deliverables / files
```
tests/isolation/*.spec.ts        # Playwright + API + SQL cases (grown since F2)
evals/billing/reconcile.ts       # test-clock scenario + checks
reports/isolation.md  reports/billing.md
```

## Tasks
- [ ] Fill every path in the matrix; run in CI on each preview
- [ ] Test-clock scenario, forced retries and replayed webhooks; reconciliation checks
- [ ] Short reports

## Acceptance criteria
- 0 leaks across all paths; billing matches within 1% and meters equal the ledger

**Interview talking point:** *"The isolation suite tries every path from pages to the Python service to raw
SQL as another tenant, and a single leak fails the build."*
