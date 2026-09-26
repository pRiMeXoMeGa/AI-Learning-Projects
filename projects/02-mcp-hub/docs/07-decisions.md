# 7. Architecture Decision Records

Format: **Context → Decision → Alternatives → Consequences.** Status is *Proposed* until validated while
building.

---

### ADR-001: Indian mutual-fund data (AMFI) for the open-source server
- **Context:** The server must be useful to real people (so it gets installs and stars), use free public
  data, and have both read and per-user write use cases.
- **Decision:** AMFI daily NAVs and NAV history for all Indian mutual-fund schemes, stored in our own
  Postgres, plus user watchlists and holdings.
- **Alternatives:** data.gov.in (needs keys, uneven quality); UAE open data (fewer daily-use questions);
  stock prices (licensing issues).
- **Consequences:** Real calculations (CAGR, XIRR, SIP back-tests) give the evals exact, checkable
  answers. **To verify before publishing:** AMFI's terms of use for redistributing the data; attribute the
  source in every response.

### ADR-002: Target MCP 2026-07-28 (stateless) with version negotiation
- **Context:** The 2026-07-28 revision removed the `initialize` handshake and sessions, replaced
  server-initiated requests with multi round-trip requests, and added routing headers.
- **Decision:** Build everything on the new protocol (Python SDK v2 / FastMCP 4, TypeScript SDK v2), and
  support older clients through per-connection version negotiation.
- **Alternatives:** Build on the older session-based protocol (more tutorials exist, but it's already
  legacy and doesn't scale behind plain load balancers).
- **Consequences:** Current, differentiated knowledge; some SDK features may be new and less documented,
  so spikes are planned early in the build.

### ADR-003: Streamable HTTP for remote, stdio for local; no legacy SSE transport
- **Decision:** Streamable HTTP for every remote endpoint; stdio for local use of the open-source servers.
- **Alternatives:** The legacy HTTP+SSE transport (deprecated).
- **Consequences:** Works with modern clients; old SSE-only clients are not supported.

### ADR-004: Use an existing authorization server (Keycloak); the hub is only a resource server
- **Context:** Writing an OAuth server is risky and not the point of the project.
- **Decision:** Keycloak issues all tokens and performs token exchange. The gateway and servers validate
  tokens as **resource servers**.
- **Alternatives:** Auth0 / Entra ID / WorkOS (hosted; good for production, less transparent locally);
  a home-made authorization server (rejected: security risk).
- **Consequences:** Runs fully locally in Docker. **To verify when building:** Keycloak's support for
  Client ID Metadata Documents and resource indicators in the version used; fall back to pre-registered
  clients or Dynamic Client Registration where needed, and document it.

### ADR-005: Client ID Metadata Documents (CIMD) for client registration
- **Decision:** The own client identifies itself with a CIMD URL. DCR is supported only as a fallback,
  because it's deprecated in 2026-07-28.
- **Consequences:** No registration endpoint to secure; the client's metadata (redirect URIs, name) is
  publicly verifiable.

### ADR-006: No token passthrough: token exchange per upstream audience
- **Context:** Forwarding the client's token to upstream servers creates the confused-deputy problem and
  breaks audit trails; the spec forbids it.
- **Decision:** The gateway exchanges the user's token (RFC 8693) for a token whose audience is the
  specific upstream, keeping the user as `sub` and the gateway as `act`. Per-user tokens for third-party
  servers (GitHub) are obtained through URL-mode consent and stored encrypted.
- **Alternatives:** Passthrough (forbidden); a shared service token for all upstreams (loses user
  identity, enables IDOR).
- **Consequences:** Upstreams can authorize per user; one extra token call per user per upstream per token
  lifetime (cached).

### ADR-007: Build the gateway ourselves on the MCP Python SDK v2
- **Context:** Open-source gateways exist (e.g. agentgateway, IBM ContextForge, Docker MCP Gateway).
- **Decision:** Write the gateway's control logic (registry, pinning, policy, confirmations, audit) on the
  MCP Python SDK v2, and run a **comparison** against one existing gateway (agentgateway) in the
  performance and security reports.
- **Alternatives:** Configure an existing gateway (faster, but shows configuration skills rather than
  protocol and security design); FastMCP's proxy/mount features (good for aggregation, but the security
  pipeline would still be custom).
