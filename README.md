# Eneo Modules & Flows — Architecture Reference

Reference architecture for two new Eneo capabilities: **Flows** (sequential AI pipelines) and **Modules** (self-contained add-on services). This repo contains the design documents, Docker Compose configurations, and a worked example — a Speech-to-Text pipeline for municipal social services.

---

## Lasanvisning

Det har repot innehaller arkitekturdokumenten for Eneos nya funktioner: Floden (sekventiella AI-pipelines) och Moduler (sjalvstandiga tillaggstjanster med BFF-monster). Malsattningen ar att ge teamet en gemensam bild av hur vi bygger sakra, granskningsbara AI-arbetsfloden for offentlig sektor. Borja med scenariot "Tal-till-text" langre ner — det gor resten av dokumentet konkret.

---

## What This Repo Contains

| File | For whom | What it covers |
|------|----------|----------------|
| `eneo-definitive-architecture-v3.2.md` | Architects, tech leads | The full architecture: module system, network model, auth, widget, flows. Source of truth. |
| `eneo-unified-implementation-plan-v3.2.md` | Developers, architects | Implementation spec: data models, API endpoints, worker tasks, frontend. 7-PR delivery plan. |
| `flowdesign-v3.2(2).md` | Developers | Flow execution internals: variable resolution, classification validation, runner, SSRF. |
| `docker-compose.core.yml` | DevOps, architects | Core infrastructure: Traefik, Frontend, Backend, Worker, PostgreSQL, Redis. Three-network isolation. |
| `docker-compose.modules.yml` | DevOps, developers | Module overlay: how to add Speech-to-Text, Widget, or any new module. Zero changes to core. |
| `taltilltextflow.excalidraw.png` | Everyone | Visual diagram of the Speech-to-Text flow pipeline. |
| `taltilltextflow.excalidraw` | Designers, architects | Editable diagram source (open with [excalidraw.com](https://excalidraw.com)). |

---

## The Big Picture

Here is the overall model. A **Flow** is a sequential AI pipeline. A **Module** is a standalone UI service (following the BFF — Backend For Frontend — pattern) that triggers flows via the backend API.

```mermaid
graph LR
    subgraph Module["Module (BFF)"]
        UI["Specialized UI"]
        BFF["/api/* proxy"]
    end

    subgraph Eneo["Eneo Core"]
        API["Backend API"]
        Engine["Flow Engine"]
        Worker["Worker\n(ARQ task queue)"]
    end

    subgraph Flow["Flow Pipeline"]
        S1["Step 1\n(own model, tools, KB)"]
        S2["Step 2\n(own model, tools, KB)"]
        S3["Step 3\n(own model, tools, KB)"]
    end

    UI --> BFF
    BFF -->|"internal network"| API
    API --> Engine
    Engine --> Worker
    Worker --> S1 --> S2 --> S3
```

The module handles the user experience. The backend handles execution, security, and audit. The flow defines what each AI step does. These three concerns are independent — you can change the module UI without touching the flow, or modify a step's prompt without redeploying the module.

---

## The Scenario: Speech-to-Text (Tal-till-text)

A social worker in Sundsvall finishes a client meeting about welfare support. They recorded the audio. Today, turning that recording into a formal journal entry takes 45 minutes of manual work: transcribe, look up patient data, cross-reference guidelines, write the entry, post it to the journal system.

With an Eneo flow, it takes one button press. The user visits `taltilltext.sundsvall.se` (the STT module), uploads the audio, fills in case number and handler name, and hits "Kor flodet." Three AI steps execute in sequence:

![Speech-to-Text Flow Architecture](./taltilltextflow.excalidraw.png)

*Editable source: [taltilltextflow.excalidraw](./taltilltextflow.excalidraw)*

### Step-by-Step Data Flow

Each step is an independent AI agent with its own model, prompt, tools, and security classification. Data flows strictly forward — Step 1 can never see Step 3's output.

```
╔══════════════════════════════════════════════════════════════════════╗
║  STEP 1: Transkribera                                               ║
║  Model:  Whisper (Azure OpenAI)                                     ║
║  Klass:  1 (public — audio transcription is not sensitive)          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  INPUT:   flow_input → audio file (meeting.mp3)                      ║
║  PROMPT:  "Transkribera lydelsen ordagrant pa svenska."              ║
║  MCP:     none                                                       ║
║  KNOWLEDGE: none                                                     ║
║                                                                      ║
║  OUTPUT (text):                                                      ║
║  "Anna: Hej, jag vill prata om ditt stodbehov framat.               ║
║   Klient: Ja, jag har haft svart med vardagliga sysslor             ║
║   sedan operationen i december..."                                   ║
║                                                                      ║
║  ── This raw transcript flows forward to Step 2 ──                   ║
║     Classification: K1 → K2 (escalation allowed)                     ║
╚══════════════════════════════════════════════════════════════════════╝
                              │
                              ▼
╔══════════════════════════════════════════════════════════════════════╗
║  STEP 2: Sammanfatta och berika                                      ║
║  Model:  GPT-4o (Azure, EU-hosted)                                  ║
║  Klass:  2 (internal — now we're adding personal identifiers)       ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  INPUT:   previous_step → the transcript from Step 1                 ║
║                                                                      ║
║  PROMPT:  "Du ar en journalassistent. Sammanfatta motet i            ║
║            strukturerat JSON-format. Hamta patientdata via           ║
║            MCP-verktyget lookup_patient med arendenummer              ║
║            {{flow_input.arendenr}}. Inkludera falt:                  ║
║            patient_id, summary, key_decisions, next_steps."          ║
║                                                                      ║
║  MCP TOOLS (restricted allowlist):                                   ║
║    ┌──────────────────┐    ┌──────────────────────┐                 ║
║    │ lookup_patient   │    │ fetch_case            │                 ║
║    │ → HSA-katalog    │    │ → Arendesystem API    │                 ║
║    │   (staff + client│    │   (case history)      │                 ║
║    │    registry)     │    │                        │                 ║
║    └──────────────────┘    └──────────────────────┘                 ║
║                                                                      ║
║  KNOWLEDGE BASE: "Journalriktlinjer"                                ║
║    (RAG retrieves relevant sections about how to                     ║
║     structure social service journal entries per                     ║
║     Socialtjanstlagen chapter 11 §5)                                 ║
║                                                                      ║
║  OUTPUT (json):                                                      ║
║  {                                                                   ║
║    "patient_id": "P-198705-2341",                                   ║
║    "case_number": "2026-BIS-0847",                                  ║
║    "handler": "Anna Lindstrom",                                     ║
║    "summary": "Klienten beskriver fortsatta svarigheter...",        ║
║    "key_decisions": ["Utoka hemtjanst till 4x/vecka"],              ║
║    "next_steps": ["Uppfoljning 2026-03-15"],                        ║
║    "meeting_date": "2026-02-20"                                     ║
║  }                                                                   ║
║                                                                      ║
║  ── Structured JSON flows forward to Step 3 ──                       ║
║     Classification: K2 → K3 (escalation allowed)                     ║
╚══════════════════════════════════════════════════════════════════════╝
                              │
                              ▼
╔══════════════════════════════════════════════════════════════════════╗
║  STEP 3: Skapa journalanteckning                                    ║
║  Model:  Llama 3.1 70B (on-premise, Sundsvall datacenter)          ║
║  Klass:  3 (secret — full journal with personal details)            ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  INPUT:   all_previous_steps                                         ║
║           ┌───────────────────────────────────────────┐             ║
║           │ <step_1>                                   │             ║
║           │   Full transcript text...                  │             ║
║           │ </step_1>                                  │             ║
║           │ <step_2>                                   │             ║
║           │   {"patient_id":"P-198705-2341",...}       │             ║
║           │ </step_2>                                  │             ║
║           └───────────────────────────────────────────┘             ║
║                                                                      ║
║  PROMPT:  "Fyll i journalmallen. Anvand                              ║
║            {{step_2.output.summary}} som sammanfattning.             ║
║            Handlaggare: {{flow_input.handlaggare}}.                  ║
║            Patient: {{step_2.output.patient_id}}.                   ║
║            Inkludera ordagrant citat fran transkriberingen            ║
║            dar klienten uttrycker sina behov."                        ║
║                                                                      ║
║  MCP TOOLS (restricted allowlist):                                   ║
║    ┌──────────────────┐                                              ║
║    │ fill_template    │  ← fills a DOCX template with structured    ║
║    │ → Mallsystem     │     data from the JSON + transcript         ║
║    └──────────────────┘                                              ║
║                                                                      ║
║  KNOWLEDGE BASE: "Journalmallar"                                    ║
║    (RAG retrieves the correct template format for                    ║
║     "Bistandshandlaggning — Motesanteckning")                        ║
║                                                                      ║
║  OUTPUT (docx):                                                      ║
║    → journalanteckning_2026-BIS-0847.docx                           ║
║                                                                      ║
║  OUTPUT MODE: http_post                                              ║
║    → POST https://journal.sundsvall.se/api/v1/entries               ║
║      Body: {"case": "{{step_2.output.case_number}}",                ║
║             "handler": "{{flow_input.handlaggare}}",                 ║
║             "document_id": "{{step_3.output.file_id}}"}             ║
║      Headers: {"Authorization": "Bearer sk_journal_xxx"}            ║
║                                                                      ║
║  K3 data CANNOT flow backwards to K1 or K2 models                   ║
║  K3 output CANNOT be sent to external webhooks                       ║
║     (journal.sundsvall.se is internal → allowed)                    ║
╚══════════════════════════════════════════════════════════════════════╝
```

After Step 3 completes, the generated DOCX is stored in Eneo (downloadable from the results tab) and simultaneously POSTed to the journal system. The user sees all three steps with green status indicators, execution times per step, and total token usage.

---

## Core Concepts

### Eneo Flows (Floden)

A flow is a sequential AI pipeline: **Start → Step 1 → Step 2 → Step 3 → End**. Each step is a full AI agent with its own prompt, model, knowledge base, and MCP (Model Context Protocol) tools. Steps execute one after another — no loops, no branching, no conditional logic.

This is deliberate. Municipal processes are inherently linear (receive → analyze → decide → archive). A strict forward-only pipeline means every execution is fully auditable — you can export the exact sequence of what happened, what each model saw, and what it produced. Non-technical administrators can build and modify flows without understanding graph theory.

**What each step can do:**

- Use a different AI model (Whisper for audio, GPT-4o for reasoning, Llama for on-premise)
- Access its own knowledge bases via RAG (Retrieval-Augmented Generation — the existing Eneo embedding pipeline is fully preserved)
- Call external APIs via MCP tools (each step has an independent tool allowlist)
- Read data from the original form input, the previous step, all previous steps, or an HTTP endpoint
- Output text, JSON, PDF, or DOCX

**Variable interpolation** connects the steps. A prompt can reference `{{flow_input.field_name}}` for form data or `{{step_2.output.summary}}` for a JSON field from a previous step's output. Variables are JSON-safe escaped to prevent injection in HTTP request bodies.

Deep dive: [eneo-unified-implementation-plan-v3.2.md](./eneo-unified-implementation-plan-v3.2.md)

---

### Eneo Modules — The Architecture and Why It Matters

A module is a self-contained service that adds a specialized capability to Eneo. The Speech-to-Text module is the first example. The architecture is designed so that any team can build and ship a module without touching Eneo's core codebase.

**The problem modules solve:**

Without modules, every new capability (STT, document scanning, form builder, etc.) would need to be built directly into the Eneo backend and frontend. That means more code in the monolith, more coupling between features, longer CI pipelines, higher risk of regressions, and every team waiting on the same deploy cycle. Over time, this creates compounding technical debt — each feature makes the next one harder to ship.

**The BFF (Backend For Frontend) pattern:**

```mermaid
graph TB
    Browser["Browser\ntaltilltext.sundsvall.se"]

    subgraph Module["STT Module Container"]
        ModUI["Module UI\n(audio recorder, progress view)"]
        ModAPI["Module /api/* routes\n(BFF proxy layer)"]
    end

    subgraph Core["Eneo Core"]
        Backend["Backend API\nhttp://eneo_backend:8000"]
        DB["PostgreSQL + Redis"]
    end

    Browser --> ModUI
    ModUI --> ModAPI
    ModAPI -->|"module_net (internal)"| Backend
    Backend -.->|"data_net (isolated)"| DB
    ModAPI -.->|"BLOCKED — no network path"| DB

    style DB fill:#f9f,stroke:#333
    linkStyle 4 stroke:red,stroke-dasharray:5
```

Each module is a standalone server that:

1. **Serves its own UI** at its own subdomain (e.g., `taltilltext.sundsvall.se`)
2. **Has its own `/api/*` routes** that proxy requests to the Eneo backend internally
3. **Runs in its own container** on the module network
4. **Cannot reach the database or Redis** — it can only talk to the backend API

The browser never communicates directly with the Eneo backend when on a module's domain. All requests go through the module's BFF layer, which adds domain-specific logic before forwarding to the backend.

**Why this reduces technical debt and increases flexibility:**

- **Isolation.** A bug in the STT module cannot crash the core platform. A compromised module container cannot read the database.
- **Independent deployment.** The STT team ships on their own schedule. No coordination with the core team unless the backend API changes.
- **Opt-in activation.** Modules are enabled via Docker profiles (`--profile speech-to-text`). Municipalities that don't need STT don't run the container — zero resource cost, zero attack surface.
- **Clean contracts.** The backend API is the only interface between core and modules. This forces good API design and prevents the internal coupling that makes monoliths hard to change over time.
- **Zero core changes.** Adding a new module requires 8 steps, none of which touch `docker-compose.core.yml` or any backend code.

**Adding a new module in practice:**

1. Copy `modules/_template` to `modules/my-module`
2. Edit `module.json` (id, name, port)
3. Add service entry to `docker-compose.modules.yml`
4. Set domain in `.env`
5. Create an `sk_` API key in the admin UI (scoped, IP-restricted, rate-limited)
6. Add key to `env_modules.env`
7. Add to CI build matrix
8. Done — core remains untouched

**Authentication for modules:**

- **User-facing modules** (like STT): use the SSO (Single Sign-On) cookie from `.sundsvall.se`. The user is already logged into Eneo — the module inherits the session.
- **Service-level calls**: use `sk_` API keys (API Key v2) — scoped to specific spaces, IP-restricted to the Docker module network, rate-limited, audited per request, and rotatable with a grace period for zero-downtime rotation.
- **Widget module** (for embedding on external sites): uses only `sk_` keys since anonymous visitors don't have Eneo accounts.

Deep dive: [eneo-definitive-architecture-v3.2.md](./eneo-definitive-architecture-v3.2.md)

---

### Security Classification (Klass 1 / 2 / 3)

Swedish municipal data is classified into three levels:

| Level | Label | What it means | Model hosting |
|-------|-------|---------------|---------------|
| K1 | Public | Non-sensitive data | Any (including cloud) |
| K2 | Internal | Personal identifiers, internal deliberations | EU-hosted only |
| K3 | Secret | Full personal records, medical data, social service journals | On-premise only |

**Monotonic enforcement:** Data can only flow to models of equal or higher classification. K1 → K2 → K3 is allowed (escalation). K3 → K1 is blocked. This is validated when the flow is saved, not at runtime — if you try to create a flow where a K3 step feeds into a K1 step, the API rejects it immediately.

```mermaid
graph LR
    K1["K1\nWhisper (cloud)"] -->|"allowed"| K2["K2\nGPT-4o (EU)"]
    K2 -->|"allowed"| K3["K3\nLlama (on-premise)"]
    K3 -.->|"BLOCKED at save time"| K1
    K3 -.->|"BLOCKED"| K2

    linkStyle 2 stroke:red,stroke-dasharray:5
    linkStyle 3 stroke:red,stroke-dasharray:5
```

In the STT example: Whisper (K1) transcribes audio (not sensitive), GPT-4o (K2, EU-hosted) adds personal identifiers from the case system, Llama (K3, on-premise) produces the full journal entry with all personal data. Each step handles data at or above the classification of what it receives.

**MCP tool restriction per step:** Each step has a tool policy — either `inherit` (use all tools from the underlying assistant) or `restricted` (explicit allowlist only). Step 2 in the STT flow can call `lookup_patient` and `fetch_case`, but Step 3 cannot — even if the underlying assistant has those tools. Tools auto-execute within the pipeline; the allowed set is pre-configured by the flow administrator at design time.

---

## Network Architecture

The platform runs on three Docker networks with strict isolation:

```
                    INTERNET
                       │
              ┌────────┴────────┐
              │     TRAEFIK     │
              │  (reverse proxy)│
              └───┬─────────┬───┘
                  │         │
      ┌───────── │ ─────────│──────────────────────────────────┐
      │  app_net │         │                                    │
      │          ▼         │                                    │
      │    ┌──────────┐    │    ┌──────────┐   ┌──────────┐   │
      │    │ Frontend │    │    │  Worker  │   │ Backend  │   │
      │    │ (Svelte) │    │    │  (ARQ)   │   │ (FastAPI)│   │
      │    └──────────┘    │    └────┬─────┘   └──┬──┬────┘   │
      │                    │         │            │  │         │
      └────────────────────│─────────│────────────│──│─────────┘
                           │         │            │  │
      ┌────────────────────│─────────│────────────│──│─────────┐
      │  data_net          │         │            │  │         │
      │  (internal: true   │         ▼            ▼  │         │
      │   — no egress)     │   ┌──────────┐  ┌──────────┐     │
      │                    │   │ PostgreSQL│  │  Redis   │     │
      │                    │   │ + pgvector│  │          │     │
      │                    │   └──────────┘  └──────────┘     │
      └────────────────────│───────────────────────────────────┘
                           │                     │
      ┌────────────────────│─────────────────────│─────────────┐
      │  module_net        │                     │             │
      │                    ▼                     ▼             │
      │    ┌───────────────────┐  ┌───────────────────┐       │
      │    │  Speech-to-Text  │  │     Widget Host    │       │
      │    │  (BFF module)    │  │   (BFF module)     │       │
      │    │                  │  │                     │       │
      │    │  → backend:8000  │  │  → backend:8000    │       │
      │    └───────────────────┘  └───────────────────┘       │
      │                                                        │
      │    Modules call backend internally.                    │
      │    Modules CANNOT reach PostgreSQL or Redis.           │
      └────────────────────────────────────────────────────────┘
```

**app_net** — Frontend, Backend, Worker, and Traefik. Outbound internet access allowed (for LLM APIs, OIDC providers, external webhooks).

**data_net** — PostgreSQL (with pgvector) and Redis only. Marked `internal: true` in Docker — no outbound internet, no routing to other networks. Only Backend and Worker are connected here. Modules, Frontend, and Traefik cannot reach the database.

**module_net** — Modules and Traefik. Each module can reach the Backend at `http://eneo_backend:8000`. Modules are isolated from each other and from the data layer.

The Backend is the only service that bridges all three networks — the single gateway between the outside world, the data layer, and the modules. Even if a module container is compromised, the attacker has no path to the database.

Reference: [docker-compose.core.yml](./docker-compose.core.yml) and [docker-compose.modules.yml](./docker-compose.modules.yml)

---

## How the STT Module Fits In

The Speech-to-Text module is not the flow itself. It is a specialized UI wrapper around a standard Eneo flow:

```
Browser (taltilltext.sundsvall.se)
   │
   ▼
STT Module (BFF)
   │── Serves audio recorder widget, drag-and-drop upload
   │── Custom progress visualization per step
   │── Domain-specific UX for social workers
   │
   └── /api/* routes (server-side)
          │
          ▼
       http://eneo_backend:8000 (internal, via module_net)
          │
          ├── POST /api/v1/flows/{id}/run    (trigger the flow)
          ├── GET  /api/v1/flow-runs/{id}    (poll for results)
          └── WebSocket for live step progress
```

The module provides the user experience. The backend provides the flow engine, security enforcement, and audit trail. If you wanted to build a different UI for the same flow — a mobile app, a different web design, an API-only integration — you would build a different module or call the backend API directly. The flow definition stays the same.

This separation is the core value of the module architecture: **the flow engine is generic, the user experience is specialized.**

---

## Further Reading

- **[eneo-definitive-architecture-v3.2.md](./eneo-definitive-architecture-v3.2.md)** — The source of truth. Full module system, auth model, network architecture, widget embedding, flow engine, and all 14 key architectural decisions.

- **[eneo-unified-implementation-plan-v3.2.md](./eneo-unified-implementation-plan-v3.2.md)** — Implementation spec. Data models, API endpoints, worker tasks, frontend components, and the 7-PR delivery sequence.

- **[flowdesign-v3.2(2).md](./flowdesign-v3.2(2).md)** — Flow execution internals. Variable resolution, classification validation, SequentialFlowRunner, SSRF protection, and codebase alignment fixes.

- **[docker-compose.core.yml](./docker-compose.core.yml)** — Core infrastructure. Defines the three-network model and all core services. This file never changes when adding modules.

- **[docker-compose.modules.yml](./docker-compose.modules.yml)** — Module overlay. STT and Widget module configs, plus step-by-step instructions for adding new modules.
