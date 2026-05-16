# SAP Go-Live Agent — Built on BrowserOS

## What We Are Building

A domain-specific AI agent layer on top of **BrowserOS** (open-source Chromium fork) that automates SAP BTP and Fiori go-live tasks for consultants using plain English instructions.

A consultant types:
> "Deploy this BTP app, assign Fiori catalog to ZROLE, run smoke test on 10 apps"

The agent executes it — no CLI, no code, no manual clicking.

---

## The Problem We Are Solving

SAP go-live phases are brutal. Consultants spend days or weeks:
- Deploying BTP apps one by one through BTP Cockpit UI
- Assigning Fiori catalogs to business roles in Launchpad Designer
- Configuring destinations and binding services manually
- Running smoke tests across 30+ Fiori apps
- Checking IDoc errors in WE02
- Setting up subaccounts, entitlements, service instances
- Configuring CPI iFlows

All of it is manual, repetitive, browser-based, and high-stakes. One wrong click in production causes incidents.

**Existing tools do not solve this:**
- SAP MCP servers target developers writing code in IDEs — not consultants clicking through web UIs
- Generic browser agents break on SAP UI5's dynamic shadow DOM components
- No SAP-specific agentic product exists for the go-live configuration phase

---

## Our Solution: Hybrid Agent Architecture

Two execution paths, one agent, one natural language interface:

```
Consultant types instruction
          │
    Agent Orchestrator (LLM)
    /                      \
Task has BTP/CF API?     Task is UI-only?
          │                      │
      MCP Layer             BrowserOS Layer
          │                      │
  CF deploy API         Fiori Launchpad Designer
  Destination API        LTMC wizard steps
  XSUAA SCIM             CPI iFlow screens
  Job Scheduler          WE02 IDoc monitor
  Launchpad API          BAS configurations
```

**If BTP has a REST API for it** → route to MCP tool (fast, stable, auditable)

**If it is UI-only** → route to browser agent (clicks through SAP web UI using semantic selectors)

**If it is both** → split into subtasks, route each appropriately

---

## Base: BrowserOS

**Repo:** https://github.com/browseros-ai/BrowserOS

BrowserOS is an open-source Chromium-based browser that runs AI agents natively. Key properties relevant to our use case:

- Supports Claude, OpenAI, Gemini, or local models via Ollama
- Built-in MCP server support (connect custom MCP servers with one click)
- `browseros-cli` for terminal and agent-driven control
- Consultant's existing SSO session is reusable — no re-authentication needed
- AGPL-3.0 licensed, self-hostable, no external data transmission
- Runtime: Bun + TypeScript (agent server), Go (CLI), WXT + React (extension)

We fork BrowserOS and add a `packages/sap-agent` workspace on top of it.

---

## Repository Structure

```
BrowserOS/
├── packages/
│   ├── sap-agent/                        ← our addition
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── types.ts
│   │   │   ├── config/
│   │   │   │   └── sap.config.ts
│   │   │   ├── session/
│   │   │   │   └── SAPSessionBridge.ts
│   │   │   ├── router/
│   │   │   │   └── TaskRouter.ts
│   │   │   ├── approval/
│   │   │   │   └── ApprovalGate.ts
│   │   │   ├── audit/
│   │   │   │   └── AuditLogger.ts
│   │   │   ├── mcp/
│   │   │   │   └── SAPMCPServer.ts
│   │   │   ├── ui5/
│   │   │   │   └── UI5Adapter.ts
│   │   │   ├── playbooks/
│   │   │   │   ├── PlaybookEngine.ts
│   │   │   │   └── library/
│   │   │   │       ├── deploy-btp-app.yaml
│   │   │   │       ├── assign-fiori-catalog.yaml
│   │   │   │       ├── run-smoke-test.yaml
│   │   │   │       ├── check-idoc-errors.yaml
│   │   │   │       ├── configure-cpi-iflow.yaml
│   │   │   │       ├── create-btp-destination.yaml
│   │   │   │       ├── assign-xsuaa-role.yaml
│   │   │   │       ├── create-service-instance.yaml
│   │   │   │       ├── mtls-cert-rotation.yaml
│   │   │   │       └── landscape-smoke-suite.yaml
│   │   │   └── runtime/
│   │   │       └── SAPAgentRuntime.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── browseros-agent/                  ← existing BrowserOS packages
│       ├── apps/
│       │   ├── server/
│       │   │   └── src/api/routes/
│       │   │       └── sap.ts            ← /sap/* routes (our addition)
│       │   ├── agent/
│       │   │   └── entrypoints/
│       │   │       ├── background/
│       │   │       │   └── sap-session-bridge.ts  ← our addition
│       │   │       └── sidepanel/
│       │   │           └── components/
│       │   │               ├── ApprovalPrompt.tsx  ← our addition
│       │   │               └── ReauthPrompt.tsx    ← our addition
│       │   └── cli/
```

