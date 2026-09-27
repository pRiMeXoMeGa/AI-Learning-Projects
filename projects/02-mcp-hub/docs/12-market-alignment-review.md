# 12. Market Alignment Review (September 2026)

> **Question answered:** Does every approach in Project 2 match how MCP servers, gateways and agent
> security are built and hired for in 2026, and what's missing?
>
> **Method:** ~20 web searches across the MCP spec and blog, vendor docs (AWS, Keycloak, Okta, Anthropic),
> security research and incident reports, 2026 benchmarks, gateway market comparisons and job-market
> guides (sources at the end). Each area was rated; recommended changes were **applied to the design docs**,
> and optional ones are listed as a backlog.

## 12.1 Verdict

**The design is well aligned and, on protocol currency, ahead of most of the market.** The evidence
supports its core bets:

1. **Build on MCP 2026-07-28.** AWS AgentCore Gateway already supports it, FastMCP 4 and the SDKs shipped
   with it, and statelessness is what lets gateways scale behind plain load balancers.
2. **A gateway is where MCP security lives.** 2026 comparisons describe the same core feature set this
   design has (OAuth 2.1 / OIDC, per-user identity, tool-level allow-lists, audit trails, aggregation,
   observability), because MCP itself has none of them.
3. **Security is a real, paid-for specialty.** There were real incidents (a backdoored `postmark-mcp` npm
   package; a CVSS 9.6 RCE in `mcp-remote`; GitHub MCP prompt injection; an RCE chain in `mcp-server-git`),
   50+ tracked MCP vulnerabilities, and dedicated "MCP security engineer" roles with high salaries.
   Tool poisoning, rug pulls, shadowing and confused deputies (all in the threat model) are the named
   attack classes.
4. **Measuring tool design.** Anthropic's own guidance says tool descriptions and naming schemes have
   large, model-dependent effects and should be chosen **by evaluation**. That is exactly what F17 does.

**But nine areas needed updating:**

