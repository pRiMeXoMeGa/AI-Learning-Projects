# 2. High-Level Architecture

## 2.1 Design principles

1. **The sandbox is hostile.** Treat everything that comes out of it as untrusted data: stdout, files,
   chart specs, even error messages. Nothing from inside is executed, rendered as active content, or
   used as instructions outside.
2. **Nothing valuable inside.** No network, no secrets, no user tokens, read-only data. An attacker with
   full control of the sandbox should find nothing to steal and nowhere to send it.
3. **Defence in layers, each tested.** The layers are: the isolation technology (microVM or gVisor), the
   runtime policy (limits, no network), the broker's output filters, the UI's rendering rules, and the
   agent's prompt rules. Every layer has its own attack tests.
4. **Charts are data.** The agent returns a Vega-Lite spec that is schema-validated and rendered by our
   code. It never returns HTML or JavaScript.
5. **One broker, many callers.** The sandbox is a service with an MCP interface, so the web agent, the
   capstone's LangGraph agents and the eval harness all use the same controlled path.
6. **Provider-neutral.** Isolation is compared, not assumed: E2B (Firecracker) and gVisor run the same
   image, limits and tests.

## 2.2 System context (C4 level 1)

```mermaid
flowchart TB
    USER(["👩‍💼 Analyst user"])
    SEC(["🛡️ Security reviewer"])
    CAP(["🤖 Other agents<br/>(capstone Forecaster)"])
    subgraph SYS["Analyst"]
        S["web app + agent + sandbox-broker"]
    end
    E2B["E2B cloud"]
    AZ["Azure VM / Container host<br/>(gVisor runtime)"]
    LLM["Anthropic · OpenAI"]
    LF["Langfuse"]
    USER --> S
    SEC -->|"audit log · escape suite"| S
    CAP -->|"MCP"| S
    S --> E2B & AZ
    S --> LLM
    S -.-> LF
```

## 2.3 Containers (C4 level 2)

```mermaid
flowchart TB
    subgraph Web["web (Next.js 16 on Vercel, P6 shell)"]
        UI["analysis page: chat + notebook panel<br/>Chart (Vega-Lite) · DataTable · AnswerCard"]
        AG["AI SDK 7 ToolLoopAgent<br/>MCP client → broker"]
    end
    subgraph BR["sandbox-broker (Python, Azure Container Apps)"]
        MCPS["FastMCP server (Streamable HTTP, OAuth/JWT)"]
        SES["session manager<br/>create · reuse · idle reap · max lifetime"]
        POL["policy engine<br/>quotas · limits · code pre-checks (advisory)"]
        OUT["output filter<br/>truncate · type allow-list ·<br/>JSON validate · PNG re-encode"]
        AUD[("audit log (Postgres)")]
        PRV["providers: E2BProvider · GVisorProvider"]
    end
    subgraph Host["gVisor host (Azure VM, Docker + runsc)"]
        RUNNER["runner agent (gRPC/HTTP on host,<br/>not reachable from sandboxes)"]
        C1["sandbox container<br/>--runtime=runsc --network=none<br/>read-only rootfs · data ro · tmpfs scratch"]
    end
    E2BC["E2B microVM<br/>(template = same image,<br/>internet access off)"]
    DS[("dataset snapshots<br/>Parquet (blob)")]
    AG -->|"MCP"| MCPS --> SES --> POL --> PRV
    PRV --> E2BC
    PRV --> RUNNER --> C1
    PRV --> OUT --> MCPS
    SES --> AUD
    DS -.->|"copied/mounted read-only<br/>at sandbox start"| E2BC & C1
```

| Container | Responsibility |
|---|---|
| **web** | UI, auth (P6), the agent loop, rendering typed outputs safely |
| **sandbox-broker** | The only component allowed to create sandboxes; MCP interface; quotas, limits, filters, audit |
| **gVisor host** | A dedicated VM running Docker with the `runsc` runtime; a small runner service that the broker calls; sandboxes can't reach it |
| **E2B** | Managed Firecracker microVMs from a custom template with internet access disabled |
| **Dataset store** | Versioned Parquet snapshots; copied into the sandbox read-only at start |

