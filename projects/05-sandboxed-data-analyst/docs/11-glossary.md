# 11. Glossary

Plain-English definitions of the terms used in these docs. Next.js and AI SDK terms are in the
[Project 6 glossary](../../06-ai-saas-nextjs/docs/11-glossary.md); MCP terms are in the
[Project 2 glossary](../../02-mcp-hub/docs/11-glossary.md).

## Sandboxing and isolation

| Term | Meaning | In this project |
|---|---|---|
| **Sandbox** | An isolated place to run untrusted code | Every `run_python` / `run_sql` |
| **Container** | A process isolated with Linux namespaces and cgroups; shares the host kernel | The gVisor sandbox is a container with a different runtime |
| **runc** | Docker's default container runtime (shares the host kernel directly) | Not used for untrusted code |
| **gVisor / runsc** | Google's user-space kernel that intercepts a container's system calls, so the host kernel is exposed much less | Self-hosted provider |
| **systrap** | gVisor's default way of intercepting syscalls (no KVM needed) | gVisor host config |
| **microVM** | A very small virtual machine that boots in milliseconds | E2B's sandboxes |
| **Firecracker** | The open-source microVM monitor (from AWS) used by E2B and others | E2B provider |
| **KVM** | Linux's built-in hypervisor, used by Firecracker | Provider comparison |
| **E2B** | A managed service that runs code sandboxes in Firecracker microVMs | Default provider |
| **Template** | A pre-built E2B sandbox image | Same packages as the gVisor image |
| **Sandbox escape** | Code inside the sandbox affecting the host or other sandboxes | Escape suite |
| **Egress** | Network traffic leaving the sandbox | Disabled |
| **Metadata endpoint** | `169.254.169.254`, the cloud address that can hand out machine credentials | Attack E3 |
| **Capabilities / no-new-privileges** | Linux privilege switches; dropping them limits what root-like code can do | Container flags |
| **cgroups / rlimits** | Kernel limits on CPU, memory, processes and file size | Resource limits |
| **Fork bomb** | Code that creates processes until the machine stalls | Attack E8 |
| **Warm pool** | Sandboxes created in advance to hide start-up time | Demo |
| **Side channel** | Learning secrets from timing or shared hardware effects | Out of scope |

## Output safety

| Term | Meaning | In this project |
|---|---|---|
| **Output filter** | Broker code that checks everything coming out of a sandbox | Type allow-list, size caps |
| **Magic bytes** | The first bytes of a file that reveal its real type | File type checks |
| **Re-encoding** | Decoding and re-writing an image so hidden payloads don't survive | PNG outputs |
| **Pickle** | Python's object format, which can run code when loaded | Never accepted |
| **Vega-Lite** | A JSON grammar for charts | All interactive charts |
| **Expression interpreter** | Vega's CSP-safe way of evaluating chart expressions without `eval` | Client rendering |
| **CSP** (Content Security Policy) | Browser rules about which scripts and connections a page may use | No `unsafe-eval`, no foreign origins |
| **XSS** | Injecting script into a web page | T7 |

## Analysis

| Term | Meaning | In this project |
|---|---|---|
| **Kernel (Jupyter)** | The Python process that runs cells and keeps variables between them | One per session |
| **Notebook cell** | One piece of executed code with its output | Notebook panel |
| **DuckDB** | An embedded analytical SQL database that reads Parquet fast | `run_sql` |
| **Parquet** | A columnar file format for data | Dataset snapshots, outputs |
| **Reference SQL** | The query that computes a benchmark question's correct answer | AnalystBench gold |
| **AnalystBench-50** | This project's 50-question benchmark | Accuracy numbers |
| **DABstep** | A public benchmark of 450+ multi-step data-analysis tasks with a leaderboard | External number |
| **Self-repair** | The agent reading an error and fixing its own code | Repair loop |