---

## Six Build Phases

### Phase 1 — Fork Setup and Hook Mapping

**What:** Add `packages/sap-agent` workspace to the BrowserOS monorepo. Map all hook points in the existing agent loop where SAP orchestration should attach.

**Deliverables:**
- `packages/sap-agent` scaffolded with package.json, tsconfig.json, src/index.ts, src/types.ts
- `sap.config.ts` with fields: landscape (dev/qa/prod), cf_api_endpoint, btp_subdomain, region
- `HOOKS.md` documenting exact file and line where each phase integrates into BrowserOS internals
- `/sap/*` routes mounted in `apps/server/src/api/server.ts`

**Complexity:** LOW

---

### Phase 2 — SAP Session Bridge

**What:** Reuse the consultant's existing BTP/Fiori SSO session from their active Chromium browser. No stored credentials. No re-login.

**How it works:**
1. Background script in Chrome extension intercepts active BTP Cockpit and Fiori session tokens
2. Extracts CF OAuth token, XSUAA token, and session cookies — memory only, never written to disk
3. Pushes session to agent server via `POST /sap/session`
4. Session watchdog polls token expiry every 60 seconds
5. On expiry: pauses all running tasks, emits `SAP_SESSION_EXPIRED` to sidepanel
6. Sidepanel shows ReauthPrompt — consultant clicks Re-authenticate, opens BTP login tab
7. On new login detected: background re-captures tokens, resumes paused tasks

**Key interface:**
```typescript
interface SAPSession {
  cfToken: string
  xsuaaToken: string
  cookies: Record<string, string>
  expiresAt: number
}
```

**Files:**
- `packages/sap-agent/src/session/SAPSessionBridge.ts`
- `apps/agent/entrypoints/background/sap-session-bridge.ts`
- `apps/agent/entrypoints/sidepanel/components/ReauthPrompt.tsx`

**Complexity:** MEDIUM

---

### Phase 3 — Task Router

**What:** Classify every incoming consultant instruction into TYPE_API, TYPE_UI, or TYPE_HYBRID. Route each subtask to the correct execution layer.

**Classification logic (in priority order):**
1. Rule-based lookup against `KNOWN_API_OPS` registry (fast, deterministic)
2. LLM fallback for unknown tasks — JSON-only structured output, 10s timeout

**KNOWN_API_OPS registry:**
```
deploy-mta, bind-service, set-destination, assign-role,
create-service-instance, delete-service-instance,
schedule-job, assign-fiori-catalog (when API available)
```

**Hard constraints on routing:**
- Destructive ops (delete, undeploy, unassign) → TYPE_API only, never TYPE_UI
- HYBRID tasks → split into ordered subtask array with dependsOn DAG

**Key interface:**
```typescript
interface RoutedTask {
  original: string
  type: 'API' | 'UI' | 'HYBRID'
  subtasks: SubTask[]
  requiresApproval: boolean
}
```

**LLM fallback system prompt:**
> You are a SAP BTP task classifier. Output ONLY JSON: { type, subtasks[], requiresApproval }. API = achievable via REST API. UI = requires browser navigation. requiresApproval = true if task involves deploy, delete, assign, configure, bind.

**Complexity:** MEDIUM

---

### Phase 4 — SAP UI5 Adapter

**What:** Custom DOM interaction layer for SAP UI5. Replaces generic CSS selector strategy with semantic, stable selectors that survive BTP Cockpit and Fiori UI updates.

**Selector priority order (never deviate from this):**
1. `aria-label` + UI5 control type
2. `data-sap-ui` attribute
3. Stable ID patterns only if confirmed stable
4. **NEVER** auto-generated CSS classes (`.sapMBtn`, `.sapUiTableCell` are unstable)

