# F11: Stateless Confirmations (signed MRTR state)

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M3 | Must | F10, F5 | 4 h | F16, F18 |

**Goal:** When policy says `require_confirmation`, return `input_required` with a human-readable summary
and a **signed** `requestState`, so any replica can complete the call on retry, and nothing can be
replayed, altered or used by another user. Also pass upstream `input_required` requests through safely.

## Diagram: issuing and verifying the state

```mermaid
flowchart TB
    subgraph Issue["first call"]
        P["policy: require_confirmation"] --> SUM["build summary<br/>('Remove 120.5 units of Fund X?')"]
        SUM --> ST["payload {v, kid, sub, client_id, tool,<br/>args_sha256, exp=now+300 s, nonce,<br/>upstream_state?}"]
        ST --> SIG["HMAC-SHA256 with active key"]
        SIG --> IR["input_required + requestState"]
    end
    subgraph Verify["retry (any replica)"]
        R["retry + inputResponses + requestState"] --> V1{"signature ok (by kid)?"}
        V1 -- no --> X["deny + audit 'state_tampered'"]
        V1 -- yes --> V2{"not expired · sub & client match ·<br/>args hash matches?"}
        V2 -- no --> X
        V2 -- yes --> V3{"nonce unused?<br/>(Redis SET NX)"}
        V3 -- no --> X2["deny + audit 'state_replayed'"]
        V3 -- yes --> V4{"user confirmed?"}
        V4 -- no --> CANCEL["result: cancelled by user"]
        V4 -- yes --> GO["continue → F9 upstream call"]
    end
```

## Diagram: wrapping an upstream's own `input_required`

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as Gateway
    participant MF as india-mf-mcp
    C->>GW: tools/call mf__watchlist_add {fund: "…"}
    GW->>MF: tools/call watchlist_add
    MF-->>GW: input_required (form, requestState = S_up)
    GW-->>C: input_required (same form, requestState = sign({…, upstream_state: S_up}))
    C->>GW: retry + answers + gateway state
    GW->>GW: verify gateway signature, unwrap S_up
    GW->>MF: retry + answers + S_up
    MF-->>GW: result
    GW-->>C: result
```

## Deliverables / files
```
gateway/src/mcphub/confirm/state.py     # sign / verify (pure, key ring with kid)
gateway/src/mcphub/confirm/summary.py   # per-tool human-readable summaries
gateway/src/mcphub/confirm/flow.py      # issue, verify, nonce, upstream wrapping
```

## Tasks
- [ ] Key ring with `kid`, two active keys, rotation without breaking in-flight confirmations
- [ ] Canonical args hash (same canonicalisation as F8)
- [ ] Human-readable summaries from tool arguments (looked up: fund name, units), never raw IDs only
- [ ] Nonce store in Redis with TTL; refuse confirmations when Redis is down
- [ ] Upstream `input_required` wrapping and unwrapping
- [ ] **Attack cases:** replay, changed args, other user's state, expired state, forged signature, state from a different tool

## Acceptance criteria
- 1,000 confirmation flows across two replicas (round-robin) all succeed
- Every tampering case is denied and audited with the right reason
- The disambiguation flow from F5 still works through the gateway

## Tests
- Unit (hypothesis): any single-byte change to payload or signature fails verification
- Integration: two replicas, Redis, real upstream `input_required`

**Interview talking point:** *"A pending confirmation isn't stored on the server. It travels with the
retry, signed and bound to the user, the tool and a hash of the exact arguments, so any replica can finish
it and nobody can reuse or alter it."*
