# F21: Azure Deployment

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M6 | Must | F12, F13 | 4 h | Public demo |

**Goal:** Deploy the gateway (2+ replicas, no session affinity), both servers, Keycloak, OPA and the
ingest jobs to Azure with Terraform, reusing Project 1's modules.

## Diagram: Azure resources

```mermaid
flowchart TB
    subgraph RG["Resource group rg-mcp-hub"]
        subgraph CAE["Container Apps Environment"]
            GW["ca-gateway<br/>2–3 replicas · no affinity<br/>+ OPA sidecar"]
            MF["ca-india-mf<br/>0–2 replicas"]
            FX["ca-fx<br/>0–1 replica"]
            KC["ca-keycloak<br/>1 replica"]
            J1["job: AMFI daily"]
            J2["job: ECB daily"]
        end
        PG[("Postgres Flexible Server<br/>DBs: hub, keycloak")]
        RC[("Azure Cache for Redis")]
        KV["Key Vault<br/>HMAC keys · KEK · client secrets"]
        ACR[Container Registry]
    end
    DNS["hub.&lt;domain&gt; · mf.&lt;domain&gt;"] --> GW & MF
    GW --> MF & FX & PG & RC & KV
    GW -.-> KC
```

## Diagram: deploy pipeline

```mermaid
flowchart LR
    TAG["gateway-v* tag"] --> IMG["images → ACR"]
    IMG --> PLAN["terraform plan (OIDC)"]
    PLAN --> APPR{"manual approval"}
    APPR --> APPLY["terraform apply"]
    APPLY --> SMK["smoke: discover · OAuth · tools/list ·<br/>one call per upstream · audit verify"]
    SMK -->|fail| RB["previous revision"]
```

## Deliverables / files
```
infra/main.tf, variables.tf, envs/prod.tfvars
infra/modules/ (reused from Project 1: container_apps, postgres, redis, keyvault)
infra/modules/keycloak/
docs/runbook.md          # deploy, rollback, rotate HMAC keys, rotate KEK, disable an upstream fast
```

## Tasks
- [ ] Reuse Project 1 modules; add Keycloak and the OPA sidecar
- [ ] Session affinity **off** for the gateway (tests statelessness in the cloud)
- [ ] Secrets via Key Vault references; managed identity
- [ ] Public endpoints: gateway (auth) and india-mf-mcp (anonymous read-only, rate-limited)
- [ ] Budget alert; scale-to-zero where possible
- [ ] Runbook, including "an upstream is compromised: disable it in 1 minute"

## Acceptance criteria
- Claude Desktop connects to the public gateway URL, logs in, and uses tools from 3 upstreams
- The registry entry for india-mf-mcp points to the working public endpoint
- Rollback in < 2 minutes

**Interview talking point:** *"In the cloud the gateway runs without session affinity, the same as
locally, which is the practical proof that the design is stateless."*
