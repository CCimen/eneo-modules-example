# Eneo Platform Expansion — Definitive Architecture v3.9 (Final)

**Date:** 2026-02-25
**Status:** Final — v3.9 aligns auth/session lifecycle, module versioning, compose-first operations, and implementation guardrails
**Scope:** Module system, Flöden (flows), widget, deployment, auth — single release

---

## Executive Summary

Architecture for three capabilities: module system, sequential AI flows ("Flöden"), embeddable widget. Validated by independent architectural reviews and aligned to municipal Docker Compose operations.

**Key insights:**
- API Key v2 eliminates custom widget auth (~500 LOC removed)
- Inline Flow Builder creates hidden assistants behind the scenes (one-page UX)
- Flow is its own DDD aggregate root (not nested inside Space)
- Svelte Flow visualizer + JSON export for municipal compliance/transparency
- Per-module image versioning enables independent module upgrades
- Auth Broker token relay uses module-scoped JWTs (no custom user-id trust headers)
- Docker Compose is primary deployment model for municipalities (K8s reference-only)

**All phases merged.** PRs sequenced for reviewability, security, and operational clarity.

---

## Part 1: Design Philosophy

### The Three Pillars

| Pillar | Principle | Effect |
|--------|-----------|--------|
| **BFF Pattern** | Modules handle own routes, call backend internally | Core compose never changes |
| **Auth Broker + Ticket Handoff** | Backend is the only OIDC client; modules exchange one-time tickets server-to-server | Module isolation preserved, no shared cookies |
| **API Key v2** | `sk_` keys for service auth | Zero new auth infrastructure |

### The Flow Builder Philosophy

One page. Define everything inline. System creates hidden assistants transparently.

"Use existing assistant" is the advanced path for reusable specialists.

---

## Part 2: Module System Architecture

### 2.1 BFF Pattern

Every module is a server-rendered BFF. Browser never talks to backend on module domains.

### 2.2 Module Contract

| Requirement | Detail |
|-------------|--------|
| Serves UI at `/` | Root path, no prefix |
| `/health` endpoint | 200 OK when healthy |
| `ENEO_BACKEND_URL` env | Server-side only |
| Own `/api/*` routes | BFF: proxies to backend via module_net |
| Delegates auth | Auth broker ticket handoff (production) |
| `module.json` manifest | id, name, version, port, health |

### 2.3 Subdomain Routing

```
eneo.sundsvall.se           → Eneo UI + Backend
taltilltext.eneo.sundsvall.se → Speech-to-text module
widget.eneo.sundsvall.se      → Widget host module
```

### 2.4 Module Health Registry

Admin-registered (not auto-discovery). `module_registry` table. Background health poll every 60s.

---

## Part 3: Authentication

### 3.1 User Auth — Auth Broker + Ticket Handoff (Production Default)

**No cross-subdomain shared auth cookie.** Modules do NOT inherit a session cookie from `.eneo.sundsvall.se`. The backend is the centralized OIDC client, and modules use ticket handoff + module-scoped JWT.

**Flow:**

```
Browser → module.eneo.sundsvall.se/login
  → Module BFF creates random handoff_state and sets temporary __Host-handoff cookie
  → 302 to eneo.sundsvall.se/api/v1/auth/module-initiate?module_client_id=speech-to-text&handoff_state=<nonce>&return_to=<path>
    → Backend creates state_id and stores OIDC state payload in Redis (tenant_id, module_client_id, nonce, pkce_verifier, return_to), TTL 10 minutes
    → 302 to IdP (OIDC login)
    → IdP callback to /api/v1/auth/callback?state=<state_id>
      → Backend GETDELs state payload from Redis (single use)
      → Backend checks state payload:
         IF module_client_id present:
           → Mint one-time ticket (Redis, 10–30s TTL, atomic GETDEL consume)
           → 302 to module callback_url?ticket=<ticket>&handoff_state=<nonce>
         IF NO module_client_id (core web login):
           → Return JSON { "access_token": "..." } (existing flow, unchanged)
      → Module BFF receives ticket + handoff_state
      → Module BFF verifies handoff_state matches __Host-handoff cookie
      → Module BFF calls POST /api/v1/auth/ticket-exchange
         with: { "ticket": "<ticket>" }
         Authorization: Bearer sk_<module_service_key>
      → Backend validates: ticket exists, not expired, not replayed,
         module_client_id matches sk_ key's registered module
      → Backend returns module-scoped JWT (sub=user_id, aud=module_client_id, scope, auth_time, iat, exp=15m)
      → Module sets host-only __Host- cookie on its own domain
```

