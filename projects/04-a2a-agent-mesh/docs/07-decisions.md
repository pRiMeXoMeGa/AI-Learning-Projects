# 7. Architecture Decision Records

Format: **Context → Decision → Alternatives → Consequences.** Status is *Proposed* until validated while
building.

---

### ADR-001: An incident-response mesh built on Project 3's agents
- **Context:** A2A only matters when there are several independent agents with real work to delegate.
  Building new agents from scratch would duplicate Project 3.
- **Decision:** Reuse OpsSim, the approval service and the Triage Agent from Project 3. Add two small agents
  (Comms, Postmortem Research) and an orchestrator (the Commander).
- **Alternatives:** A travel/booking demo (the standard A2A sample domain, overdone); a CPG planning mesh
  (saved for the capstone).
- **Consequences:**
  - Outcomes are graded with Project 3's graders.
  - The capstone can reuse the mesh pattern.
  - The project depends on Project 3 being finished.

### ADR-002: A2A 1.0, JSON-RPC everywhere, HTTP+JSON on one agent, gRPC optional
- **Context:** A2A 1.0 (2026) defines three equivalent bindings and a required `A2A-Version` header.
- **Decision:** JSON-RPC on every agent (the most widely supported binding). HTTP+JSON also on the Triage
  Agent, so that binding selection from `supportedInterfaces` is exercised. gRPC is a Could.
- **Alternatives:** gRPC everywhere (fast, but less supported by framework clients); A2A 0.3 (legacy).
- **Consequences:** Interop with ADK and the a2a-sdk client. The TCK covers both bindings on Triage.

### ADR-003: The orchestrator is built with Google ADK
- **Context:** Project 3 covered LangGraph, the OpenAI Agents SDK and the Claude Agent SDK. ADK has the
  most native A2A support (`RemoteA2aAgent`, `to_a2a`), and Project 3's ADR-004 left it for this project.
- **Decision:** The Commander uses ADK with `RemoteA2aAgent` peers created from registry cards.
- **Alternatives:** A LangGraph orchestrator (no new framework learned); the raw a2a-sdk client (no
  framework-level orchestration).
- **Consequences:**
  - Four frameworks in the mesh.
  - Gemini or Claude via ADK's model support (recorded in the build plan).
  - ADK's A2A support must work with A2A 1.0. **To verify in week 1**, using the a2a-sdk client as the fallback.

### ADR-004: Remote agents wrapped with the a2a-sdk `AgentExecutor`
- **Context:** LangGraph's built-in A2A endpoint is part of the LangSmith Agent Server (a hosted platform).
  The OpenAI and Claude SDKs have no built-in A2A server.
- **Decision:** One shared executor base (auth, input guard, event mapping, task store) plus a small
  adapter per framework that reuses Project 3's normalized events.
- **Alternatives:** Each framework's own server option where it exists (uneven, some hosted-only).
- **Consequences:**
  - Uniform behaviour, easier comparison.
  - One piece of shared code to test well.

### ADR-005: Cross-agent approval through `auth-required` with out-of-band credentials
- **Context:** A2A 1.0 §7.6 defines in-task authorization: agents move to `AUTH_REQUIRED`, credentials
  arrive out of band, and a chain of `auth-required` tasks lets the request reach the user.
- **Decision:** The Triage/Comms executor sets `auth-required` when Project 3's approval flow starts. The
  Commander mirrors it on its own task. The approval service gives the token **directly** to the acting
  agent.
- **Alternatives:**
  - `input-required` with approval as a message (it would pass the decision, and maybe a credential, through the Commander).
  - An in-band credential extension (credentials travel the chain).
- **Consequences:**
  - A compromised Commander can't capture or reuse approvals.
  - This follows the spec's own security guidance.

### ADR-006: Identity via Keycloak token exchange with `act` claims
- **Decision:**
  - Reuse Project 2's Keycloak, with one client per agent.
  - The Commander exchanges the user token per callee audience (RFC 8693), with `sub` = user and `act` = Commander.
  - Remote agents can't exchange tokens.
  - The delegation depth limit is 2.
