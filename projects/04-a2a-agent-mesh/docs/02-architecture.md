# 2. High-Level Architecture

## 2.1 Design principles

1. **Reuse the agents, add the protocol.** The agents from Project 3 keep their logic. A thin
   `AgentExecutor` wrapper turns each into an A2A server, so the interesting work is the protocol, the
   orchestration and the trust model.
2. **MCP down, A2A across.** Each agent reaches its tools through MCP. Agents reach each other only through
   A2A. No agent calls another agent's tools directly.
3. **Nothing trusted by default.** Only agents in the registry, with a signed card and a pinned version,
   can be called. Another agent's output is data, never instructions.
4. **Credentials never travel down the chain.** Each hop gets a fresh, audience-restricted token by token
   exchange. Approval credentials go straight to the agent that needs them.
5. **Long-running by default.** Every delegation is a task that can stream, push, be cancelled and be
   resumed.
6. **One trace across all hops.** W3C trace context goes through every A2A call.

## 2.2 System context (C4 level 1)

```mermaid
flowchart TB
    ENG(["🧑‍🚒 On-call engineer"])
    ADM(["🛡️ Platform admin"])
    PART(["🏢 Partner org's agent<br/>(simulated tenant B)"])
    subgraph SYS["A2A Agent Mesh"]
        M["Commander + remote agents + registry"]
    end
    P3["Project 3 services<br/>OpsSim · approval service"]
    KC["Keycloak (Project 2)"]
    LLM["Model APIs<br/>Gemini · Anthropic · OpenAI"]
    LF["Langfuse"]
    ENG -->|"A2A / CLI"| M
    ENG -->|"approve"| P3
    ADM -->|"approve cards, allow-lists"| M
    PART -->|"A2A (own credentials)"| M
    M --> P3
    M -.->|"tokens"| KC
    M --> LLM
    M -.->|"OTLP"| LF
```

## 2.3 Containers (C4 level 2)

```mermaid
flowchart TB
    subgraph Commander["commander (Python, Google ADK)"]
        IC["ADK root agent<br/>+ RemoteA2aAgent per peer"]
        ICS["A2A server facade<br/>(tasks from users / partners)"]
        TRK["task tracker<br/>stream · push receiver · resume"]
    end
    subgraph Agents["remote agents (one container each)"]
        TR["triage-agent<br/>LangGraph + a2a-sdk executor<br/>JSON-RPC + REST"]
        CM["comms-agent<br/>OpenAI Agents SDK + a2a-sdk executor"]
        PM["postmortem-agent<br/>Claude Agent SDK + a2a-sdk executor"]
    end
    subgraph Reg["registry (FastAPI)"]
        RG["card fetch · JWS verify ·<br/>pin · quarantine · allow-lists"]
    end
    subgraph Tools["MCP servers"]
        OPS["opsdesk-mcp (P3)"]
        KB["postmortem-kb-mcp<br/>(P1 retrieval over<br/>~60 synthetic postmortems)"]
    end
    APR["approval service (P3)"]
    KC["Keycloak (P2)"]
    PG[("Postgres<br/>A2A task stores · registry ·<br/>audit · checkpoints")]
    LF["Langfuse"]

    ICS --> IC --> TRK
    IC -->|"list agents"| RG
    TRK -->|"A2A JSON-RPC / REST + SSE"| TR & CM & PM
    TR -->|"push webhook"| TRK
    TR & CM --> OPS
    PM --> KB
    TR & CM --> APR
    IC & TR & CM & PM -.->|"token exchange / validate"| KC
    TR & CM & PM & ICS & RG --> PG
    IC & TR & CM & PM -.->|"OTLP"| LF
```

| Container | Responsibility |
|---|---|
| **commander** | Planning, delegation, task tracking, approval relay, final report; also an A2A server for users and partners |
| **triage-agent** | Project 3's LangGraph agent behind A2A; long tasks with push; approvals via `auth-required` |
| **comms-agent** | Drafts updates from structured findings; publishing needs approval |
| **postmortem-agent** | Read-only research over past postmortems; fast, streaming |
| **registry** | The only source of agent addresses; signature checks, pinning, quarantine, allow-lists |
| **postmortem-kb-mcp** | Hybrid search over a synthetic postmortem corpus (Project 1 code) |

## 2.4 Inside a remote agent: the A2A wrapper

```mermaid
flowchart LR
    REQ["A2A request<br/>(JSON-RPC or REST)"] --> AU["auth middleware<br/>JWT: iss · aud=this agent ·<br/>sub · act · tenant"]
    AU --> RH["a2a-sdk request handler<br/>(DefaultRequestHandler)"]
    RH --> TS[("DatabaseTaskStore<br/>(Postgres, tenant column)")]
    RH --> EX["AgentExecutor<br/>(our wrapper)"]
    EX --> IN["input guard:<br/>schema check · spotlight text ·<br/>size limits"]
    IN --> FW["framework agent<br/>(LangGraph / OpenAI SDK / Claude SDK)"]
    FW --> MAP["event mapper:<br/>framework events → TaskStatusUpdate /<br/>TaskArtifactUpdate"]
    MAP --> Q["event queue → SSE stream<br/>or push sender"]
```

