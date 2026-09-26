# 2. High-Level Architecture

## 2.1 Design principles

1. **Current protocol first.** Everything speaks MCP **2026-07-28**: stateless requests, `server/discover`,
   multi round-trip requests (MRTR) for user input, and `Mcp-Method` / `Mcp-Name` routing headers. Older
   clients are handled by version negotiation, not by designing around the old protocol.
2. **Stateless everywhere.** No gateway replica holds per-client state. Anything that must survive between
   two requests (a pending confirmation, for example) travels **signed** inside the request itself.
3. **The gateway is a resource server, never an identity provider.** Keycloak issues tokens; the gateway
   and servers only validate them. Tokens are never passed through to upstream servers.
4. **Default deny.** A tool is invisible and uncallable until its definition is approved *and* a policy
   allows it for the caller's tenant.
5. **Everything is audited and traced.** Every call produces an audit event and an OpenTelemetry trace that
   spans client → gateway → upstream server.
6. **Measure design choices.** Tool granularity, descriptions and defences are compared with evals, not
   chosen by taste.

## 2.2 System context (C4 level 1)

```mermaid
flowchart TB
    User([Investor / analyst])
    Admin([Platform admin])
    Dev([Developer])
    subgraph Clients["MCP clients"]
        TP["Third-party clients<br/>Claude Desktop · Claude Code · VS Code"]
        OWN["Own client (C4)"]
    end
    subgraph Hub["MCP Hub (this project)"]
        GW[MCP Gateway]
        SRV["Own MCP servers<br/>india-mf-mcp · fx-rates-mcp"]
        CON[Admin console]
    end
    KC["Keycloak<br/>(authorization server)"]
    AMFI[("AMFI<br/>daily NAV files")]
    ECB[("ECB<br/>reference FX rates")]
    GHM["GitHub remote<br/>MCP server"]
    LLM["LLM APIs<br/>Claude · OpenAI"]
    OBS["Grafana LGTM<br/>traces · metrics · logs"]
    REG["Official MCP Registry<br/>PyPI · npm"]

    User --> TP & OWN
    TP & OWN -->|"MCP over HTTPS + OAuth"| GW
    OWN --> LLM
    Admin --> CON --> GW
    GW --> SRV
    GW -->|"optional"| GHM
    TP & OWN & GW & SRV -.->|"OIDC / JWKS / token exchange"| KC
    SRV -->|"daily ingest"| AMFI & ECB
    GW & SRV -.->|OTel| OBS
    Dev -->|"publish"| REG
```

## 2.3 Containers (C4 level 2)

```mermaid
flowchart TB
    subgraph Edge
        LB["Load balancer / ingress<br/>(round-robin, TLS)"]
    end
    subgraph GWC["Gateway (N stateless replicas)"]
        GWA["gateway<br/>Python · MCP SDK v2 · Starlette"]
    end
    subgraph Policy
        OPA["OPA sidecar<br/>policy bundle (Rego)"]
    end
    subgraph Upstreams
        MFS["india-mf-mcp<br/>FastMCP 4 · Streamable HTTP"]
        FXS["fx-rates-mcp<br/>TS SDK · Streamable HTTP"]
        BR["stdio bridge<br/>(runs a stdio server as a subprocess)"]
    end
    subgraph Jobs
        ING["ingest worker<br/>AMFI + ECB daily jobs"]
    end
    subgraph Data
        PG[("PostgreSQL 16<br/>schemas: mf · gateway · audit")]
        RD[("Redis 7<br/>rate limits · JWKS · token cache")]
    end
    KC["Keycloak<br/>(+ its own Postgres DB)"]
    CON["Admin console<br/>React + TS (Vite)"]
    OT["OTel Collector → Grafana LGTM"]

    LB --> GWA
    GWA --> OPA
    GWA --> MFS & FXS & BR
    GWA --> PG & RD
    GWA -.-> KC
    MFS --> PG
    FXS --> PG
    ING --> PG
    CON -->|"admin API (OIDC)"| GWA
    GWA & MFS & FXS -.-> OT
```

