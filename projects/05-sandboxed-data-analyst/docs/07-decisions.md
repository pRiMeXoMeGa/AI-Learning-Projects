# 7. Architecture Decision Records

Format: **Context → Decision → Alternatives → Consequences.** Status is *Proposed* until validated while
building.

---

### ADR-001: A data-analyst agent on public datasets
- **Context:** Code execution is the capability, and sandboxing is the lesson. Data analysis is the most
  common real use of agent code execution and has checkable answers.
- **Decision:** An analyst agent over **UCI Online Retail II** (CC BY 4.0) and an **NYC TLC taxi** sample,
  plus user uploads.
- **Alternatives:**
  - M5 Walmart data (the capstone's domain, but Kaggle competition terms limit redistribution).
  - A coding agent (the sandbox story is similar, but answers are harder to grade).
- **Consequences:**
  - Gold answers come from reference SQL.
  - Online Retail II has realistic traps (returns, cancellations, missing ids).

### ADR-002: A sandbox broker service, exposed over MCP
- **Decision:** One Python service owns sandbox creation, policy, output filtering and audit. It is
  exposed as an **MCP server** (FastMCP 4, 2026-07-28 protocol), and the web agent is one of its clients.
- **Alternatives:** Call E2B's SDK directly from the Next.js agent (simpler, but every caller would need
  to re-implement policy, and nothing is reusable by the capstone).
- **Consequences:**
  - One enforcement point, usable by the capstone's LangGraph agents and fronted by Project 2's gateway if wanted.
  - An extra network hop (~10–30 ms).

### ADR-003: Two providers, E2B (Firecracker) and self-hosted gVisor, behind one interface
- **Context:** Isolation claims should be compared, not assumed. E2B is the best-known managed option
  (microVMs). gVisor is the standard self-hostable strong-isolation runtime.
- **Decision:** Implement both with the same image, limits and escape suite. **E2B is the default for the
  demo; gVisor is the self-hosted option.**
- **Alternatives:**
  - Daytona (fastest cold starts; container isolation by default).
  - Modal (GPUs; containers).
  - Vercel Sandbox (close to the P6 stack).
  - Plain Docker (runc; not an acceptable boundary for untrusted code).
  - Kata Containers (microVMs on Kubernetes; heavier setup).
- **Consequences:**
  - A provider comparison with evidence.
  - Two integrations to maintain.
  - A third provider is a "Could" behind the same interface.

### ADR-004: The gVisor host is dedicated and isolated
- **Decision:** A separate VM runs only Docker + `runsc` and a tiny runner service. It is reachable only
  from the broker over a private network. Nothing else runs there: no database, no secrets, no broker.
- **Consequences:** A theoretical escape lands on an empty machine. The VM is stopped when not in use, to
  save cost.

### ADR-005: No network inside sandboxes (for this project)
- **Decision:** No egress, no DNS, no metadata access. Everything the code needs (the datasets and the
  image's packages) is present at start.
- **Alternatives:** An egress proxy with an allow-list (needed for agents that browse or install packages;
  noted as future work).
- **Consequences:** It removes the whole exfiltration and C2 class of attacks. Package needs are handled by
  rebuilding the image.

### ADR-006: Everything leaving the sandbox is data, validated outside
- **Context:** The main 2026 escape pattern was a trusted outside component running or rendering
  something the sandbox produced.
- **Decision:**
  - Output types are allow-listed by magic bytes, and PNGs are re-encoded.
  - Pickle, HTML and SVG are never accepted.
  - Charts are **Vega-Lite specs** validated against a restricted subset and rendered with the CSP-safe
    interpreter.
  - stdout is shown as escaped text.
- **Alternatives:** Let the sandbox render charts as HTML/SVG (flexible, dangerous); matplotlib PNGs
  only (safe but not interactive).
- **Consequences:** Interactive, safe charts. Some chart types are unavailable; PNG is the fallback.

### ADR-007: AI SDK 7 agent in the web app, reusing Project 6's shell
- **Decision:**
  - The agent runs in the Next.js app with AI SDK 7, using the broker's MCP tools and a typed final `Output.object`.
  - The UI reuses Project 6's shell packages: auth, orgs, AI Elements, layout.
- **Alternatives:** A Python agent (LangGraph) behind an API (it would duplicate the streaming UI work);
  the Claude Agent SDK (strong for coding agents, but its built-in tools would need disabling, as in
  Project 3).
- **Consequences:**
  - Fast UI, with shared components and patterns.
  - The capstone's Python agents still reach the sandbox through MCP.

### ADR-008: Stateful kernel per session, with an execution budget
- **Decision:** A persistent IPython kernel per conversation. It is capped at 6 executions per question and
  60 minutes of lifetime.
- **Alternatives:** Stateless runs (each cell re-loads data; simpler and more reproducible, but slower
  and more tokens).
- **Consequences:** Faster iterative analysis. Experiment X2 measures the trade-off, and the notebook
  export records cell order for reproducibility.

### ADR-009: AnalystBench-50 plus one DABstep submission
- **Decision:**
  - Our own benchmark, with gold answers from reference SQL, graded on numbers, lists and chart data.
  - One DABstep leaderboard submission as an external reference.
- **Consequences:**
  - An internal number we control, and an external number others can compare with.
  - The DABstep submission is made once with a frozen agent, to avoid overfitting.

### ADR-010: Fixed sandbox image, built and scanned in CI
- **Decision:**
  - One Dockerfile with pinned, hashed packages, built in CI and scanned (Trivy).
  - The same image is used for the E2B template and gVisor.
  - No `pip install` at run time.
- **Consequences:** Reproducible behaviour and a smaller supply-chain surface. New packages go through a
  PR.
