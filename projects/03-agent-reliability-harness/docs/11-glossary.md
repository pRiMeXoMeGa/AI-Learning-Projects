# 11. Glossary

Plain-English definitions of the terms used in these docs, and where each one shows up in this project.

## Agents and frameworks

| Term | Meaning | In this project |
|---|---|---|
| **Agent** | A model in a loop that decides which tools to call until the task is done | The Triage Agent |
| **Raw loop** | An agent written directly against the model API, with no framework | The baseline implementation |
| **LangGraph** | A framework that models an agent as a graph of nodes and edges with saved state | Main implementation; also the multi-agent variant |
| **OpenAI Agents SDK** | OpenAI's lightweight agent framework: agents, tools, handoffs, guardrails, sessions, tracing | Third implementation |
| **Claude Agent SDK** | Anthropic's library version of the Claude Code agent loop, with hooks and permission callbacks | Fourth implementation (built-in tools disabled) |
| **Microsoft Agent Framework** | Microsoft's agent SDK (1.0 in April 2026) for .NET and Python, with workflows, HITL and checkpoints | Optional fifth implementation |
| **Agent spec** | The shared definition every implementation uses: prompt, tools, risk policy, budgets, output schema | `agents/spec/triage_agent.yaml` |
| **Adapter** | Thin code that runs one framework behind a common `run()` interface and emits common events | One per implementation |
| **Supervisor / specialists** | A multi-agent design where one agent routes work to focused sub-agents | LangGraph variant in E7 |
| **Handoff** | Passing control from one agent to another (OpenAI Agents SDK term) | Mentioned in the DX comparison |

## Reliability and evaluation

| Term | Meaning | In this project |
|---|---|---|
| **Scenario** | One planted incident with a ticket, faults, attacks, approver rules and expected outcome | OpsDesk-50 has 50 |
| **Run / episode** | One attempt by one implementation at one scenario | ~800 in E1 |
| **pass@1** | Share of runs that succeed | Headline metric (with pass^k) |
| **pass^k** | Chance that **all** k runs of a scenario succeed; measures consistency | k = 1…4 curves |
| **Consistency / robustness / predictability / safety** | The four reliability dimensions from 2026 agent-reliability research | Report structure (§4.2) |
| **Calibration** | Whether the agent's stated confidence matches how often it's right | Brier score, ECE |
| **ECE** (expected calibration error) | Average gap between confidence and accuracy across confidence bins | Predictability metric |
| **Trajectory** | The sequence of steps and tool calls in a run | Trajectory metrics from Project 1 |
| **Key-call recall** | Share of the essential tool calls the agent made | Trajectory grader |
| **Oracle policy** | A scripted perfect agent, used to prove a scenario is solvable and graders are correct | Scenario QA |
| **Bootstrap CI** | A confidence interval from resampling scenarios many times | All reported numbers |
| **Paired comparison** | Comparing two implementations on the same scenarios and seeds | E1, E2 |
| **Holm correction** | An adjustment so that many comparisons don't produce false "wins" | E1 pairwise tests |
| **Dev / test split** | Scenarios used for tuning vs. scenarios held back for the final numbers | 35 / 15 |
| **Failure taxonomy** | A fixed list of failure modes each failed run is labelled with | F1–F12 (§4.8), adapted from MAST |
| **MAST** | Multi-Agent System failure Taxonomy (2025), 14 failure modes from 1,600+ traces | Basis for F1–F12 |
| **LLM judge** | A model grading text against a rubric | Only the summary quality |

## Human-in-the-loop and durability

| Term | Meaning | In this project |
|---|---|---|
| **HITL** (human in the loop) | The agent pauses for a person to approve, deny or edit an action | Approvals for high/critical tools |
| **Interrupt** | LangGraph's way to pause a graph and wait for input | Approval node |
| **RunState** | OpenAI Agents SDK's serializable state of a paused run | Approval + resume |
| **Permission callback / hook** | Claude Agent SDK functions that allow, deny or change a tool call | Approval + guards |
| **Approval token** | A signed, single-use proof that a specific call with specific arguments was approved | Checked by the simulator |
| **Backstop** | A second, independent check that catches what the first one missed | Simulator rejects unapproved risky calls |
| **Checkpointer** | Saves an agent's state after each step so it can resume | LangGraph Postgres checkpointer |
| **Durable execution** | Running code so that it survives crashes and resumes exactly where it stopped | E5; Temporal variant (optional) |
| **Idempotency key** | An ID that makes repeating the same action safe (the second time does nothing) | Write tools |
| **Chaos mode** | Killing the agent on purpose to test recovery | E5 |
| **Time travel** | Replaying a run from an earlier checkpoint, optionally with edited state | LangGraph debugging |

## Safety

| Term | Meaning | In this project |
|---|---|---|
| **Indirect prompt injection** | Instructions hidden in data the agent reads (logs, tickets) | S4 scenarios, E6 |
| **Spotlighting** | Marking untrusted text clearly as data so the model is less likely to obey it | Spec flag, E6 |
| **ASR** (attack success rate) | Share of attack runs where the attacker's goal happened | E6, E8 |
| **Canary secret** | A fake secret whose appearance in an output proves a leak | Config values, outbox scan |
| **Lethal trifecta** | Untrusted input + private data + outward communication in one agent | §5.4 |
| **Excessive agency** | An agent doing more than it needs to, or more than it's allowed | S6 scenarios, S3 violations |
| **Memory poisoning** | Planting false instructions in an agent's long-term memory | S7, write policy |
| **Violation severity** | S1 critical harm, S2 policy breach, S3 unnecessary risky action | Safety grader |
| **Guard / budget** | Hard limits on steps, tool calls, tokens, cost and time | Guard lib, E3 |
| **Loop detection** | Stopping an agent that repeats itself without progress | Guard lib |

## Incident domain

| Term | Meaning | In this project |
|---|---|---|
| **Incident / SEV** | An unplanned disruption, ranked by severity (SEV1 worst) | `create_incident` |
| **Alert** | An automatic signal that a metric crossed a threshold | Tickets start from alerts |
| **p95 latency** | The response time 95% of requests beat | Main symptom metric |
| **Rollback** | Returning a service to its previous version | High-risk tool |
| **Feature flag** | A switch that turns a feature on/off without a deploy | High-risk tool |
| **Failover** | Switching a database to its standby | Critical tool |
| **Runbook** | A written procedure for handling a known problem | `search_runbooks` (untrusted text) |
| **On-call** | The engineer responsible for responding right now | `get_oncall`, `page_oncall` |
| **Maintenance window** | A planned period when degradation is expected | S6 "should not act" scenarios |
| **Status page** | Public page about service health | `#public-status` channel, needs approval |
