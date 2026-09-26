# F9: Routing, Token Exchange & stdio Bridge

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M3 | Must | F8 | 6 h | F12, F16, F17 |

**Goal:** Forward each allowed call to the right upstream with the **right credential**: an exchanged,
audience-restricted token (never the client's token), a stored per-user token, or none for a local stdio
server. Protect the gateway from slow or failing upstreams.

## Diagram: credential selection

```mermaid
flowchart TB
    T["resolved tool → upstream"] --> M{"upstream.auth"}
    M -->|"token_exchange"| C{"cached token for<br/>(user, audience)?"}
    C -- "yes, > 60 s left" --> USE[use it]
    C -- no --> EX["RFC 8693 exchange at Keycloak<br/>subject_token = user token<br/>audience = upstream"] --> STORE["cache in Redis<br/>(TTL = exp − 60 s, encrypted)"] --> USE
    M -->|"user_oauth"| UC{"stored user token?"}
    UC -- yes --> USE
    UC -- no --> URL["→ F16: URL-mode consent"]
    M -->|"none (stdio)"| NONE["no token;<br/>isolation by process"]
    USE & NONE --> CALL["upstream call"]
```

## Diagram: upstream call protection

```mermaid
flowchart LR
    CALL["call upstream"] --> CB{"circuit open?"}
    CB -- yes --> FAST["isError: temporarily unavailable"]
    CB -- no --> TO["timeout (default 10 s)<br/>+ traceparent header"]
    TO -->|"ok"| RES[result → filters F12]
    TO -->|"error / timeout"| CNT["failure count++<br/>(5 in 30 s → open 30 s)"] --> FAST
```

## Deliverables / files
```
gateway/src/mcphub/upstream/pool.py        # SDK clients per upstream, connection reuse
gateway/src/mcphub/upstream/breaker.py     # circuit breaker (pure state machine)
gateway/src/mcphub/upstream/stdio.py       # subprocess bridge, restart policy, resource limits
gateway/src/mcphub/credentials/exchange.py # token exchange + Redis cache
gateway/src/mcphub/credentials/vault.py    # envelope encryption for stored tokens
gateway/src/mcphub/router.py               # tools/call path: name → upstream → credential → call
```

## Tasks
- [ ] Router: exposed name → upstream name; pass `_meta` and trace context
- [ ] Token exchange with caching; never forward the client's `Authorization` header (test enforces it)
- [ ] Envelope encryption helper (used for cached and stored tokens)
- [ ] stdio bridge: run the reference `time` server as a subprocess with CPU/memory limits and restart on crash
- [ ] Circuit breaker + per-upstream timeouts; `isError` results on failure
- [ ] **Attack cases:** token passthrough check (upstream sees only exchanged token), replaying a gateway token at an upstream, wrong-audience token at an upstream

## Acceptance criteria
- india-mf-mcp receives tokens with `aud = mf` and `sub = alice`, never the hub token
- Killing an upstream makes only its tools fail; others keep working
- The stdio server's tools work through the gateway like any remote tool

## Tests
- Unit: breaker state machine; credential selection table
- Integration: exchange against Keycloak; stdio bridge restart; header inspection at a test upstream

**Interview talking point:** *"The gateway never forwards the client's token. It exchanges it for a token
scoped to one upstream that still names the user, so upstreams can authorize per user and the audit trail
stays intact."*