| Container | Responsibility | Tech |
|---|---|---|
| **Gateway** | MCP endpoint for clients; token validation; aggregation; routing; registry; policy calls; confirmations; token exchange; filters; audit; admin API | Python 3.12, MCP Python SDK v2, Starlette/Uvicorn, httpx |
| **OPA** | Evaluates policy decisions from a versioned Rego bundle | Open Policy Agent (sidecar) |
| **india-mf-mcp** | MF tools, resources, prompts; per-user watchlists and holdings | Python, FastMCP 4, SQLAlchemy, Postgres |
| **fx-rates-mcp** | FX tools (rates, conversion, history) | TypeScript, MCP TypeScript SDK v2, Node 22 |
| **stdio bridge** | Runs a stdio-only server as a subprocess and exposes it to the gateway | Part of the gateway package |
| **Ingest worker** | Daily AMFI NAV + ECB rates jobs, history backfill | Python, scheduled job (cron in compose / Container Apps job) |
| **PostgreSQL** | `mf` (funds, NAVs, user data), `gateway` (servers, tools, tenants, credentials), `audit` (events) | Postgres 16, partitioned NAV table |
| **Redis** | Rate-limit buckets, JWKS cache, upstream-token cache (short TTL) | Redis 7 |
| **Keycloak** | OAuth 2.1 / OIDC authorization server, token exchange | Keycloak (Docker) |
| **Admin console** | Tool approvals, tenant policies, audit viewer | React, TypeScript, Vite, TanStack Query |
| **Observability** | Traces, metrics, logs in one container for local dev | OTel Collector + `grafana/otel-lgtm` |

## 2.4 Module structure (gateway)

```mermaid
flowchart LR
    subgraph gateway["gateway package"]
        direction TB
        EDGE["edge<br/>HTTP · header checks · version negotiation"]
        AUTHN["authn<br/>JWT validation · PRM · challenges"]
        RL["ratelimit<br/>Redis token buckets"]
        REGY["registry<br/>upstreams · tool pinning · aggregation"]
        POL["policy<br/>OPA client · decision cache"]
        CONF["confirm<br/>MRTR · signed requestState"]
        CRED["credentials<br/>token exchange · upstream OAuth"]
        UP["upstream<br/>MCP client pool · stdio bridge"]
        FIL["filters<br/>size · schema · injection · PII"]
        AUD["audit<br/>hash-chained events"]
        ADM["admin API"]
        OBS["telemetry<br/>OTel spans · metrics"]
    end
    EDGE --> AUTHN --> RL --> REGY --> POL --> CONF --> CRED --> UP --> FIL --> AUD
```

Each request passes through the modules **in this order**. Every module can end the request early (for
example `authn` with a `401`, `policy` with a deny), and every exit path still writes an audit event.

## 2.5 Data flow A: first connection (OAuth discovery, CIMD, PKCE)

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant C as MCP client
    participant GW as Gateway (resource server)
    participant KC as Keycloak (authorization server)

    C->>GW: POST /mcp  (Mcp-Method: server/discover), no token
    GW-->>C: 401 WWW-Authenticate: Bearer resource_metadata=".../.well-known/oauth-protected-resource"
    C->>GW: GET /.well-known/oauth-protected-resource
    GW-->>C: {resource: "https://hub.example/mcp", authorization_servers: [KC], scopes_supported}
    C->>KC: GET /.well-known/oauth-authorization-server (or OIDC discovery)
    KC-->>C: endpoints, supports client_id_metadata_document
    Note over C: client_id = HTTPS URL of the client's<br/>metadata document (CIMD)
    C->>U: open browser: /authorize?client_id=<CIMD URL>&code_challenge=…&resource=https://hub.example/mcp&scope=mf:read
    KC->>KC: fetch + validate the CIMD document (redirect URIs)
    U->>KC: log in + consent
    KC-->>C: redirect with code + iss
    C->>C: check iss == expected issuer (RFC 9207)
    C->>KC: POST /token (code, code_verifier, resource)
    KC-->>C: access token (aud = hub.example/mcp, scope = mf:read)
    C->>GW: POST /mcp (Bearer token, Mcp-Method: server/discover)
    GW-->>C: protocol versions, capabilities, server identity
