# 11. Glossary

Plain-English definitions of the terms used in these docs, and where each one shows up in this project.
Terms from earlier projects (MCP, OAuth, pass^k, HITL…) are in the
[Project 2](../../02-mcp-hub/docs/11-glossary.md) and [Project 3](../../03-agent-reliability-harness/docs/11-glossary.md)
glossaries.

## A2A protocol

| Term | Meaning | In this project |
|---|---|---|
| **A2A** (Agent2Agent) | An open standard (Linux Foundation) for agents to discover each other, delegate tasks and exchange results | All agent-to-agent traffic |
| **A2A 1.0** | The first stable version of the spec (2026) | The version targeted |
| **`A2A-Version` header** | Tells the server which protocol version the client speaks; empty means 0.3 | Sent on every request |
| **A2A server / remote agent** | An agent that accepts A2A tasks | Triage, Comms, Postmortem, and the Commander |
| **A2A client** | Code that sends tasks to an A2A server | The Commander, the harness |
| **Agent Card** | A JSON document describing an agent: name, skills, endpoints, auth, capabilities | `/.well-known/agent-card.json` |
| **Skill** | One capability listed on a card, with examples and input/output types | e.g. `triage_incident` |
| **Extended Agent Card** | A fuller card shown only to authenticated callers | Admin-only skills |
| **Signed card** | A card with a JWS signature so clients can check who published it and that it wasn't changed | All cards |
| **JWS / JWKS** | JSON Web Signature / a published set of public keys for verifying signatures | ES256 keys per organization |
| **JCS** (RFC 8785) | A canonical JSON form, so the same card always produces the same bytes to sign | Card signing |
| **Binding** | How A2A is carried: JSON-RPC, HTTP+JSON (REST) or gRPC | JSON-RPC everywhere, REST on Triage |
| **`supportedInterfaces`** | The card's ordered list of bindings and URLs | Clients pick the first one they support |
| **Task** | A unit of work with an ID and a lifecycle | Every delegation |
| **Task states** | submitted, working, input-required, auth-required, completed, failed, canceled, rejected | §3.3 of the LLD |
| **`input-required`** | The agent needs more information to continue | Triage asks a question |
| **`auth-required`** | The agent needs an authorization (e.g. a human approval) to continue | Cross-agent approvals |
| **`contextId`** | Groups related tasks and messages into one conversation | One per incident |
| **Message / Part** | A turn in the conversation, made of parts (text, file, or structured data) | Data parts carry schemas |
| **Artifact** | An output a task produces | `TriageResult`, `DraftUpdate` |
| **Streaming** | Live task updates over Server-Sent Events | Research and comms |
| **`SubscribeToTask`** | Re-attach to a running task's stream | Resume after disconnect |
| **Push notification** | The agent calls the client's webhook when the task changes | Long triage tasks |
| **`returnImmediately`** | Ask the agent to return the task at once instead of waiting for it to finish | Triage delegation |
| **TCK** | The official A2A Technology Compatibility Kit | Conformance tests |
| **A2A Inspector** | A debugging UI for A2A agents | Manual checks |
| **AgentExecutor** | The a2a-sdk class you implement to connect your agent logic to the protocol | Shared wrapper |
| **`RemoteA2aAgent`** | Google ADK's way to use a remote A2A agent as a sub-agent | The Commander's peers |

## Identity and trust

| Term | Meaning | In this project |
|---|---|---|
| **Token exchange** (RFC 8693) | Swapping one token for another with a different audience, keeping the user identity | Commander → each agent |
| **`act` claim** | Records who is acting on the user's behalf (the delegation chain) | `act` = Commander |
| **Audience (`aud`)** | Which service a token is meant for | One per agent |
| **Confused deputy** | A service tricked into using its own authority for someone who shouldn't have it | Attack A6 |
| **Agent registry** | Our service that verifies, pins and allow-lists agents | Only discovery path |
| **Pinning / quarantine** | Recording an approved card's hash, and blocking the agent if it changes | Attack A4 |
| **Rug pull** | An approved component changes its behaviour or description later | Card rug pull |
| **Tenant** | An isolated customer or organization in a shared system | ShopLite (A) and Partner (B) |
| **SSRF** | Tricking a server into calling internal addresses | Push URL validation |

## Orchestration

| Term | Meaning | In this project |
|---|---|---|
| **Orchestrator** | The agent that splits work and delegates it | Incident Commander |
| **Provenance** | A record of which agent produced which part of a result | The final incident report |
| **Cancel propagation** | Cancelling a parent task cancels its children | Incident cancel |
| **Mesh** | A set of independent agents that can call each other through a common protocol | This project |