**`waitForUI5Ready` — three conditions must all pass:**
1. `sap.ui.getCore().isInitialized() === true` (injected via page.evaluate)
2. No elements with `data-sap-ui-busy="true"` present
3. Network idle

**Self-healing:**
- On StaleElementReferenceError: re-query and retry once
- On second failure: throw `UI5ElementNotFoundError` with screenshot attached

**Screenshot on failure:**
- Capture `sap-agent/logs/ui5-fail-[timestamp].png` on every `UI5ElementNotFoundError`
- Annotate with red border around failing element region

**Complexity:** HIGH

---

### Phase 5 — Approval Gate and Audit Log

**What:** Intercept every destructive operation before execution. Surface approval prompt to consultant. Log every action.

**Approval gate rules:**
- Triggers on any action containing: deploy, undeploy, delete, assign, unassign, bind, unbind, configure, set
- Shows consultant: action + target + landscape + system + estimatedEffect
- Waits for explicit Approve or Cancel — no timeout, no auto-approve, no bypass flag — ever
- On Cancel: abort task, log CANCELLED
- On Approve: execute, log APPROVED then COMPLETED or FAILED

**Sidepanel ApprovalPrompt.tsx:**
- Renders at TOP of sidepanel, above all other content
- Multiple pending approvals stack in order
- Sends `SAP_APPROVAL_RESPONSE { approved: boolean, taskId }` back to server

**Audit log format (JSONL):**
```json
{
  "timestamp": "2025-05-16T10:23:00Z",
  "taskId": "uuid",
  "actor": "agent",
  "action": "cf_deploy_mta",
  "target": "app-name",
  "landscape": "dev",
  "status": "APPROVED",
  "error": null
}
```

**Complexity:** MEDIUM

---

### Phase 6 — SAP MCP Server and Playbook Engine

**What:** Custom MCP server wrapping BTP/CF REST APIs. Playbook engine that runs ordered step templates for common go-live sequences.

**MCP Tools (all inputs Zod-validated, all auth from SAPSessionBridge):**

| Tool | API | Method |
|------|-----|--------|
| `cf_deploy_mta` | CF API v3 `/v3/deployments` | POST + poll |
| `cf_bind_service` | CF API v3 `/v3/service_bindings` | POST |
| `cf_set_env` | CF API v3 `/v3/apps/{guid}/environment_variables` | PATCH |
| `btp_set_destination` | Destination Service API | POST |
| `xsuaa_assign_role` | XSUAA SCIM `/Groups/{groupId}` | PATCH |
| `launchpad_assign_catalog` | Launchpad Service API | POST |
| `job_schedule` | Job Scheduling Service API | POST |

**All MCP tools must:**
- Source all auth headers from `SAPSessionBridge.getSession()`
- Throw typed `SAPAPIError { code, message, httpStatus, sapErrorCode }` on non-2xx
- Retry 3x with exponential backoff on 429 and 503
- Emit progress events during long-running ops (MTA deploy polling)
- Return `{ success: boolean, result: any, summary: string }`
- Fall back to UI route if API returns 501 or 404 (Launchpad catalog assignment)

**Playbook format (YAML):**
```yaml
name: deploy-btp-app
description: Deploy MTA to CF space, bind services, set destinations
steps:
  - id: deploy
    type: API
    tool: cf_deploy_mta
    params:
      mtaPath: "{{mtaPath}}"
      cfSpace: "{{cfSpace}}"
  - id: bind
    type: API
    tool: cf_bind_service
    dependsOn: [deploy]
    params:
      appGuid: "{{deploy.result.appGuid}}"
      serviceInstanceGuid: "{{serviceInstanceGuid}}"
```

**Built-in playbooks:**
1. `deploy-btp-app.yaml` — CF push + bind services + set destinations
2. `assign-fiori-catalog.yaml` — assign catalog to role via API (UI fallback)
3. `run-smoke-test.yaml` — open each Fiori app, assert it loads, return pass/fail
4. `check-idoc-errors.yaml` — navigate WE02, filter errors, return count and list
5. `configure-cpi-iflow.yaml` — open iFlow, set endpoint params, deploy
6. `create-btp-destination.yaml` — POST to Destination Service, verify connectivity
7. `assign-xsuaa-role.yaml` — SCIM assign role collection to user
8. `create-service-instance.yaml` — CF API v3 create instance + binding
9. `mtls-cert-rotation.yaml` — update cert in credential store + rebind
10. `landscape-smoke-suite.yaml` — run smoke test across dev/qa/prod in sequence