```

**Key points**
- **CIMD** (Client ID Metadata Documents): the client's ID is a URL to a JSON document it hosts, so no
  registration step is needed. Dynamic Client Registration is deprecated in this spec version; it's kept
  only as a fallback if the authorization server doesn't support CIMD.
- The `resource` parameter (RFC 8707) makes the token **valid only for the gateway**. A token stolen from
  the gateway can't be replayed against another service.
- Step-up: if a later call needs a scope the token doesn't have (e.g. `portfolio:write`), the gateway
  answers `403` with `error="insufficient_scope"` and the required scope, and the client re-authorizes
  with the **union** of old and new scopes.

## 2.6 Data flow B: a tool call through the gateway

```mermaid
sequenceDiagram
    autonumber
    participant C as MCP client
    participant GW as Gateway
    participant R as Redis
    participant P as OPA
    participant KC as Keycloak
    participant MF as india-mf-mcp
    participant A as Audit (Postgres)

    C->>GW: POST /mcp  Mcp-Method: tools/call · Mcp-Name: mf__compare_schemes<br/>Bearer token · body {name, arguments, _meta}
    GW->>GW: validate JWT (sig via cached JWKS, iss, aud, exp, scope)
    GW->>GW: header/body match check
    GW->>R: rate-limit bucket (user, tool)
    GW->>GW: registry: mf__compare_schemes → upstream "mf", pinned hash ✓ approved
    GW->>P: decide {tenant, user, scopes, tool, arg summary, risk tags}
    P-->>GW: allow
    GW->>R: cached upstream token for (user, aud=mf)?
    alt cache miss
        GW->>KC: token exchange (subject_token = user token, audience = mf)
        KC-->>GW: token for mf (sub = user, act = gateway)
    end
    GW->>MF: tools/call compare_schemes (Bearer = exchanged token)
    MF-->>GW: result {content, structuredContent}
    GW->>GW: filters: size cap · outputSchema check · injection heuristics · PII
    GW->>A: audit event (hash-chained)
    GW-->>C: result
```

## 2.7 Data flow C: confirmation with multi round-trip requests (stateless)

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant C as MCP client
    participant GW as Gateway (any replica)
    participant MF as india-mf-mcp

    C->>GW: tools/call mf__portfolio_remove_holding {holding_id: 42}
    GW->>GW: policy → require_confirmation
    GW-->>C: resultType: "input_required"<br/>inputRequests: [confirm form: "Remove 120.5 units of Fund X?"]<br/>requestState: signed{sub, tool, args_hash, exp, nonce}
    C->>U: show confirmation form
    U-->>C: confirm = true
    C->>GW: same tools/call + inputResponses + requestState  (may reach a different replica)
    GW->>GW: verify HMAC, expiry, sub, args_hash, unused nonce
    GW->>MF: tools/call portfolio_remove_holding
    MF-->>GW: result
    GW-->>C: result
```

- The pending confirmation isn't stored anywhere on the server. It's the **signed `requestState`**, bound
  to the user, the tool and a hash of the exact arguments, with a short expiry. Changing the arguments or
  replaying it for another user fails verification.
- The only shared state is a short-lived **nonce set** in Redis to stop the same confirmation being used
  twice.