- **Consequences:** More code, but every control is understood and explainable. The comparison shows you
  know the landscape and can judge build vs. buy.

### ADR-008: Server-prefixed tool names (`server__tool`)
- **Decision:** Exposed names are `<prefix>__<tool>` (e.g. `mf__compare_schemes`).
- **Why:** OpenAI's tool names only allow `[a-zA-Z0-9_-]`, which rules out dots; the prefix also prevents
  name collisions and shadowing between servers.

### ADR-009: Pin and approve tool definitions
- **Decision:** Hash the canonical JSON of every tool definition (name, title, description, schemas,
  annotations). Only approved hashes are exposed; changed definitions are quarantined until an admin
  approves the diff.
- **Alternatives:** Trust upstream `tools/list` (vulnerable to rug pulls); allow-list by name only (misses
  description changes).
- **Consequences:** Upstream releases need a review step, which is intended.

### ADR-010: OPA (Rego) for policy decisions
- **Decision:** The gateway is the enforcement point; OPA, as a sidecar with a versioned bundle, is the
  decision point.
- **Alternatives:** Cedar (analysable, in-process, less common in job descriptions); rules hard-coded in
  Python (fast to write, hard to audit and test separately).
- **Consequences:** Policies are testable on their own (`opa test`), versioned, and written into every
  audit event. A few milliseconds per decision, reduced with caching.

### ADR-011: Stateless confirmations with a signed `requestState`
- **Decision:** Pending confirmations live in the client's retry, as an HMAC-signed state bound to user,
  client, tool, argument hash and expiry; only single-use nonces go to Redis.
- **Alternatives:** Server-side confirmation records (needs sticky sessions or shared storage for every
  step, against the stateless design).
- **Consequences:** Any replica can finish any confirmation; key rotation is handled with `kid`.

### ADR-012: Hash-chained, append-only audit log in Postgres
- **Decision:** Every event stores the previous event's hash; the app role can only insert; a daily anchor
  is exported.
- **Alternatives:** Plain log table (tampering undetectable); an external immutable ledger (overkill here).
- **Consequences:** Tamper evidence at low cost; a verification endpoint for reviewers.

### ADR-013: Don't use deprecated protocol features
- **Decision:** No sampling, roots or logging from servers; no legacy SSE transport.
- **Consequences:** Server-side reasoning stays out of the servers (they return data; the client's model
  reasons). Diagnostics use OpenTelemetry, not MCP logging.

### ADR-014: Own MCP client in a raw tool-calling loop
- **Decision:** A small client written directly against the Claude and OpenAI APIs and the MCP SDK, with no
  agent framework.
- **Alternatives:** LangGraph or a vendor agent SDK (hides the protocol details this project is meant to
  show; those are compared in Project 3).
- **Consequences:** Full control over OAuth, `input_required` handling and trajectory recording for evals.

### ADR-015: Frozen data snapshot for evals
- **Decision:** Evals run against a fixed database snapshot; expected answers are computed by code from it.
- **Consequences:** Reproducible results; the snapshot is refreshed deliberately, with a new eval baseline.

### ADR-016: Python for the gateway and main server, TypeScript for the second server and the console
- **Decision:** Python where your depth is (gateway, india-mf-mcp); TypeScript for fx-rates-mcp and the
  React console.
- **Consequences:** Shows both official SDKs (a common JD requirement) without splitting the core logic
  across languages.

### ADR-017: Azure Container Apps for deployment (as in Project 1)
- **Decision:** Same platform and Terraform patterns as Project 1, without session affinity for the gateway.
- **Alternatives:** AWS ECS / Lambda (the stateless protocol would also fit Lambda well; documented as an
  alternative).
- **Consequences:** Reuses Project 1's infrastructure code.