## 2.4 Data flow A: answering a question

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant A as Agent (AI SDK 7)
    participant B as Broker (MCP)
    participant S as Sandbox
    U->>A: "Top 10 products by revenue lost 2010→2011, as a chart"
    A->>B: describe_table(online_retail)
    B-->>A: schema + profile (no raw rows beyond 5-row sample)
    A->>B: run_python(code v1)
    B->>S: create session (if none) → execute with limits
    S-->>B: error: KeyError 'Revenue'
    B-->>A: filtered stderr (truncated)
    A->>B: run_python(code v2: compute revenue = qty × price, exclude returns)
    S-->>B: result.parquet + stdout summary
    B-->>A: outputs (validated), cell record
    A-->>U: stream: NotebookCell ×2 · DataTable · Chart(spec) · AnswerCard + caveats
```

## 2.5 Data flow B: chart rendering (why it's safe)

```mermaid
flowchart LR
    SB["sandbox writes chart.json<br/>(or agent emits spec)"] --> BF["broker: JSON parse ·<br/>size ≤ 1 MB"]
    BF --> AGV["agent/server: validate against<br/>Vega-Lite schema subset<br/>(no external URLs, no 'expr' with functions<br/>outside allow-list, data inline)"]
    AGV --> UI["client: vega-embed with<br/>loader disabled for URLs,<br/>expression interpreter (CSP-safe)"]
    X["❌ HTML / JS / SVG with scripts<br/>from the sandbox"] -.->|"never rendered"| UI
```

Vega-Lite specs can contain expressions and data URLs. Our validator therefore allows only:
- a subset of the schema;
- inline data;
- no remote loaders.

The client renders with Vega's **CSP-safe expression interpreter**, so no `eval` is needed.

## 2.6 Data flow C: another agent using the broker

```mermaid
sequenceDiagram
    autonumber
    participant L as LangGraph agent (capstone)
    participant B as Broker MCP
    participant S as Sandbox
    L->>B: OAuth token (P2 Keycloak) → tools/list
    L->>B: run_python("fit ETS on series, write forecast.parquet")
    B->>S: execute (same limits, audited with the caller identity)
    S-->>B: forecast.parquet
    B-->>L: structured result + file handle
```

## 2.7 Deployment view

```mermaid
flowchart TB
    subgraph Vercel
        W["web (Next.js)"]
    end
    subgraph Azure
        ACA["Container Apps: sandbox-broker"]
        VM["VM (Standard D4s): Docker + gVisor runsc<br/>+ runner service (private network only)"]
        PG[("Postgres: audit, sessions")]
        BLOB[("Blob: dataset snapshots")]
    end
    E2B["E2B (managed)"]
    W -->|"MCP over HTTPS"| ACA
    ACA -->|"private VNet"| VM
    ACA --> E2B
    ACA --> PG
    ACA --> BLOB
```

The gVisor VM doesn't need nested virtualization (gVisor's default `systrap` platform doesn't use
KVM). It runs on its own VM, never on the same host as the broker or database, so a full escape
lands on a machine with nothing else on it ([ADR-004](07-decisions.md)).

## 2.8 Proposed repository layout

```
analyst/
├─ apps/web/                  # Next.js (P6 shell packages): analysis pages, agent, renderers
├─ services/broker/           # FastAPI + FastMCP: sessions, policy, filters, providers, audit
│  └─ providers/e2b.py  providers/gvisor.py
├─ services/runner/           # tiny service on the gVisor host (creates/destroys runsc containers)
├─ sandbox-image/             # Dockerfile + E2B template (same packages, no network tools)
├─ datasets/                  # snapshot builder, schema docs, profiles
├─ evals/                     # AnalystBench-50, DABstep runner, provider bench
├─ redteam/                   # escape/abuse suite, CSV-injection cases
├─ infra/                     # Terraform: VM + broker + storage
└─ docs/
```
