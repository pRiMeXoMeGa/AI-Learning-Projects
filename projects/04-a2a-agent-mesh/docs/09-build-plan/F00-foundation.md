# F0: Foundation & Spikes

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M1 | Must | — (Project 3 M1–M2 done) | 3.5 h | Everything |

**Goal:** A uv workspace and Compose stack that runs Project 3's services and Project 2's Keycloak next to
the new agents, each on its own hostname. Three spikes settle the risky assumptions before anything is
built on them.

## Diagram: local stack

```mermaid
flowchart TB
    subgraph Compose["docker compose (network: mesh)"]
        subgraph New["this project"]
            CMD["commander.mesh.local"]
            TRA["triage.mesh.local"]
            COM["comms.mesh.local"]
            PMA["postmortem.mesh.local"]
            REG["registry.mesh.local"]
            KB["kb-mcp.mesh.local"]
        end
        subgraph Reused["reused images"]
            OPS["opssim + approvals (P3)"]
            KC["keycloak (P2 realm + agent clients)"]
        end
        PG[("postgres 16 + pgvector")]
        LF["langfuse"]
        TX["toxiproxy (profile: resilience)"]
        ROG["rogue agents (profile: security)"]
    end
```

## Diagram: spikes (1 hour each, results in `docs/notes/spikes.md`)

```mermaid
flowchart LR
    S1["S1 · ADK RemoteA2aAgent vs an a2a-sdk 1.x server:<br/>stream · auth-required · push config · SubscribeToTask"] --> R1{"all work?"}
    R1 -- no --> FB1["plan: Commander tool using<br/>the a2a-sdk client for gaps"]
    S2["S2 · a2a-sdk signing extra:<br/>detached JWS over JCS, verify with joserfc"] --> R2{"spec-compliant?"}
    R2 -- no --> FB2["plan: joserfc + rfc8785"]
    S3["S3 · Keycloak delegation exchange:<br/>subject + actor token → act claim;<br/>standard exchange rejects act tokens"] --> R3{"works on pinned version?"}
    R3 -- no --> FB3["plan: custom mapper (weaker),<br/>stated in the report"]
```

## Deliverables / files
```
pyproject.toml (uv workspace)  docker-compose.yml  Makefile (up, down, test, tck, security, resilience)
.env.example   config/models.yaml   config/agents.yaml (hostnames, audiences)
keycloak/realm-mesh.json         # agent clients, token-exchange permissions, delegation feature flags
docs/notes/spikes.md
.github/workflows/ci.yml
```

## Tasks
- [ ] Workspace packages: `commander`, `agents/*`, `registry`, `kb`, `rogue`, `evals`; Project 3 packages installed as dependencies
- [ ] Compose with per-service hostnames (TLS optional locally; HTTPS in the cloud)
- [ ] Keycloak: realm import with one client per agent; delegation + parameterized-scopes features enabled; pinned version
- [ ] Spikes S1–S3 with written outcomes and chosen fallbacks
- [ ] CI: lint, types, unit tests

## Acceptance criteria
- `make up` → all containers healthy; P3's OpsSim and approval service reachable from the agent containers
- Spike notes committed with a decision for each

## Tests
- Smoke test: each hostname answers `/healthz`; Keycloak issues a client-credentials token for each agent

**Interview talking point:** *"I spent the first three hours proving the three riskiest assumptions: ADK's
A2A 1.0 client, spec-compliant card signing and Keycloak's delegation tokens. Each one had a fallback
decided before I built anything on it."*