**Key security properties:**

1. **Module-scoped session token:** The JWT returned by ticket-exchange is scoped to `module_client_id` + constrained permissions/audience. A compromised module session cannot be reused as a full core session.

2. **One-time ticket controls:**
   - TTL 10–30 seconds
   - Atomic consume-once in Redis (`GETDEL`)
   - Bound to `tenant_id` + `module_client_id`
   - Reject replay and client mismatch

3. **Module callback CSRF protection:** `handoff_state` MUST roundtrip through broker redirects and match temporary `__Host-handoff` cookie before ticket exchange.

4. **Exchange endpoint protection:** `POST /api/v1/auth/ticket-exchange` requires the module's `sk_` service key in the Authorization header. The browser never directly exchanges tickets — only the module's BFF (server-side) does.

5. **Leakage mitigation:**
   - Callback responses include `Cache-Control: no-store`
   - Callback page immediately clears query ticket via redirect
   - Strict `Referrer-Policy: no-referrer` on callback routes

6. **Open-redirect prevention:** Redirect targets must come from registered module client allowlist only (exact match, no wildcards).

**Threat model:** If a module is fully compromised, the attacker obtains a module-scoped JWT that can only access endpoints allowed for that specific module. They cannot escalate to the core Eneo admin panel or access other modules' data.

**Backwards compatibility:** The existing `/api/v1/auth/callback` JSON token flow for the core web app continues to work unchanged. The callback handler branches on whether `module_client_id` is present in the stored state payload.

### 3.2 Auth Broker API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/auth/module-initiate` | GET | Start module auth flow. Params: `module_client_id`, `return_to`, `handoff_state` |
| `/api/v1/auth/ticket-exchange` | POST | Exchange one-time ticket for module-scoped JWT. Requires `sk_` key. |
| `/api/v1/auth/module-refresh` | POST | Refresh module-scoped JWT. Requires `sk_` key + expiring module JWT. |
| `/api/v1/auth/module-logout` | POST | Optional: revoke module session, clear module cookie |
| `/api/v1/auth/callback` | GET | Existing OIDC callback — now branches: JSON (core) or 302+ticket (module) |

### 3.3 Module Client Registration

Each module must be registered in a `module_clients` configuration (table or JSONB per tenant):

| Field | Type | Description |
|-------|------|-------------|
| `client_id` | str | Unique module identifier (e.g., `speech-to-text`) |
| `callback_url` | str | Exact-match callback URL for redirect (e.g., `https://taltilltext.eneo.sundsvall.se/auth/callback`) |
| `active` | bool | Whether this module client is enabled |
| `allowed_scopes` | list | Optional: constrained permission set for module-scoped JWTs |
| `module_api_contract` | str | Required API contract identifier (e.g., `core-api-v1`) |
| `sk_key_id` | FK | Links to the module's `sk_` API key for exchange endpoint auth |

### 3.4 Module-to-Backend Auth Patterns

API Key v2 (`sk_` keys). Scoping, IP allowlists, rate limiting, audit, lifecycle.

| Scenario | Mechanism |
|----------|-----------|
| User visits module | Auth broker ticket handoff → module-scoped JWT |
| Module BFF → Backend (user context) | `Authorization: Bearer <module-scoped-jwt>` |
| Module BFF → Backend (system context) | `Authorization: Bearer sk_<module_key>` |
| Widget BFF (public) | `sk_` key from env (no user auth cookie) |
| External API | `sk_`/`pk_` key |

