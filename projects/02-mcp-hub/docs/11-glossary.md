# 11. Glossary

Plain-English definitions of the terms used in these docs, and where each one shows up in this project.

## MCP protocol

| Term | Meaning | In this project |
|---|---|---|
| **MCP** (Model Context Protocol) | An open standard for connecting AI applications to tools and data | Everything here speaks it |
| **MCP server** | A program that offers tools, resources and prompts to AI clients | india-mf-mcp, fx-rates-mcp |
| **MCP client / host** | The AI application that connects to servers and lets a model use their tools | Claude Desktop, VS Code, the own client (C4) |
| **Tool** | A function a model can call, with a name, description and JSON input schema | e.g. `compute_returns` |
| **Output schema / structured content** | A JSON schema for a tool's result, and the result as typed JSON (not just text) | Every read tool returns structured content |
| **Tool annotations** | Hints about a tool, e.g. read-only or destructive | Used by the gateway's risk rules |
| **Resource / resource template** | Read-only data a client can fetch by URI | `mf://scheme/{code}` |
| **Prompt (MCP)** | A reusable prompt template a server offers | `analyze_fund` |
| **2026-07-28 specification** | The MCP protocol version this project targets: stateless core, MRTR, routing headers, tighter OAuth | [ADR-002](07-decisions.md) |
| **Stateless protocol** | Each request carries everything needed; no session or handshake | Any gateway replica can serve any request |
| **`server/discover`** | A request that returns a server's supported versions, capabilities and identity | First call from a client |
| **`Mcp-Method` / `Mcp-Name` headers** | HTTP headers naming the operation and tool, so infrastructure can route without reading the body | Gateway routing and rate limits |
| **Streamable HTTP** | MCP's HTTP transport: POST requests, with an optional event stream for streamed responses | All remote endpoints |
| **stdio transport** | Running a server as a local process that talks over standard input/output | Local use of the open-source servers; the stdio bridge |
| **MRTR** (multi round-trip request) | A server answers "input required" with what it needs; the client retries the same call with the answers | Disambiguation and confirmations |
| **Elicitation** | Asking the user for information during a tool call, via a **form** or a **URL** | Forms for choices/confirmations; URL mode for GitHub consent |
| **`requestState`** | Opaque state the client must send back on the retry | Signed by the gateway so it can't be forged |
| **Tasks extension** | A standard way to run long jobs and poll for results | `sip_backtest_batch` |
| **MCP Apps** | An extension for server-provided interactive UI | Optional NAV chart |
| **MCP Registry** | The official public list of MCP servers, with `server.json` metadata | Where india-mf-mcp is published |
| **Deprecated features** | Sampling, roots, logging and the legacy SSE transport, which new code shouldn't use | Not used ([ADR-013](07-decisions.md)) |

## Identity and OAuth

| Term | Meaning | In this project |
|---|---|---|
| **OAuth 2.1** | The standard for giving an app limited access on a user's behalf | Client → gateway login |
| **Authorization server** | The service that logs users in and issues tokens | Keycloak |
| **Resource server** | A service that accepts tokens and protects data | The gateway and each MCP server |
| **Access token / JWT** | A signed, short-lived credential listing who the user is and what's allowed | Validated on every request |
| **Scope** | A named permission inside a token | `mf:read`, `portfolio:write`… |
| **Audience (`aud`)** | Which service a token is meant for | Tokens are valid for exactly one service |
| **Resource indicator (RFC 8707)** | Asking for a token for a specific service | Binds the client's token to the gateway |
| **Protected Resource Metadata (RFC 9728)** | A JSON file where a resource server says which authorization server to use | `/.well-known/oauth-protected-resource` |
| **PKCE** | A one-time secret that stops stolen authorization codes being used | Every login |
| **CIMD** (Client ID Metadata Document) | The client's ID is a URL to a public JSON document describing it, so no registration step is needed | The own client ([ADR-005](07-decisions.md)) |
| **DCR** (Dynamic Client Registration) | The older way for clients to register themselves; deprecated in 2026-07-28 | Fallback only |
| **`iss` validation (RFC 9207)** | The client checks which server sent the authorization response | Stops authorization-server mix-up |
| **Step-up authorization** | Asking for more permissions only when needed (`403 insufficient_scope`) | `portfolio:write` |
| **Token passthrough** | Forwarding a user's token to another service; forbidden in MCP | Never done ([ADR-006](07-decisions.md)) |
| **Token exchange (RFC 8693)** | Trading one token for another meant for a different service, keeping the user's identity | Gateway → each upstream |
| **Confused deputy** | A trusted service tricked into using its own authority for someone else | Prevented by per-user tokens + row-level security |