- **Alternatives:** Shared API keys between agents (no user identity, no audience); mTLS only (proves the
  agent, not the user it acts for).
- **Consequences:** Per-user authorization and audit at every hop. Keycloak's token-exchange permissions
  must be configured carefully (tested by A5–A7). `act` needs Keycloak's **Token Exchange Delegation**
  feature (still experimental/preview in 26.7), with the Commander's token as `actor_token`. Pin a version
  with the 2026 fix that makes standard exchange reject `act`-carrying tokens
  ([08 §8.3](08-tech-stack.md#keycloak--267-token-exchange-with-delegation)).

### ADR-007: Signed Agent Cards and a registry with pinning
- **Context:** A2A lets cards be signed but doesn't require clients to verify them. Spoofing and card
  changes are the main documented A2A risks.
- **Decision:**
  - Every card is signed (JWS over JCS, ES256, keys per organization).
  - The registry verifies signatures against trusted JWKS, pins approved versions by hash, and quarantines changes.
  - The Commander discovers agents only through the registry.
- **Alternatives:**
  - Direct well-known URL fetch with no verification (the default in many samples).
  - A cloud registry (AgentCore, Azure AI Foundry catalogs), which is less transparent to build on and less to learn from.
- **Consequences:**
  - The same "pin and quarantine" story as Project 2's tools, now for agents.
  - A small service to build and test.

### ADR-008: Remote output is data: schema-validated data parts
- **Decision:**
  - Skills exchange structured data parts validated against versioned JSON Schemas.
  - Free text is spotlighted.
  - The Commander never puts remote text in its instructions, and it flags instruction-like content.
- **Alternatives:** Free-text messages between agents (the common demo style; easy to inject, hard to
  verify).
- **Consequences:**
  - Measurable injection resistance (A11) and a clear contract per skill.
  - More up-front schema work.

### ADR-009: Postgres task stores; push as a hint, `GetTask` as the truth
- **Decision:**
  - The a2a-sdk `DatabaseTaskStore` on Postgres for every agent.
  - Push notifications carry a per-task token, and the receiver re-fetches the task before acting.
  - A 30 s polling fallback.
- **Consequences:** Resumability after restarts, safe push handling, and simple reasoning about
  consistency.

### ADR-010: Measure MCP vs A2A for the same delegation
- **Context:** "When do I use MCP and when A2A?" is the most common interview question on this topic.
  MCP 2026-07-28 added a Tasks extension, which narrows the gap.
- **Decision:** Expose the Triage Agent both as an MCP server (Tasks extension, MRTR for approval) and as
  an A2A agent, run the same scenarios through both, and compare them (§4.6).
- **Consequences:** The write-up asked for in the project list becomes a measured comparison. About a day of
  extra work.

### ADR-011: Own small registry, no standard registry API
- **Context:** A2A defines discovery by well-known URL, registries or configuration, but no standard
  registry API.
- **Decision:** A small FastAPI registry with the features we need to test (verify, pin, quarantine,
  allow-list).
- **Consequences:** When a standard registry API appears, it becomes a migration. That is noted in the report.

### ADR-012: One trace across hops with W3C trace context
- **Decision:** `traceparent` on every A2A request and in push metadata; a2a-sdk telemetry plus Project
  3's instrumentation; Langfuse.
- **Consequences:** A whole incident across five services can be followed in one trace (this is the
  video's closing shot).

### ADR-013: Keep scope to what two to three weeks can prove
- **Decision:**
  - Core: JSON-RPC/REST, streaming, push, cancel, resume, signed + pinned cards, token exchange, the security and resilience suites, the MCP-vs-A2A comparison.
  - Could: gRPC, the extended card polish, Azure deployment, and the OpenAI SDK as an A2A *client*.
  - Out: payments (AP2) and public discovery.
- **Consequences:** The build plan sizes it and may offer a core/lean option, as for Projects 2 and 3.