The backend never trusts custom user identity headers for authorization decisions. User identity is extracted from signed JWT claims (`sub`, `aud`, `tenant_id`, `scope`).

### 3.5 Session Lifecycle

- Module-scoped JWT TTL: 15 minutes
- Refresh grace window: 5 minutes after `exp`
- Absolute session timeout: 12 hours via immutable `auth_time`
- Revocation watermark: backend enforces `iat > logout_after`
- Refresh denial (expired absolute window, revoked user, suspended tenant) clears module cookie and forces re-authentication

### 3.6 Security Hardening — Auth Broker

| Control | Detail |
|---------|--------|
| OIDC state transport | Compact `state_id` in callback URL; state payload in Redis, TTL 10 minutes |
| OIDC state consume | Redis `GETDEL` (single use) |
| Callback CSRF | `handoff_state` must match `__Host-handoff` cookie before ticket exchange |
| Ticket TTL | 10–30 seconds, configurable per deployment |
| Ticket TTL start time | Starts when backend mints ticket in callback handler |
| Ticket storage | Redis with atomic consume-once |
| Ticket binding | `tenant_id` + `module_client_id` |
| Exchange auth | Module `sk_` key required (server-to-server only) |
| Token scoping | JWT `aud` = module_client_id, constrained permissions |
| Redirect allowlist | Exact-match from `module_clients.callback_url` |
| Cache headers | `Cache-Control: no-store` on all callback responses |
| Referrer policy | `Referrer-Policy: no-referrer` on callback routes |
| Core login compat | Existing `/api/v1/auth/callback` JSON flow unchanged |

---

## Part 4: Network Architecture

| Network | Services | `internal` | Purpose |
|---------|----------|-----------|---------|
| `app_net` | Traefik, Frontend, Backend, Worker | No | Core |
| `data_net` | Backend, Worker, DB, Redis | **Yes** | Isolated |
| `module_net` | Traefik, Backend, Modules | No | Module ↔ Backend |

Backend bridges all three. Modules CANNOT reach DB or Redis. In Docker Compose, modules share `module_net` and can probe each other's internal ports; auth-layer controls (`sk_` scoping + module JWT audience/scope enforcement) are the primary isolation boundary.

---

## Part 5: Widget Architecture

**Embedding:** `embed.js` → iframe. CSP `frame-ancestors`. Aggressive caching.

**Security boundary:**
- Primary: server-side `sk_` key (scoped permissions, rate-limited, audited, rotatable).
- Secondary: CSP `frame-ancestors` allowlist + strict Origin/CORS checks in widget BFF.
- Required: edge rate limiting (Traefik/Ingress) before requests hit widget runtime.

**Rate limiting:** edge per-IP + backend per-key (Redis). In-memory per-process limiter remains fallback only.

**Config:** Flow `metadata_json.widget_config` (V1). V2: `widget_configs` table.

---

## Part 6: Sequential Flows ("Flöden")

### 6.1 Core Concept

Ordered chains of AI steps. Each step stateless. Can be inline agent (hidden assistant) or linked specialist.

### 6.2 Flow as Own Aggregate Root

**Critical DDD decision:** Flow is NOT nested inside the Space aggregate. It has its own `FlowRepository` with dedicated CRUD operations.

**Why:** The Space aggregate saves everything inside it recursively (`_set_apps`, `_set_group_chats`). Adding `_set_flows` would mean every Space update recursively saves all flow steps, MCP tool allowlists, and configs — causing massive write latency. The Space only holds lightweight `FlowSparse` references for the UI.

```
Space aggregate:         Flow aggregate:
├── assistants           ├── flow entity
├── apps                 ├── flow_steps (with configs)
├── group_chats          ├── flow_step_mcp_tools
├── services             └── CRUD via FlowRepository
└── flows: FlowSparse[]
    (read-only refs)
```

### 6.3 The Inline Builder UX

