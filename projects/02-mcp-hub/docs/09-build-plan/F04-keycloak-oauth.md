# F4: Keycloak Realm & OAuth Setup

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M2 | Must | F0 | 5 h | F5, F6, F7, F13 |

**Goal:** A reproducible Keycloak realm (exported as JSON) with users, tenants, scopes, audiences and
token exchange, plus a verified answer to "which OAuth features does this Keycloak version support?"

## Diagram: realm layout

```mermaid
flowchart TB
    subgraph Realm["realm: mcp-hub"]
        U["users: alice (tenant demo-retail),<br/>bob (tenant demo-retail), carol (tenant demo-pro)<br/>attribute: tenant"]
        CS["client scopes:<br/>mf:read · fx:read · watchlist:* ·<br/>portfolio:* · gh:read · hub:admin"]
        C1["client: own MCP client<br/>(CIMD URL or pre-registered, public, PKCE)"]
        C2["client: gateway<br/>(confidential, token-exchange permission)"]
        C3["client: admin console<br/>(public SPA, PKCE, hub:admin)"]
        AUD["audience mappers:<br/>hub → https://hub.example/mcp<br/>mf → https://mf.hub.internal<br/>fx → https://fx.hub.internal"]
    end
    C1 --> CS
    C2 --> AUD
    U --> CS
```

## Diagram: verification spike (first 2 hours)

```mermaid
flowchart TB
    S["Spike: Keycloak version X"] --> Q1{"CIMD supported?"}
    Q1 -- yes --> A1["own client uses a CIMD URL"]
    Q1 -- no --> B1["pre-registered public client for own client;<br/>DCR enabled for third-party clients"]
    S --> Q2{"resource indicator (RFC 8707)<br/>sets token aud?"}
    Q2 -- yes --> A2["use resource param"]
    Q2 -- no --> B2["audience via client-scope mapper<br/>+ resource param validated at gateway"]
    S --> Q3{"standard token exchange<br/>with audience?"}
    Q3 -- yes --> A3["gateway exchanges per upstream"]
    Q3 -- no --> B3["STOP: re-plan ADR-006<br/>(e.g. switch authorization server)"]
    A1 & B1 & A2 & B2 & A3 --> ADR["record results in ADR-004/005"]
```

## Deliverables / files
```
infra/keycloak/realm-mcp-hub.json      # exported realm (no secrets; secrets via env)
infra/keycloak/README.md               # how to change and re-export
docs/notes/oauth-spike.md              # what this Keycloak version supports, with evidence
scripts/get_token.py                   # dev helper: auth code + PKCE in a browser
```

## Tasks
- [ ] **Spike** the three questions above and record the answers
- [ ] Realm: users with a `tenant` attribute (mapped into tokens), scopes, clients, audience mappers
- [ ] Token lifetimes: access 10 min, refresh 8 h (demo values)
- [ ] Token-exchange permission only for the gateway client, only to the upstream audiences
- [ ] Issuer consistency: the same `iss` from inside Docker and from the host (set Keycloak's hostname explicitly)
- [ ] Realm import on `docker compose up`

## Acceptance criteria
- `scripts/get_token.py` yields a token with the right `aud`, `scope` and `tenant` claim
- The gateway client can exchange a user token for an `mf` token (`sub` = user); no other client can
- The spike note answers all three questions with evidence (request/response examples)

## Tests
- Integration (testcontainers Keycloak): token contents, token-exchange allowed/denied cases

**Interview talking point:** *"Before designing around CIMD and token exchange, I spent two hours proving
what my authorization server actually supports, and wrote the fallbacks into the ADRs."*
