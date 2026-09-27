# 6. Non-Functional Design

## 6.1 Latency budget: first notebook cell (p95 ≤ 5 s)

| Step | gVisor | E2B |
|---|---|---|
| Model: first tool call (plan + code) | ≤ 2.5 s | ≤ 2.5 s |
| Sandbox create (cold) | ≤ 1.0 s (pre-pulled image) | ≤ 0.5 s (template snapshot) |
| Attach datasets | ≤ 0.5 s (bind mount) | ≤ 1.0 s (copy of the snapshot; cache in the template for demo data) |
| Execute a simple cell | ≤ 0.5 s | ≤ 0.5 s |
| Filter + stream to UI | ≤ 0.2 s | ≤ 0.2 s |
| **Total** | **≈ 4.7 s** | **≈ 4.7 s** |

**Warm pool (should):** keep 2 pre-created sandboxes per provider for the demo, to hide cold starts. Pool
sandboxes are single-use, never reused across users.

## 6.2 Cost model

| Item | Estimate |
|---|---|
| E2B | Per-second billing (~$0.00003/s for the default 2 vCPU); ~20 s of sandbox time per question → ≈ $0.001; Hobby tier hours cover development |
| gVisor VM (D4s-class, 4 vCPU/16 GB) | ≈ $140/month if always on; **stop when not testing** → ≈ $10–20/month for this project |
| Model per question (cheap tier, ~6 steps) | ≈ $0.01–0.04 |
| AnalystBench full run (50 × 3 runs × 2 providers) | ≈ $5–12 |
| Demo | ≤ $15/month (E2B for the public demo; gVisor VM started on demand) |

## 6.3 Observability

- One trace per question: `invoke_agent` → `execute_tool run_python` → broker span → provider span.
  It uses the OTel GenAI + MCP conventions (as in Projects 2–3), and goes to Langfuse.
- Broker metrics: sessions active, create latency, executions by exit status, kills by limit type,
  dropped outputs, flags (network attempts, PID limit hits).
- A **security dashboard** (Grafana or a Langfuse view) shows flags over time, so a spike in network
  attempts means someone is probing.

## 6.4 Failure modes

| Failure | Effect | Handling |
|---|---|---|
| E2B outage | No sandboxes | Broker fails over to gVisor if the VM is running; otherwise a clear "unavailable" |
| gVisor VM down | No gVisor sandboxes | Fail over to E2B (config) |
| Kernel crash inside the sandbox | Lost variables | Restart kernel; tell the agent and user |
| Stuck execution | Blocks the session | Wall-clock kill; session remains usable |
| Reaper failure | Leaked sandboxes, cost | Hard lifetime set at creation (provider-side timeout); daily reconciliation job |

## 6.5 Scaling path

| Now | Later |
|---|---|
| One gVisor VM | VM scale set; or Kubernetes with a `runsc` RuntimeClass (GKE Sandbox / AKS equivalents) |
| Warm pool of 2 | Pool sized per hour of day |
| Datasets copied per session | Shared read-only volume snapshots; lazy loading |
| One broker replica | Stateless replicas; session map in Redis |

## 6.6 Testing strategy

| Layer | Tests |
|---|---|
| Broker | Unit: output filter, Vega validator, quota maths; integration: both providers with a trivial image |
| Harness | Limits enforced (CPU, memory, file size, processes) |
| Security | Escape suite, injection cases, UI safety tests (§4.4–§4.7) |
| Agent | Stub-model tests of the repair loop; AnalystBench smoke |
| Image | Trivy scan; reproducible build; package hash pinning |