```
┌─────────────────────────────────────────────────────────────┐
│  FLOW: "Ärendehantering Byggnadslov"                        │
│                                                             │
│  📝 INPUT FORM (from form_schema):                          │
│  ┌─────────────────────────────────────────────┐            │
│  │ Namn:          [text]    Personnummer: [text]│            │
│  │ Bifogad ritning:        [image upload]      │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  Step 1: "Analysera ritning"                                │
│  ┌─────────────────────────────────────────────┐            │
│  │ Input: flow_input │ Type: image             │            │
│  │ Prompt: [Analysera ritning mot PBL...]      │            │
│  │ Model: [GPT-4o (vision) ▾]                  │            │
│  │ Knowledge: [+ Bygglovsdatabas]  Output: json│            │
│  └─────────────────────────────────────────────┘            │
│                          ↓                                  │
│  Step 2: "Hämta fastighetsdata"                             │
│  ┌─────────────────────────────────────────────┐            │
│  │ Input: http_post → URL + body               │            │
│  │   Body: {"pnr":"{{flow_input.personnummer}}"}│            │
│  │ Prompt: [Jämför analys med data]            │            │
│  │ Model: [Claude Sonnet (EU) ▾]  Output: text │            │
│  └─────────────────────────────────────────────┘            │
│                          ↓                                  │
│  Step 3: "Skriv beslut"                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │ Input: all_previous_steps                   │            │
│  │ Prompt: [Skriv beslut för {{flow_input.namn}}│            │
│  │   baserat på {{step_1.output}}]             │            │
│  │ Model: [GPT-4o ▾]  Knowledge: [Juridisk]   │            │
│  │ Output: pdf  Webhook: archive.sundsvall.se  │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  [+ Steg]                          [Spara] [Publicera]      │
│                                                             │
│  Tabs: [Redigera] [Översikt ◈] [Resultat]                  │
└─────────────────────────────────────────────────────────────┘
```

### 6.4 Hidden Assistants

`hidden: bool = False` on existing `assistants` table. Flow builder creates `hidden=true` assistants. Default list hides them. Hourly cleanup for orphans.

### 6.4.1 Inline Builder UX Refinements

Three patterns that make building flows foolproof for non-technical municipal administrators:

**1. Variable Picker ("Infoga variabel" button):**
Above the prompt textarea in each step, a `{ }` button opens a context-aware dropdown. It reads `form_schema` fields and previous step definitions, presenting a clickable list:
- "Inmatning: Namn" → inserts `{{flow_input.namn}}`
- "Inmatning: Personnummer" → inserts `{{flow_input.pnr}}`
- "Steg 1: Analys (output)" → inserts `{{step_1.output}}`
- "Steg 1: Analys (JSON-fält)" → expands sub-fields if output_type=json

Clicking inserts the tag at cursor position. Users never type `{{...}}` manually.

**2. Auto-Save with Draft Indicator:**
Building a 4-step flow takes time. The frontend auto-saves with debounced PATCH calls (500ms). Flow starts as `published: false`. A subtle indicator in the top-right corner:
- "Sparad ✓" (saved) — green, normal state
- "Sparar..." (saving) — gray, during PATCH
- "Ej sparad" (unsaved) — yellow, if offline/error

First save creates the flow via POST. Subsequent edits PATCH. User never loses work.

**3. Model Capability Warnings:**
When a user connects steps with incompatible model capabilities, the UI shows a yellow warning icon between the steps:
- Step outputs `image` → next step uses model without vision → "⚠️ Modellen i Steg 2 stöder inte bildanalys"
- Step outputs `audio` → next step uses non-Whisper model → "⚠️ Modellen i Steg 2 kan inte transkribera ljud"
- Step outputs `json` → next step expects `image` → "⚠️ Steg 1 producerar JSON, Steg 2 förväntar sig bild"

Model capabilities checked via existing model metadata in the space (vision support, audio support flags).

**4. Reuse Last Test Data ("Återanvänd senaste testdata"):**
On the run tab, a ghost button fetches the user's most recent `FlowRun` for this flow (`GET /flow-runs?flow_id=X&limit=1`), takes `input_payload_json`, and pre-fills the form. Zero backend changes. Eliminates re-typing during iterative debugging.