| # | Gap | Why it matters in 2026 | Change applied |
|---|---|---|---|
| 1 | **Policy engine: OPA** | AWS built **AgentCore Policy** (GA March 2026), the best-known managed MCP gateway policy layer, on **Cedar**, not OPA. Cedar is purpose-built for authorization, runs in-process, and its analysis tools can *prove* properties such as "no destructive tool without confirmation" | **Switch to Cedar** (in-process via `cedarpy`); OPA stays the documented alternative |
| 2 | **Keycloak support was unverified** | Keycloak now publishes an official "Keycloak as MCP authorization server" guide: **CIMD** experimental (`--features=cimd`), **resource indicators** since 26.7, CIMD + resource indicators merged, and both expected to reach *preview* in 26.8 | Pin **Keycloak ≥ 26.7** (26.8 when released); the F4 spike becomes a confirmation, not an open question |
| 3 | **Tool-list bloat not addressed** | The 2026 fixes for context bloat are **tool search / deferred loading** (Anthropic Tool Search Tool: ~85% fewer tokens) and code execution with MCP ("code mode"). Gateways now offer tool search | Gateway **search mode** (`hub__search_tools` + `hub__call_tool`) and toolset variant **T6** in the evals |
| 4 | **india-mf-mcp has competition** | At least 5 open-source Indian mutual-fund MCP servers already exist (AMFI/mfapi based) | Explicit **differentiation** in requirements and README: current spec, hosted OAuth, structured outputs, interactive tools, tested XIRR/SIP maths, per-user portfolio, FX/NRI view, published evals |
| 5 | **MCP Apps undervalued** | 11 clients now render MCP Apps (Claude, ChatGPT, VS Code, Cursor, M365 Copilot…), with launch partners like Figma, Slack and Salesforce | The **NAV chart as an MCP App** moves into the core plan (a small, visible full-stack piece while the React console stays deferred) |
| 6 | **Injection defence = regex heuristics only** | 2026 research: static attack success is near zero but **adaptive attacks succeed > 90%** against most published defences; classifier-based detectors are the common production layer; "lethal trifecta" is the standard framing | Add an **open prompt-injection classifier** option, **adaptive attacks** in the security evals, and lethal-trifecta analysis in the threat model |
| 7 | **No supply-chain scanning** | Scanners such as **Snyk Agent Scan** (formerly Invariant's mcp-scan) are the standard check for poisoned tool descriptions | Scan upstream tool definitions **at registration/approval** and scan our own servers in CI; stdio servers run in network-less containers |
| 8 | **Frameworks outside OWASP LLM Top 10 not mapped** | **OWASP Top 10 for Agentic Applications (ASI01–ASI10, Dec 2025)** and the **OWASP MCP Top 10** (beta) are now the reference lists | Threat model mapped to both, plus a table of real 2025–26 incidents and which control addresses each |
| 9 | **Observability not on MCP conventions** | OpenTelemetry **MCP semantic conventions** exist since January 2026 (`mcp.method.name`, `gen_ai.tool.name`, span name `tools/call {tool}`), still "Development" | Adopt them and pin the version |

**Effort impact:** about **+11 h** (details in §12.4). The core plan goes from ~103 h to **~114 h** and
**stays at 8 weeks** at ~14 h/week (inside the 12–15 h/week range). The roadmap doesn't change.

## 12.2 Scorecard

✅ aligned · ⚠️ updated · ➕ added · 🔸 optional backlog · ⛔ considered and rejected (with reason)

| Area | Status | Summary |
|---|---|---|
| Protocol: MCP 2026-07-28, stateless, MRTR, routing headers | ✅ | Current; AWS AgentCore and FastMCP 4 already support it |
| Transports: Streamable HTTP + stdio; no legacy SSE | ✅ | Matches the spec's deprecations |
| Open-source server domain (Indian MFs) | ✅ ⚠️ | Useful, but crowded; differentiation added |
| Structured outputs, annotations, tool style guide | ✅ | Matches Anthropic's "writing tools for agents" guidance |
| Interactive tools (MRTR forms, URL mode) | ✅ | Current spec feature |
| MCP Apps | ⚠️ | NAV chart moved into core |
| Tasks extension | ✅ 🔸 | Still deferred; low demand in the target JDs |
| OAuth: resource server, PRM, RFC 8707, `iss` check, CIMD, step-up | ✅ ⚠️ | Keycloak support now confirmed (version pinned) |
| No token passthrough; token exchange | ✅ | Required by the spec; standard gateway practice |
| Enterprise-Managed Authorization (ID-JAG / Okta Cross App Access) | 🔸 | Stable since June 2026 and supported by Claude, VS Code and agentgateway; backlog (depends on IdP support) |
| Gateway: aggregation, namespacing, per-tenant filtering | ✅ | Standard 2026 feature set |
| Gateway: definition pinning / rug-pull quarantine | ✅ | Ahead of many gateways (the spec has no change tracking) |
| Gateway: tool search / progressive disclosure | ➕ | Search mode + T6 variant |
| Policy engine | ⚠️ | OPA → Cedar |
| Confirmations with signed state | ✅ | Needed for statelessness; uncommon and a good interview story |
| Rate limits, audit chain | ✅ | Standard (audit) and above standard (hash chain) |
| Injection filters | ⚠️ | Classifier option + adaptive attacks |
| Supply-chain scanning, stdio sandboxing | ➕ | Scanner at approval + CI; containers without network |
| Threat model framing | ⚠️ | OWASP Agentic Top 10, OWASP MCP Top 10, lethal trifecta, real incidents |
| Own client (raw loop, two model families) | ✅ | Shows protocol mastery; frameworks are compared in Project 3 |
| Tool-design evals (variants, pass^k, per-type) | ✅ ➕ | Consistent with MCPMark (pass^4) and LiveMCPBench style; T6 added |
| Public MCP benchmarks (MCPMark, MCP-Universe, MCP-Bench, LiveMCPBench) | 🔸 | Different domains; cite them, optionally run one small subset |
| Observability | ⚠️ | OTel MCP semantic conventions |
| Build-your-own gateway vs. buy | ✅ | Comparison now includes AWS AgentCore Gateway (managed) as well as agentgateway |
| Code execution with MCP ("code mode") | ⛔ | See §12.3 |

## 12.3 Considered and rejected (and how to defend it in an interview)

| Technique | Why it's not in this project | Evidence |
|---|---|---|
| **Code execution with MCP / Code Mode** | Biggest token savings (up to ~98%) at hundreds of tools; needs a secure code sandbox, which is **Project 5's** topic. With ~30 tools here, tool search gives most of the benefit | Anthropic and Cloudflare write-ups; independent reproductions |
| **CaMeL-style dual-LLM architecture** | Strong guarantees in theory, but no production implementation and it would redesign the client. The design breaks the **lethal trifecta** instead (confirmations + cross-server flow rules) | 2026 prompt-injection surveys |
| **Building an authorization server** | Unchanged: security risk; Keycloak now supports what MCP needs | Keycloak MCP guide |
| **FastMCP's OAuth proxy as the gateway's auth** | Useful for servers whose IdP lacks DCR/CIMD, but the gateway must be a pure resource server with token exchange | FastMCP docs; ADR-004/006 |
| **OPA (kept as alternative)** | Still widely used in platform teams; Cedar fits agent/tool authorization better and matches AWS's MCP gateway | AWS security blog on choosing Cedar |

## 12.4 Changes applied to the design

| # | Change | Where | Effort |
|---|---|---|---|
| B1 | **Cedar** replaces OPA: in-process evaluation (`cedarpy`), schema, policy tests, **Cedar analysis** check that no destructive tool is permitted without confirmation | F10 (rewritten), [03 §3.5](03-low-level-design.md), ADR-010, [08](08-tech-stack.md), [06](06-non-functional.md) | 0 h (same effort, no sidecar) |
| B2 | Keycloak **≥ 26.7** pinned; CIMD via `--features=cimd`; resource indicators; official MCP guide followed | F4, ADR-004, [08](08-tech-stack.md) | 0 h |
| B3 | Gateway **search mode**: `hub__search_tools(query)` returns matching tool definitions, `hub__call_tool(name, args)` calls them through the same pipeline; toolset variant **T6** | F8, F17, [04](04-evaluation-design.md) | +3 h |
| B4 | **Differentiation** of india-mf-mcp (requirements + README comparison table) | [01](01-requirements.md), F3, F22 | +0.5 h |
| B5 | **MCP Apps NAV chart** in the core plan | F15 (split), [01](01-requirements.md) | +2 h |
| B6 | **Prompt-injection classifier** option (e.g. an open model such as Llama Prompt Guard 2) compared with heuristics and "off" | F12, F18 | +1.5 h |
| B7 | **Adaptive attacks** in the end-to-end security suite (an attacker model rewrites a failed injection up to N times) | F18, [04 §4.6](04-evaluation-design.md) | +1.5 h |
| B8 | **Scanner at approval** (Snyk Agent Scan or equivalent) + CI scan of our own servers; **stdio servers in containers without network** | F8, F9, F20 | +1.5 h |
| B9 | Threat model mapped to **OWASP Agentic Top 10** and **OWASP MCP Top 10**; **lethal trifecta** analysis; **real incidents** table | [05](05-security-threat-model.md) | +0.5 h |
| B10 | **OTel MCP semantic conventions** | [06 §6.3](06-non-functional.md), F0 | +0.5 h |
| B11 | Build-vs-buy comparison adds **AWS AgentCore Gateway** | ADR-007, F19 | 0 h |
| | **Total added** | | **≈ +11 h** |

## 12.5 Optional backlog

| ID | Item | Why it's interesting | Effort |
|---|---|---|---|
| P1 | **Enterprise-Managed Authorization (ID-JAG)** | Stable since June 2026; Okta ships it as Cross App Access; zero-consent enterprise rollout. Needs IdP support (check Keycloak) | ~3 h |
| P2 | Run a **small subset of a public MCP benchmark** (e.g. MCPMark's PostgreSQL tasks) through the gateway | Shows the gateway doesn't hurt agent performance on a known benchmark | ~3 h |
| P3 | **Cedar vs OPA micro-benchmark** (latency, policy size, analysis) | A crisp build-vs-buy / tool-choice story | ~1.5 h |
| P4 | **Prefix vs. suffix tool naming** variant (T7) | Anthropic reports model-dependent effects | ~1 h |
| P5 | Deferred features from the core plan: admin console, Tasks/resources/prompts, GitHub upstream | As before | ~14 h |

## 12.6 Market keywords this project now covers

MCP 2026-07-28 (stateless, MRTR, routing headers), MCP servers (Python + TypeScript SDKs), MCP gateway,
OAuth 2.1 / OIDC for MCP (CIMD, PKCE, RFC 8707/9728/9207, step-up, token exchange), Keycloak, **Cedar**
policy-as-code, tool poisoning / rug-pull / confused-deputy defences, **OWASP Agentic Top 10**,
**OWASP MCP Top 10**, prompt-injection detection, **tool search / progressive disclosure**, **MCP Apps**,
MCP Registry, supply-chain scanning, OpenTelemetry (MCP conventions), tool-design evaluation.

## 12.7 Sources

**Protocol and platforms**
- [The 2026-07-28 MCP specification (MCP blog)](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- [How AgentCore Gateway supports MCP 2026-07-28 (AWS)](https://aws.amazon.com/blogs/machine-learning/how-agentcore-gateway-supports-the-mcp-2026-07-28-spec/)
- [MCP Apps announcement (MCP blog)](https://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/) · [MCP Apps in 2026: what shipped, host by host](https://www.devmoment.dev/journal/mcp-apps-field-log-2026)
- [Everything your team needs to know about MCP in 2026 (WorkOS)](https://workos.com/blog/everything-your-team-needs-to-know-about-mcp-in-2026)

**Authorization**
- [Keycloak as an MCP authorization server (Keycloak docs)](https://www.keycloak.org/securing-apps/mcp-authz-server) · [Keycloak 26.6.0 release](https://www.keycloak.org/2026/04/keycloak-2660-released) · [CIMD + resource indicators issue](https://github.com/keycloak/keycloak/issues/51413) · [Keycloak CIMD for MCP (Skycloak)](https://skycloak.io/blog/keycloak-cimd-mcp-authorization/)
- [Enterprise-Managed Authorization (MCP blog)](https://blog.modelcontextprotocol.io/posts/enterprise-managed-auth/) · [Inside the ID-JAG (WorkOS)](https://workos.com/blog/mcp-enterprise-managed-authorization-id-jag) · [agentgateway supports ID-JAG (AAIF)](https://aaif.io/blog/agentgateway-now-supports-id-jag-and-mcp-enterprise-managed-auth)
- [FastMCP OAuth proxy](https://gofastmcp.com/servers/auth/oauth-proxy)

**Policy**
- [Why AgentCore Policy chose Cedar (AWS Security Blog)](https://aws.amazon.com/blogs/security/why-policy-in-amazon-bedrock-agentcore-chose-cedar-for-securing-agentic-workflows/) · [AgentCore Policy core concepts](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-core-concepts.html) · [MCP access control: OPA vs Cedar (Natoma)](https://natoma.ai/blog/mcp-access-control-opa-vs-cedar-the-definitive-guide) · [OPA vs Cedar (Styra)](https://www.styra.com/knowledge-center/opa-vs-cedar-agent-and-opal/)

**Gateways and tool loading**
- [MCP gateway comparison 2026 (Requesty)](https://www.requesty.ai/blog/mcp-gateway-comparison-2026-enterprise-scalability-security) · [13 best MCP gateways (Obot)](https://obot.ai/blog/the-13-best-mcp-gateways-for-enterprise-teams/) · [Enterprise MCP gateways compared (Tyk)](https://tyk.io/learning-center/best-enterprise-mcp-gateways/)
- [MCP context bloat fix: tool search, code mode (MCP.Directory)](https://mcp.directory/blog/mcp-context-bloat-fix-2026-tool-search-code-mode-progressive-disclosure) · [Code execution with MCP (AIMultiple)](https://aimultiple.com/code-execution-with-mcp)
- [Writing effective tools for agents (Anthropic)](https://www.anthropic.com/engineering/writing-tools-for-agents)

**Security**
- [State of MCP security 2026 (PipeLab)](https://pipelab.org/blog/state-of-mcp-security-2026/) · [MCP security statistics 2026 (Practical DevSecOps)](https://www.practical-devsecops.com/mcp-security-statistics-2026-report/) · [CSA research note: MCP security crisis](https://labs.cloudsecurityalliance.org/research/csa-research-note-mcp-security-crisis-20260504-csa-styled/)
- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) · [OWASP LLM Top 10: what comes next, incl. MCP Top 10 (Imperva)](https://www.imperva.com/blog/owasp-llm-top-10-what-comes-next-agentic-mcp/)
- [Inside the lethal trifecta (Sophos)](https://www.sophos.com/en-us/blog/inside-the-lethal-trifecta-blast-radius-reduction-in-ai-agent-deployments) · [Prompt injection 2026 guide (Sysdig)](https://www.sysdig.com/learn-cloud-native/prompt-injection)
- [Snyk Agent Scan (GitHub)](https://github.com/snyk/agent-scan) · [MCP-Scan review 2026](https://appsecsanta.com/mcp-scan)

**Observability**
- [OpenTelemetry MCP attributes](https://opentelemetry.io/docs/specs/semconv/registry/attributes/mcp/) · [GenAI + MCP semantic conventions (GitHub)](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/mcp.md) · [MCP Python SDK OpenTelemetry](https://py.sdk.modelcontextprotocol.io/run/opentelemetry/)

**Benchmarks and market**
- [MCPMark](https://mcpmark.ai/) · [MCP-Universe](https://mcp-universe.github.io/) · [LiveMCPBench](https://icip-cas.github.io/LiveMCPBench/) · [MCP-Bench (Accenture)](https://github.com/Accenture/mcp-bench)
- [Existing Indian MF MCP servers: SetuAI AMFI MCP](https://glama.ai/mcp/servers/SetuAI/amfi-mcp-server) · [mutual-fund-mcp](https://github.com/bikramjitchawla/mutual-fund-mcp) · [mcp-mfapi-india](https://github.com/pipeworx-io/mcp-mfapi-india) · [indian-mf-mcp](https://github.com/LogeshR15/indian-mf-mcp)
- [MCP security jobs and salaries 2026 (Practical DevSecOps)](https://www.practical-devsecops.com/mcp-security-jobs-salaries-2026/)