- The upstream server can also return `input_required` itself (e.g. `india-mf-mcp` asking "which of these
  3 funds did you mean?"). The gateway passes such requests through, adding its own signature layer
  around the upstream's `requestState` so the upstream state can't be tampered with.

## 2.8 Data flow D: aggregated `tools/list`

```mermaid
flowchart LR
    C["client: tools/list"] --> AUTH["validate token"]
    AUTH --> CACHE{"per-tenant list<br/>in cache (ttl)?"}
    CACHE -- yes --> F
    CACHE -- no --> REG["registry: approved tool<br/>definitions (pinned hashes)"]
    REG --> NS["namespace names<br/>mf__search_schemes · fx__convert"]
    NS --> F["filter by policy:<br/>tenant allow-list + user scopes"]
    F --> ORD["stable ordering<br/>(keeps LLM prompt caches warm)"]
    ORD --> OUT["result + ttlMs + cache scope"]
```

The gateway **doesn't** fetch upstream `tools/list` on every client request. A background refresher polls
each upstream (respecting its cache hints), hashes every tool definition, and compares it with the
approved hash. A changed definition goes to **pending review** and disappears from client lists until an
admin approves it (see [05 Security](05-security-threat-model.md)).

## 2.9 Data flow E: ingestion (india-mf-mcp)

```mermaid
flowchart LR
    CRON["daily schedule<br/>(after AMFI publishes)"] --> DL["download NAV file"]
    DL --> HASH{"content hash<br/>changed?"}
    HASH -- no --> SKIP[skip]
    HASH -- yes --> PARSE["parse lines → schemes + NAVs<br/>(validate dates, numbers)"]
    PARSE --> UPS["upsert amc, scheme<br/>insert nav (partitioned by year)"]
    UPS --> MV["refresh derived data<br/>(latest NAV, returns cache)"]
    BF["one-off history backfill<br/>(date ranges, polite rate)"] --> PARSE
    FXJ["daily ECB rates job"] --> FXT[("fx.rate")]
```

## 2.10 Deployment view

**Local (development and CI)**

```mermaid
flowchart LR
    subgraph compose["docker compose"]
        gw1[gateway ×2]
        lb[nginx round-robin]
        opa[opa]
        mf[india-mf-mcp]
        fx[fx-rates-mcp]
        kc[keycloak]
        pg[(postgres)]
        rd[(redis)]
        lgtm[otel-lgtm]
        con[admin console]
    end
    lb --> gw1
    gw1 --> opa & mf & fx & pg & rd & kc
    mf & fx --> pg
    con --> lb
    gw1 & mf & fx -.-> lgtm
```

Two gateway replicas behind nginx round-robin run **locally from day one**, so statelessness bugs show up
immediately rather than in production.

**Cloud (Azure, consistent with Project 1)**

```mermaid
flowchart LR
    GHA[GitHub Actions] --> ACR[Container Registry]
    ACR --> GWA["Container Apps: gateway<br/>(1–3 replicas, no session affinity)"]
    ACR --> MFA[Container Apps: india-mf-mcp]
    ACR --> FXA[Container Apps: fx-rates-mcp]
    ACR --> KCA[Container Apps: keycloak]
    ACR --> ING[Container Apps job: ingest]
    GWA --> PGF[(Postgres Flexible Server)]
    GWA --> RC[(Azure Cache for Redis)]
    MFA --> PGF
    GWA --> KV[Key Vault]
    GWA -.-> MON[Azure Monitor / OTel]
```

The public `india-mf-mcp` endpoint for open-source users is the same container, reachable directly for
anonymous read-only use (strict rate limits), while signed-in features go through the gateway.

## 2.11 Proposed repository layout

```
02-mcp-hub/
├── docs/                         # these design docs
├── servers/
│   ├── india-mf-mcp/             # Python, FastMCP 4 (published to PyPI + MCP Registry)
│   └── fx-rates-mcp/             # TypeScript, MCP TS SDK (published to npm + MCP Registry)
├── gateway/                      # Python package: edge, authn, registry, policy, confirm, audit…
├── policies/                     # Rego policy bundle + tests (opa test)
├── client/                       # own MCP client (Claude + OpenAI adapters), CLI
├── console/                      # React + TS admin console
├── evals/
│   ├── tool_design/              # tasks, toolset variants, runner, report
│   ├── security/                 # attack cases, rogue test servers, report
│   └── perf/                     # k6 scripts
├── infra/                        # Terraform (Azure), Keycloak realm export
├── docker-compose.yml
└── .github/workflows/
```