**5. Semantic Step Names in Visualizer:**
`user_description` field on `flow_steps` is exposed as a bold title input in the step card (e.g., "Hämta personuppgifter", "Skapa PDF"). The graph endpoint uses this as the node label instead of "Step 1". Compliance diagrams read like real business processes: *Formulär → Transkribering → Sammanfattning → Webhook*.

### 6.5 Expanded I/O Types

**Input types** (UI hints for rendering + compatibility):
`text | json | image | audio | document | file | any`

**Output types (V1)** — non-streaming completion path returns text only:
`text | json | pdf | docx`

**Output types (V2)** — deferred until streaming/event handling is integrated:
`image | audio` (generated media requires stream-based file extraction, not available in current non-stream completion path)

Backend handles all **input** types through existing `file_ids` + assistant pipeline. Output types `pdf|docx` use `DocumentGeneratorService` to convert LLM markdown output.

### 6.6 Form Schema

`metadata_json.form_schema` defines dynamic input forms. Field types: `text | number | select | image | audio | document | file`. File fields store `file_id` UUID in `form_data`.

### 6.7 Variable Interpolation

`{{flow_input.namn}}`, `{{step_1.output}}`, `{{step_1.output.field}}`. Simple regex, resolved before LLM call. Applied to: system prompt, input_config.url, input_config.body, output_config.url.

