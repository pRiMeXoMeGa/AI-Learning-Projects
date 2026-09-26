# F14: Admin Console

| Milestone | Priority | Depends on | Effort | Unblocks |
|---|---|---|---|---|
| M4 | Should (full plan) | F12 | 7 h | Demo video, screenshots |

**Goal:** A small React + TypeScript app for admins: review and approve tool definitions with a
**side-by-side diff**, manage tenant allow-lists, and browse and **verify** the audit log.

In the **core plan** this feature is replaced by the `hubctl` CLI from F8 (approvals) and the admin API
(audit), and the console moves to "later".

## Diagram: pages

```mermaid
flowchart TB
    APP["Admin console (OIDC login, hub:admin)"] --> P1["Tools"]
    APP --> P2["Tenants"]
    APP --> P3["Audit"]
    APP --> P4["Upstreams"]
    P1 --> P1a["list by status: pending · approved · quarantined"]
    P1 --> P1b["diff view: old vs new definition<br/>+ injection-scan warnings"]
    P1 --> P1c["approve / reject (audited)"]
    P2 --> P2a["allow-list per tenant · confirmation overrides"]
    P3 --> P3a["filter by user, tool, decision, time"]
    P3 --> P3b["verify chain → ✅ / ❌ with first broken row"]
    P4 --> P4a["health · circuit state · last refresh"]
```

## Diagram: data flow

```mermaid
sequenceDiagram
    participant UI as Console (browser)
    participant KC as Keycloak
    participant API as Gateway admin API
    UI->>KC: login (auth code + PKCE, public client)
    KC-->>UI: token (scope hub:admin, aud = admin API)
    UI->>API: GET /admin/api/tools?status=quarantined
    API-->>UI: definitions + diffs
    UI->>API: POST /admin/api/tools/{id}/approve
    API->>API: status change + audit event
    API-->>UI: 200
```

## Deliverables / files
```
console/src/pages/Tools.tsx, Tenants.tsx, Audit.tsx, Upstreams.tsx
console/src/components/DefinitionDiff.tsx
console/src/api/client.ts            # typed client (OpenAPI-generated types)
console/src/auth.ts                  # OIDC PKCE (oidc-client-ts)
console/e2e/approve.spec.ts          # Playwright
```

## Tasks
- [ ] OIDC login with PKCE; tokens kept in memory, not local storage
- [ ] Typed API client generated from the admin API's OpenAPI schema
- [ ] Diff view (JSON, with description text highlighted)
- [ ] Tenant allow-list editor (writes policy data → OPA bundle reload)
- [ ] Audit browser + verify button
- [ ] CSP headers and CSRF-safe design (bearer tokens, no cookies for the API)

## Acceptance criteria
- A quarantined tool from a rug-pull test can be reviewed and approved in the console, and the approval appears in the audit log
- A tenant change takes effect on the next `tools/list` without a redeploy

## Tests
- vitest for components; Playwright end-to-end for the approval flow against the compose stack

**Interview talking point:** *"The console turns the security model into something an admin can operate:
you see exactly what changed in a tool's description before you let agents use it again."*
