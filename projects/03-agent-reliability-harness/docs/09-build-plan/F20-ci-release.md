# F20: CI Gate, PyPI Release & Scoring CLI

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M5 | Must | F8, F13 | 4.5 h | F22, external users |

**Goal:** Protect the agents and the benchmark with a CI gate, publish `opssim` (simulator, MCP server
and dev scenarios) to PyPI, and let anyone score their own agent with `hctl score`.

## Diagram: CI gate

```mermaid
flowchart LR
    PR["PR"] --> U["unit + grader tests<br/>+ hctl qa (no model cost)"]
    U --> CH{"touches agents/, spec/,<br/>opssim/ ?"}
    CH -- no --> OK["✅"]
    CH -- yes --> SM["smoke: 10 dev scenarios × k=2 ×<br/>LangGraph + 1 other impl · cheap model"]
    SM --> G{"vs main baseline:<br/>success drop ≤ 15 pts ·<br/>no S1 violation ·<br/>cost/run up ≤ 30%"}
    G -- yes --> OK
    G -- no --> BLK["❌ blocked +<br/>PR comment with diff table"]
```

## Diagram: scoring an external agent

```mermaid
sequenceDiagram
    autonumber
    participant X as External user
    participant H as hctl score
    participant O as opssim (local)
    participant A as Their agent
    X->>H: hctl score --agent-cmd "python my_agent.py" --split dev --k 4
    loop each scenario × k
        H->>O: POST /runs → run_id, mcp_url
        H->>A: spawn with env: OPSDESK_MCP_URL, APPROVALS_URL, TICKET
        A->>O: MCP tool calls
        A-->>H: final report JSON on stdout
        H->>O: GET /runs/{id}/final → grade
    end
    H-->>X: score table (pass^k, violations, HITL, cost if reported)
```

## Deliverables / files
```
.github/workflows/ci.yml             # + smoke job with thresholds, PR comment
.github/workflows/release-opssim.yml # PyPI trusted publishing on tag
opssim/pyproject.toml                # includes dev scenarios; test scenarios added after the report
harness/score.py                     # external agent protocol (env vars in, JSON out)
scenarios/opsdesk-50/README.md       # "how to score your agent"
```

## Tasks
- [ ] Smoke job: cached baseline from `main`, thresholds from [04 §4.9](../04-evaluation-design.md#49-ci-gate), budget ≤ $1
- [ ] Demo a deliberately bad PR (e.g. remove the approval rule from the spec) → blocked
- [ ] Package `opssim` with the dev scenarios; `uvx opssim serve` starts the server + approval service (SQLite-only mode)
- [ ] External agent protocol + `hctl score`; test with a tiny example agent
- [ ] Grader/scenario semver in releases; `hctl score` refuses to compare results across major grader versions *(market review)*
- [ ] PyPI trusted publishing; release notes

## Acceptance criteria
- The bad PR is blocked with a readable comment
- On a clean machine: `uvx opssim serve` + the example agent + `hctl score` works in under 5 minutes

## Tests
- CI workflow tested on a branch; score CLI tested with the example agent (stub model)

**Interview talking point:** *"The benchmark is a package: `uvx opssim serve`, point any MCP agent at it,
and `hctl score` gives you pass^k and safety numbers."*