**JSON-safe escaping:** When the interpolation target is inside a JSON string context (detected by surrounding quotes), the resolver JSON-escapes the value (escaping `"`, `\`, newlines) to prevent payload corruption.

### 6.8 Input Sources

`flow_input | previous_step | all_previous_steps | http_get | http_post`

`http_post` input enables GraphQL, SOAP, enterprise REST. Same SSRF protection as `http_get`.

### 6.9 Security Classification

Klass 3 = most sensitive. `output_classification_override` for declassification. Egress validation on webhooks. Runtime check on `all_previous_steps`.

### 6.10 Webhook Header Security

Headers in `input_config` and `output_config` are encrypted at rest in the database using symmetric encryption (Fernet or equivalent). Decrypted only at execution time in the worker. Admin-only access to configuration.

**V1 pragmatic path:** If no encryption library is already in the codebase, store plaintext with explicit documentation: "Webhook credentials are stored plaintext in DB. Admin-only access. Use least-privilege tokens. Encryption-at-rest is a V2 hardening item." Check if Eneo already has a secrets/encryption utility before deciding.

### 6.11 Flow Preview & Compliance Export

**Why:** Municipal transparency requires proving how AI decisions are made. If a flow handles "Biståndshandläggning" (welfare assessment), the municipality must show the decision pipeline.

**Backend:** `GET /api/v1/flows/{id}/graph`

Returns a node/edge structure for visualization:

```json
{
  "nodes": [
    {"id": "input", "label": "Formulär: Byggnadslov", "type": "input",
     "fields": ["namn", "personnummer", "ritning"]},
    {"id": "step_1", "label": "Analysera ritning", "type": "llm",
     "model": "GPT-4o", "input_type": "image", "output_type": "json"},
    {"id": "step_2", "label": "Hämta fastighetsdata", "type": "llm",
     "model": "Claude Sonnet", "input_source": "http_post", "output_type": "text"},
    {"id": "step_3", "label": "Skriv beslut", "type": "llm",
     "model": "GPT-4o", "output_type": "pdf", "has_webhook": true},
    {"id": "output", "label": "Resultat", "type": "output"}
  ],
  "edges": [
    {"source": "input", "target": "step_1"},
    {"source": "step_1", "target": "step_2"},
    {"source": "step_2", "target": "step_3"},
    {"source": "step_3", "target": "output"},
    {"source": "input", "target": "step_3", "style": "dashed",
     "label": "all_previous_steps"}
  ]
}
```

**Frontend: "Översikt" (Overview) tab** — Read-only Svelte Flow diagram:
- Renders nodes as colored cards (input=green, llm=orange, output=yellow, like the Svelte Flow screenshot)
- Edges show data flow with labels (input type, variable references)
- Steps with `all_previous_steps` get dashed edges from all prior nodes
- Steps with `http_get`/`http_post` input show external source indicator
- Steps with webhooks show outbound arrow
- Minimap in bottom-right corner (matching Svelte Flow default)
- **Not for building/editing flows** — purely visualization and compliance

**Stateful visualizer (run results view):**
When the Översikt tab is accessed from a completed `FlowRun` (not the editor), the graph endpoint receives `?run_id=` and returns per-step execution status + timing. Svelte Flow then:
- Colors completed nodes green, failed nodes red, skipped nodes gray
- Shows execution time inside each node ("2.3s", "14.7s")
- Shows token usage badge on each LLM node
- Makes debugging visual — instantly see where a flow failed and how long each step took

**Export buttons on Översikt tab:**
- "Ladda ner flöde (JSON)" — full flow definition for compliance audit
- "Ladda ner diagram (PNG)" — `html-to-image` library captures Svelte Flow DOM → download
- "Ladda ner diagram (SVG)" — vector format for print/reports

**Compliance Audit Report (V1.5):**
"Ladda ner granskningsrapport" generates a PDF (via `DocumentGeneratorService`) containing:
1. Flow name, creator, creation date, last modified
2. Embedded PNG of the Svelte Flow diagram
3. Form schema (what data is collected from users)
4. Each step: system prompt, model name, classification level, I/O types
5. If from a specific run: execution times, token counts, pass/fail status per step
This is what municipal auditors need when evaluating AI decision-making.

---

## Part 7–9: Versioning, Compose, SDK

Separate image per module with independent tags:
- Core services use `${ENEO_VERSION}`.
- Module services use `${MODULE_<ID>_VERSION}`.

Compatibility gate is contract-first:
- Hard gate: `module_api_contract` match (e.g., `core-api-v1`).
- Advisory metadata: `core_compat_min`/`core_compat_max`.
- Optional audited admin override for exceptional municipal operations.

Deployment posture:
- Docker Compose is primary and fully supported for municipal environments (1–20+ modules).
- Kubernetes/Helm is reference-only for organizations that already run K8s and need advanced HA/network-policy controls.

SDK: `intric-js` directly.

---

## Part 10: Monorepo Structure

```
eneo/
├── frontend/apps/web/                 # Flöden inline builder + Översikt tab
├── frontend/packages/intric-js/       # ONE typed API client
├── backend/src/intric/
│   ├── flows/                         # Flow aggregate root (own repo)
│   │   ├── domain/entities/
│   │   ├── application/               # FlowService, FlowRunService, Runner, VariableResolver
│   │   ├── infrastructure/
│   │   │   └── flow_repo.py           # Dedicated Flow repository (NOT in space_repo)
│   │   └── presentation/              # Routers, models, assemblers, worker, graph endpoint
│   ├── assistants/                    # Add hidden flag, execute_once, prompt_override
│   ├── module_registry/
│   ├── authentication/                # API Key v2 (flows in ResourcePermissions)
│   └── audit/
├── modules/
│   ├── speech-to-text/ │ widget/ │ shared/auth/ │ _template/
├── docker-compose.core.yml
├── docker-compose.modules.yml
└── VERSION
```

---

## Part 11: Decision Matrix

| # | Decision | Choice | Why |
|---|----------|--------|-----|
| 1 | Module → Backend | **BFF** | Core never changes |
| 2 | Networks | **3-tier** | Modules can't reach DB |
| 3 | Module user auth | **Auth Broker + Ticket Handoff** | Module isolation, no shared cookies, no IdP sprawl |
| 3b | Service auth | **`sk_` API Key v2** | Already built |
| 4 | Flow execution | **`execute_once()`** | Preserves RAG |
| 5 | Flow DDD boundary | **Own aggregate root** | Avoids Space write latency |
| 6 | Step definition | **Inline builder + hidden assistants** | One-page UX |
| 7 | Flow input | **Form schema in metadata_json** | Non-technical users |
| 8 | Parameterization | **`{{variable}}` with JSON-safe escaping** | Dynamic prompts + HTTP bodies |
| 9 | I/O types | **Expanded (image, audio, document, file)** | Multimodal pipelines |
| 10 | HTTP inputs | **GET + POST** | Enterprise APIs |
| 11 | Worker transactions | **Manual short-lived sessions** | No connection pool exhaustion |
| 12 | Compliance | **Graph endpoint + Svelte Flow + JSON export** | Municipal transparency |
| 13 | Webhook secrets | **Encrypted at rest (or documented plaintext V1)** | Enterprise security |
| 14 | Entity cloning | **Check codebase: model_copy OR deepcopy** | Depends on MCP server type |
| 15 | Module session cookie | **`__Host-` host-only cookie** | Minimize blast radius on module compromise |
| 16 | OIDC state transport | **Redis-backed `state_id`** | Avoid IdP URL/state size issues |
| 17 | Module JWT refresh | **Dedicated `/api/v1/auth/module-refresh`** | 15-minute TTL without redirect UX churn |
| 18 | Module versioning | **Per-module `${MODULE_<ID>_VERSION}`** | Independent update path |
| 19 | Compatibility gate | **`module_api_contract` primary** | Version-agnostic compatibility |
| 20 | Flow resume determinism | **Execution-hash comparison** | Predictable reruns after logic changes |
| 21 | Deployment target | **Compose-first (1–20+)** | Matches municipal operational reality |

### What NOT to Build

- Custom widget auth / `widgets/` backend / separate SDK
- Full template engine (simple regex with JSON escaping)
- Visual DAG editor for building flows (Svelte Flow is read-only preview only)
- Widget config table V1 / auto-discovery / container orchestration UI
- Flow import from Svelte Flow diagram (export/preview only)

---

## Part 12: Codebase Alignment (Codex Review — P0/P1 Fixes)

These findings are from actual codebase inspection (file:line references verified). All P0 items must be fixed before PR 1 merges.

### P0 Fixes (blocks first commit)

**P0-1: Worker transaction wraps full task**
`worker.py:445` — `@worker.task` wraps the full task body in `session.begin()`. A flow step taking 60s for LLM locks a DB connection the entire time.
**Fix:** Create a `long_running_task` pattern (or flow-specific task decorator) that does NOT auto-open a session. The runner uses `session_factory` for short-lived persist calls only.

**P0-2: `timeout=1800` not in decorator signature**
`worker.py:431` — decorator only accepts `with_user` and `channel_type`, not `timeout`.
**Fix:** Use worker-level timeout config (ARQ `job_timeout` setting) or extend the decorator. Document which approach is used.

**P0-3: It's `prompt`, not `instructions`**
`assistant.py:281` uses `prompt` text. `completion_service.get_response` takes `prompt` parameter (`completion_service.py:198`), not `instructions`.
**Fix:** Rename all references: `instructions_override` → `prompt_override` throughout all docs and code.

**P0-4: Non-streaming misses `tool_calls_metadata`**
`tenant_model_adapter.py:415` — non-stream path executes tools but doesn't populate `Completion.tool_calls_metadata`.
**Fix:** Explicitly populate `tool_calls_metadata` in the non-stream branch of `tenant_model_adapter.py`. Add to PR 2.

**P0-5: Security level path is nested**
Security level is at `completion_model.security_classification.security_level`, NOT `completion_model.security_level`.
**Fix:** All classification validation code must use the nested path: `step.assistant.completion_model.security_classification.security_level`.

### P1 Fixes (blocks production readiness)

**P1-6: Step 1 defaulting to `previous_step` = empty input**
**Fix:** API validation: if `step_order == 1`, reject `previous_step` and `all_previous_steps`. Default step 1 to `flow_input`.

**P1-7: Hidden assistant filter breaks execution lookups**
`space_repo.py:832` and `get_space_by_assistant` (`space_repo.py:1329`) load assistants — if hidden filter is global, flow execution can't find step assistants.
**Fix:** Apply `WHERE hidden = false` ONLY in list/UI endpoints. By-ID lookups and `get_space_by_assistant` must include hidden records.

**P1-8: SessionProxy breaks in long-running tasks**
Container's SessionProxy pattern (`container.py:1367`) assumes request-scoped sessions. Long-running worker tasks need explicit session management.
**Fix:** Runner must use explicit `session_factory` for all DB operations, never rely on ambient SessionProxy.

**P1-9: File GC enforces owner check**
`FileService.delete_file` (`file_service.py:93`) checks file ownership. Cleanup jobs running as system can't delete user-uploaded files.
**Fix:** Add `delete_file_system(file_id, tenant_id)` — tenant-scoped system GC path that bypasses user ownership check but requires tenant authorization.

---

## Part 13: Risk Assessment

| Risk | Mitigation |
|------|------------|
| Worker holds long DB transaction | P0-1: `long_running_task` pattern, no auto-session |
| `prompt` vs `instructions` mismatch | P0-3: `prompt_override` throughout |
| Security level wrong attribute path | P0-5: Use nested `.security_classification.security_level` |
| Hidden assistants invisible to runner | P1-7: Filter only in list/UI, not by-ID lookups |
| Step 1 empty input | P1-6: Validate step_order=1 cannot use `previous_step` |
| File GC fails on ownership | P1-9: System-level GC path with tenant scope |
| Space write latency from flows | Flow is own aggregate root |
| Module compromise escalation | Module-scoped JWT with constrained aud + permissions |
| Ticket replay attack | Atomic Redis consume-once, 10–30s TTL |
| Open redirect via module callback | Exact-match allowlist from module_clients table |
| Variable injection | JSON-safe escaping; variables only in prompt template |
| Webhook header leak | Encrypted at rest or documented plaintext + admin-only |
| Entity copy crash | `_safe_copy()` checks Pydantic vs domain entity |
| Output type image/audio | V1: text\|json\|pdf\|docx only. V2: generated media |

---

## Part 14: Known Limitations & V2 Boundaries

| Limitation | V1 | V2 Trigger |
|------------|----|-----------| 
| Widget config in flow metadata | Duplicate flow | 2 widgets same flow → `widget_configs` table |
| BFF rate limiter per-replica | Edge + backend limits required | Add distributed global limiter when needed |
| Webhook headers plaintext | Admin-only + docs | Enterprise audit → Fernet encryption |
| Module east-west on shared `module_net` | Accepted Compose trade-off; auth-layer isolation is primary | Strict east-west policies required |
| No partial resume optimization | Upstream logic change forces full rerun from step 1 | Add earliest-changed-step partial rerun |
| Independent module sessions | Module JWT revocation via TTL+refresh checks (`auth_time`, `logout_after`) | Full cross-domain single logout orchestration |
| Simple variable interpolation | `{{var}}` regex | Need conditionals → template engine |
| Svelte Flow read-only | Preview + export only | Need visual editor → evaluate builder mode |
| No flow import | JSON export only | Need flow sharing → import endpoint |
| No dry run / simulation | Run to test | V2: "Testa flöde" step-by-step preview |
| No compliance audit PDF | JSON + PNG export | V1.5: "Granskningsrapport" PDF |
| Output type image/audio | text\|json\|pdf\|docx | Streaming/event integration |
| Worker timeout via config | ARQ `job_timeout` | Extend decorator signature |

---

## Part 15: Documentation Updates

| Document | Update |
|----------|--------|
| `README.md` | Modules, flow builder, Översikt tab |
| `docs/deployment.md` | Compose overlay, DNS, env |
| `docs/flows.md` (new) | Form schema, variables, I/O types, compliance export, graph API |
| `docs/security.md` | Classification, SSRF, webhook header handling |
| `modules/README.md` (new) | Module dev guide, contract, hooks |
| `CHANGELOG.md` | Flows, modules, widget, inline builder, compliance export |
