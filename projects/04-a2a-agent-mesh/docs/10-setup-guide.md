# 10. Setup Guide: accounts, keys and the first run

> **Status:** No code exists yet. This page is the **target setup**: commands become real as F0–F7 are
> built and may change slightly. Keep it in sync with the repo.

## 10.1 What you need, and when

| Needed from | Tool / account | Used for | Cost |
|---|---|---|---|
| **M1** | Everything from Project 3's setup (Docker, uv, Anthropic + OpenAI keys, Langfuse) and its built images | OpsSim, approvals, Triage Agent | As in Project 3 |
| M1 | Project 2's Keycloak image and realm export | Identity | Free |
| M1 | Node (for `npx`) or the A2A Inspector container | Manual debugging | Free |
| M1 | **A2A TCK** (`git clone a2aproject/a2a-tck`, `uv pip install -e .`) | Conformance | Free |
| **M3** | **Google AI Studio (Gemini) API key**, or Vertex AI | Commander model (Claude via LiteLlm as fallback) | Free tier / pay per use |
| M4 | **Toxiproxy** (container, in the resilience profile) | Fault injection | Free |
| Optional | Azure subscription + Terraform | F19 deployment | Low with scale to zero |

Docker needs about **8 GB RAM**: Keycloak, Postgres, Langfuse and six Python services.

## 10.2 Environment variables (`.env`)

```bash
# --- Reused from Projects 2–3 ---
DATABASE_URL=postgresql://mesh:mesh@postgres:5432/mesh
OPSSIM_URL=http://opssim.mesh.local:8100
APPROVALS_URL=http://approvals.mesh.local:8200
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=http://langfuse:3000

# --- Commander model (M3) ---
GOOGLE_API_KEY=                          # Gemini via AI Studio (or configure Vertex)
COMMANDER_MODEL=gemini-flash-tier        # pinned dated ID in config/models.yaml

# --- Identity (M2) ---
KEYCLOAK_URL=http://keycloak.mesh.local:8080
KEYCLOAK_REALM=mesh
COMMANDER_CLIENT_ID=commander
COMMANDER_CLIENT_SECRET=
# each agent validates tokens for its own audience: triage | comms | postmortem

# --- Card signing (M1) ---
ORG_SIGNING_KEY_PATH=./secrets/shoplite-es256.pem    # never committed
ORG_JWKS_URL=http://registry.mesh.local/orgs/shoplite/jwks.json

# --- Safety limits ---
MESH_MAX_COST_USD=5                      # per eval command
MESH_STUB_MODELS=1                       # default for tests; set 0 for live runs
```

Generate an ES256 key with `openssl ecparam -name prime256v1 -genkey -noout -out secrets/shoplite-es256.pem`.

## 10.3 First run

```mermaid
flowchart LR
    A["1 · Project 3 images<br/>built"] --> B["2 · .env + keys"]
    B --> C["3 · make up<br/>(stub models)"]
    C --> D["4 · register + approve<br/>agents in the registry"]
    D --> E["5 · make tck"]
    E --> F["6 · mesh incident ALR-7781<br/>(stub models)"]
    F --> G["7 · live run<br/>MESH_STUB_MODELS=0"]
```

```bash
# 1–3
make up                                   # stub-model agents by default

# 4. Register and approve the agents (prints card diffs before approval)
uv run meshctl agents register http://triage.mesh.local
uv run meshctl agents register http://comms.mesh.local
uv run meshctl agents register http://postmortem.mesh.local
uv run meshctl agents approve --all

# 5. Conformance
make tck

# 6. One incident, stub models
uv run mesh incident ALR-7781

# 7. One live incident (cheap models)
MESH_STUB_MODELS=0 uv run mesh incident ALR-7781 --max-cost 1
```

## 10.4 Keeping costs safe

| Control | Setting |
|---|---|
| Stub models by default | `MESH_STUB_MODELS=1`; protocol, trust and resilience tests never call a model |
| Cost cap | `--max-cost` on every live command (default `MESH_MAX_COST_USD`) |
| Cheap models for development | Project 3's cheap tier for remote agents; the Gemini Flash tier for the Commander |
| Provider limits | Monthly limits set in the Anthropic, OpenAI and Google consoles |

## 10.5 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Registry rejects every card | Wrong `ORG_JWKS_URL` or the key isn't in the trusted org list | Check `meshctl orgs list`; re-sign cards after key changes |
| Agents return 401 to the Commander | Audience or scope mismatch; exchange not permitted | Check the token's `aud`/`scope`; realm exchange permissions (F6) |
| No `act` claim in exchanged tokens | Delegation feature flags off, or using the standard exchange | Enable `token-exchange-delegation,parameterized-scopes`; send `actor_token` |
| ADK can't reach a peer | Peer quarantined, or the URL was used instead of the registry card | `meshctl agents list`; re-approve after a card change |
| Stream drops during long approvals | Proxy/idle timeouts | Clients re-subscribe (`SubscribeToTask`); check that push is configured |
| TCK fails on version tests | Missing `A2A-Version` handling | See F1's version check |
| Claude SDK agent fails in its container | Session directory / subprocess setup | Same fixes as Project 3's setup guide |