## Gateway and security

| Term | Meaning | In this project |
|---|---|---|
| **MCP gateway** | A single endpoint in front of many MCP servers that applies auth, policy and logging | C3 |
| **Upstream server** | A server behind the gateway | mf, fx, ref, gh |
| **Tool registry** | The gateway's list of known tools and their approval status | `tool_definition` table |
| **Definition pinning** | Storing a hash of an approved tool definition and rejecting any change until re-approved | Rug-pull defence ([ADR-009](07-decisions.md)) |
| **Tool poisoning** | Hidden instructions inside a tool's description | Security eval category |
| **Rug pull** | A server changing a tool after it was approved | Security eval category |
| **Tool shadowing** | A malicious tool using the same name as a trusted one | Prevented by `server__tool` names |
| **Indirect prompt injection** | Instructions hidden in data the model reads (e.g. a tool result) | Result filters + confirmations |
| **Policy engine / OPA / Rego** | A separate service that answers "is this allowed?"; Rego is its policy language | Allow / deny / confirm / step-up decisions |
| **PEP / PDP** | Policy Enforcement Point (applies decisions) / Policy Decision Point (makes them) | Gateway / OPA |
| **Row-level security (RLS)** | Postgres rules that filter rows by user automatically | Holdings and watchlists |
| **Hash-chained audit log** | Each log entry includes the hash of the previous one, so edits are detectable | `audit_event` |
| **HMAC** | A signature made with a secret key, proving data wasn't changed | Signs `requestState` |
| **Nonce** | A value that may be used only once | Stops confirmation replay |
| **Circuit breaker** | Stops calling a failing service for a while | Per upstream |
| **STRIDE** | A threat checklist: spoofing, tampering, repudiation, information disclosure, denial of service, elevation of privilege | [05 §5.4](05-security-threat-model.md) |
| **Attack success rate (ASR)** | Share of attacks that achieved their goal | Headline security metric |

## Mutual-fund domain

| Term | Meaning | In this project |
|---|---|---|
| **AMFI** | Association of Mutual Funds in India; publishes daily NAVs for all schemes | Data source |
| **Scheme code** | AMFI's numeric ID for one scheme variant | Primary key |
| **NAV** | Net Asset Value: the price of one unit of a fund on a date | `mf.nav` |
| **Direct / Regular plan** | Same fund, bought without / with a distributor (different costs and NAVs) | A search filter |
| **Growth / IDCW option** | Reinvest gains vs. pay them out | A search filter |
| **CAGR** | Compound annual growth rate between two dates | `compute_returns` |
| **SIP** | Systematic Investment Plan: investing a fixed amount every month | `sip_backtest` |
| **XIRR** | The annual return of a series of cash flows at different dates | SIP and portfolio returns |
| **Max drawdown** | The largest fall from a peak to a later low | `compare_schemes` |
| **ELSS** | A tax-saving equity fund category | Example category |

## Evaluation

| Term | Meaning | In this project |
|---|---|---|
| **Toolset variant** | One version of the tool design (granularity, descriptions, schemas) | T1–T5 |
| **Task success** | The agent produced the right result and final state | Main tool-design metric |
| **pass^k** | The task succeeded in all k repeated runs (a reliability measure) | pass^3 |
| **Frozen snapshot** | A fixed copy of the data so expected answers never change | Eval database |
| **User simulator** | Scripted answers to forms, so interactive tasks are reproducible | "Needs user input" tasks |
| **Paired bootstrap** | Comparing two variants on the same tasks with confidence intervals | Reused from Project 1 |
| **False-positive rate** | Normal tasks wrongly blocked by a defence | Security report |
| **Gateway overhead** | Time the gateway adds on top of the upstream server's time | p95 ≤ 25 ms |
