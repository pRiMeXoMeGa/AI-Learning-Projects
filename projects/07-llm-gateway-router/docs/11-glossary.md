# 11. Glossary

Plain-English definitions of the terms used in these docs, and where each one shows up in this project.

## Gateway

| Term | Meaning | In this project |
|---|---|---|
| **LLM gateway** | A proxy between apps and model providers that adds control (keys, budgets, caching, routing, logging) | Switchboard |
| **OpenAI-compatible API** | The Chat Completions request/response format most tools can talk to | Front door |
| **Virtual key** | A key the gateway issues to an app; maps to real provider keys the app never sees | Per app/team |
| **Model alias** | A friendly name (`fast`, `smart`, `auto`) mapped by policy to real models | Routing |
| **Budget** | A spending limit per key/team over a period | Reserve-then-reconcile |
| **Rate limit (RPM/TPM)** | Requests or tokens allowed per minute | Per key |
| **Retry with jitter** | Trying again after a random short delay, so clients don't all retry at once | Reliability |
| **Circuit breaker** | Stops sending to a failing provider for a while, then tests it again | Per provider/model |
| **Fallback** | Sending the request to another model when the first fails | Alias chains |
| **Hedged request** | Sending a duplicate request if the first is slow; the first to finish wins | Could |
| **TTFT** | Time to first token | Latency metric |
| **Ledger** | The record of every request's tokens and cost | Postgres table |

## Caching

| Term | Meaning | In this project |
|---|---|---|
| **Prompt caching (provider)** | The provider stores a long, repeated prompt prefix and charges much less to reuse it | Helper keeps prefixes stable |
| **Cached input tokens** | Input tokens billed at the cheaper cached rate | Cost maths |
| **Exact cache** | Returns a stored answer only for an identical request | Redis |
| **Semantic cache** | Returns a stored answer for a *similar* request, based on embedding similarity | pgvector, opt-in |
| **Threshold (τ)** | The minimum similarity for a semantic hit | Chosen per app from the study |
| **False hit** | A cache hit that returns an answer that's wrong for the new question | Key risk metric |
| **Verify-on-hit** | A cheap model checks that a cached answer fits the new question | High-risk apps |
| **Cache scope** | The exact-match fields (tenant, model, system prompt, tools) that must match before similarity is even checked | Isolation |
| **Cache poisoning / collision** | Getting a malicious or wrong answer stored so others receive it | P2–P4 tests |

## Routing

| Term | Meaning | In this project |
|---|---|---|
| **Model routing** | Choosing which model answers each request | Router |
| **Cascade** | Try a small model first; escalate to a bigger one if needed | Policy |
| **RouteLLM** | An open-source learned-router framework (LMSYS) | Off-the-shelf baseline |
| **Oracle router** | The best possible choice per request, known only in hindsight from labels | Upper bound |
| **Pareto curve** | The best quality achievable at each cost level | Policy comparison |
| **APGR** | Average performance gap recovered: how much of the small-to-big quality gap a router closes | Routing metric |
| **ONNX** | A portable format for running small ML models fast | Router model |

## Security and operations

| Term | Meaning | In this project |
|---|---|---|
| **Supply-chain attack** | Compromising software you depend on, rather than you directly | LiteLLM March 2026 |
| **`.pth` file** | A Python file that can run code automatically when Python starts | Used in the LiteLLM attack; CI check |
| **Pinned by SHA** | Referring to an exact commit of a CI action, not a movable tag | CI hardening |
| **Trusted publishing (OIDC)** | Publishing packages without long-lived tokens | If anything is published |
| **Denial of wallet** | Making a service spend lots of money instead of crashing it | Limits and budgets |
| **Trace replay** | Re-sending recorded real requests to test a system | Evaluation |