The same wrapper pattern works for all three frameworks. Only the `framework agent` box and the event
mapper differ, and they reuse Project 3's adapters and normalized events.

## 2.5 Data flow A: an incident through the mesh

```mermaid
sequenceDiagram
    autonumber
    participant U as Engineer
    participant IC as Commander
    participant RG as Registry
    participant KC as Keycloak
    participant PM as Postmortem agent
    participant TR as Triage agent
    participant CM as Comms agent
    U->>IC: SendStreamingMessage("checkout p95 alert ALR-7781")
    IC->>RG: agents for tenant A with skills [triage, research, comms]
    RG-->>IC: pinned, verified cards
    IC->>KC: token exchange (sub=engineer, act=commander, aud=triage)
    IC->>KC: token exchange (aud=postmortem)
    par research
        IC->>PM: SendStreamingMessage(find_similar_incidents, contextId=INC-7)
        PM-->>IC: artifact SimilarIncidents (2 past incidents, rollback worked)
    and triage
        IC->>TR: SendMessage(triage_incident, returnImmediately, push config)
        TR-->>IC: Task(submitted)
        TR-->>IC: push: working · findings · auth-required · completed
    end
    IC->>KC: token exchange (aud=comms)
    IC->>CM: SendMessage(draft_stakeholder_update, data part: TriageResult)
    CM-->>IC: artifact DraftUpdate
    IC-->>U: final report (with per-agent provenance)
```

## 2.6 Data flow B: identity through the hops

```mermaid
flowchart LR
    E["engineer logs in<br/>token T0: sub=eng, aud=commander"] --> IC["Commander"]
    IC -->|"exchange T0 →<br/>T1: sub=eng, act=commander,<br/>aud=triage, scope=triage:run"| TR["Triage"]
    TR -->|"MCP call with its own<br/>client token (aud=opsdesk)<br/>+ user in _meta for audit"| OPS["opsdesk-mcp"]
    IC -->|"T2: sub=eng, act=commander,<br/>aud=comms"| CM["Comms"]
    X["❌ Triage re-using T1<br/>at Comms"] -.->|"aud mismatch → 401"| CM
```

Every hop checks `aud` (it is the intended recipient), `sub` (whom it acts for) and `act` (who is calling).
The audit log records the full chain.

## 2.7 Data flow C: card registration, pinning and a rug pull

```mermaid
sequenceDiagram
    autonumber
    participant AD as Admin
    participant RG as Registry
    participant AG as Agent (host)
    participant IC as Commander
    AD->>RG: register https://comms.local
    RG->>AG: GET /.well-known/agent-card.json
    RG->>RG: verify JWS (JCS-canonical card, key from org JWKS)
    RG-->>AD: card diff: new skills, security schemes
    AD->>RG: approve → pin(card hash, version)
    Note over RG: later, background refresh (ETag)
    AG-->>RG: card changed: new skill "export_all_customer_data"
    RG->>RG: hash ≠ pinned → QUARANTINE, audit event
    IC->>RG: list agents
    RG-->>IC: comms-agent excluded (quarantined)
```

## 2.8 Data flow D: Commander crash and resume

```mermaid
sequenceDiagram
    autonumber
    participant IC1 as Commander (before crash)
    participant DB as Commander task store
    participant IC2 as Commander (restarted)
    participant TR as Triage agent
    IC1->>TR: SendMessage → task T-55 (working)
    IC1->>DB: save child task ids for INC-7
    Note over IC1: crash
    IC2->>DB: load open incidents + child tasks
    IC2->>TR: GetTask(T-55) → status + history
    IC2->>TR: SubscribeToTask(T-55) → resume stream
    TR-->>IC2: remaining events → completed
```

## 2.9 Deployment view

```mermaid
flowchart TB
    subgraph Compose["docker compose (local / CI)"]
        C1["commander"]
        C2["triage-agent"]
        C3["comms-agent"]
        C4["postmortem-agent"]
        C5["registry"]
        C6["postmortem-kb-mcp"]
        P3S["opssim + approvals (P3 images)"]
        KCC["keycloak (P2 realm + agent clients)"]
        PGC[("postgres + pgvector")]
        LFC["langfuse"]
        ROG["rogue agents (profile: security)"]
    end
```

Each agent is its own container on its own hostname, as it would be across teams. Azure deployment is
optional (the build plan decides). If it's included, it reuses the Container Apps modules.

## 2.10 Proposed repository layout

```
a2a-agent-mesh/
├─ commander/          # ADK agent, A2A server facade, task tracker, push receiver
├─ agents/
│  ├─ common/          # AgentExecutor base, auth middleware, input guard, event mapper, card signing
│  ├─ triage/          # wraps P3 LangGraph agent (installed as a package)
│  ├─ comms/           # OpenAI Agents SDK
│  └─ postmortem/      # Claude Agent SDK
├─ registry/           # FastAPI: cards, JWS verify, pinning, quarantine, allow-lists
├─ kb/                 # postmortem corpus + postmortem-kb-mcp
├─ rogue/              # test-fixture agents for security evals
├─ evals/              # interop/TCK, mesh scenarios, security, resilience, mcp_vs_a2a
├─ keycloak/           # realm additions (agent clients, token-exchange permissions)
├─ reports/
└─ docs/
```