**Complexity:** HIGH

---

## What the Consultant Experiences

1. Opens BrowserOS (our fork) — already logged into BTP and Fiori via SSO
2. Types in the agent chat: *"Deploy app-hr to CF dev space, bind to HANA instance, assign Fiori catalog HR_CATALOG to role ZROLE_HR, run smoke test"*
3. Agent classifies task → HYBRID
4. Agent splits into subtasks: deploy (API), bind (API), assign catalog (UI fallback), smoke test (UI)
5. Before deploy: approval card appears at top of sidepanel
   > "I will deploy app-hr v1.2 to CF space dev in landscape DEV. Approve?"
6. Consultant clicks Approve
7. Agent executes — streams progress back to chat
8. Smoke test runs — agent navigates each Fiori app, returns pass/fail table
9. Audit log records every action with timestamp and outcome

---

## What Maps to API vs UI

| Go-Live Task | Execution Layer |
|---|---|
| Deploy MTA/CAP app | API — CF API v3 |
| Create BTP destination | API — Destination Service |
| Bind service instances | API — CF API v3 |
| Assign role collection to user | API — XSUAA SCIM |
| Create service instance | API — CF API v3 |
| Schedule background job | API — Job Scheduling Service |
| Assign Fiori catalog to business role | UI — Launchpad Designer |
| Launchpad tile group config | UI — Launchpad Designer |
| LTMC migration steps | UI — LTMC web cockpit |
| IDoc monitor (WE02/WE05) | UI — SAP Web GUI / Fiori |
| CPI iFlow configuration | UI — CPI web editor |
| Smoke test navigation | UI — Fiori Launchpad |

---

## Non-Negotiable Engineering Rules

1. **No credentials to disk.** Session tokens in memory only. Redacted in all logs.
2. **Approval gate cannot be bypassed.** No flag, no env var, no config option disables it.
3. **Semantic UI5 selectors only.** Never auto-generated CSS class names.
4. **All MCP tool inputs Zod-validated** before any execution.
5. **Errors are typed and thrown.** Never swallowed, never console.log and continue.
6. **One confirmation before destructive or irreversible operations.** Never assume.
7. **SAP terminology is precise.** CF space not environment. MTA not package. Role collection not role. Landscape not tier. Transport not deploy (for ABAP context).
8. **Auth always from SAPSessionBridge.** No hardcoded tokens anywhere in codebase.

---

## Config Required Before Live API Calls

Jules must ask for these before implementing Phase 6 live calls:

- CF API endpoint (e.g. `api.cf.eu10.hana.ondemand.com`)
- BTP subdomain
- CF region (`eu10` / `us10` / `ap10` / etc.)
- XSUAA service instance name
- Launchpad service plan (`standard` or `advanced` — different API shapes)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Browser runtime | BrowserOS (Chromium fork, AGPL-3.0) |
| Agent server | Bun + TypeScript |
| Browser extension | WXT + React |
| CLI | Go |
| Browser automation | Playwright |
| Input validation | Zod |
| MCP protocol | @browseros-ai/agent-sdk |
| Playbook format | YAML |
| Audit log format | JSONL |
| AI model | Claude / GPT / local via Ollama (BYOK) |

---

## Proof-of-Concept Target

Before anything else, get this single path working end-to-end:

```
POST /sap/execute
Body: { task: "Deploy MTA to CF dev space" }

Expected flow:
1. Router classifies → TYPE_API → cf_deploy_mta tool
2. ApprovalGate intercepts → approval card in sidepanel
3. Consultant approves
4. cf_deploy_mta executes against real CF API
5. Progress streamed back to chat
6. AuditLogger records APPROVED + COMPLETED
7. Result returned to consultant
```

Everything else is built on top of this working signal.

---

## Why This Exists

MCP covers developers building SAP apps in IDEs.

This covers **functional and technical SAP consultants executing go-live configurations in a browser** — a completely different persona, a completely different phase of the project, and a gap nobody in the SAP ecosystem has productized yet.

That is the white space this project fills.
