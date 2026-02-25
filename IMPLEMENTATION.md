# Eneo Unified Implementation Plan v3.9 — Final

**Date:** 2026-02-25
**Status:** Final — v3.9 aligns broker handoff, refresh lifecycle, deterministic resume semantics, and compose-first operations
**Scope:** Flöden + Modules + Widget + Health Registry + Auth Broker — single release, 7 PRs

---

## Design Principles

1. **No mutation** of `assistant.mcp_servers` in-place during execution
2. **No user-visible sessions** polluted by internal flow-step runs
3. **Explicit MCP allowlist semantics** (`mcp_policy` enum)
4. **Security classification with explicit declassification** via `output_classification_override`
5. **Data retention from day one** with file garbage collection
6. **JSON payload fields** for extensibility (text + file_ids + form_data)
7. **Preserve full RAG pipeline** in `execute_once` (do NOT bypass `assistant.ask()`)
8. **Reuse API Key v2** for all service-level auth
9. **Incremental step result persistence** — each step saved immediately
10. **Configurable SSRF protection** — strict by default, allowlist for enterprise intranets
11. **Idempotent step execution** — ARQ retries skip completed steps
12. **Manual short-lived DB sessions** — no auto-transaction during LLM calls
13. **Inline builder with hidden assistants** — one-page flow creation
14. **Variable interpolation with JSON-safe escaping**
15. **Flow is its own aggregate root** — not nested in Space aggregate

---

## 0. Codebase Alignment (Codex Review — P0/P1 Fixes Required)

From actual codebase inspection with file:line references. P0 items MUST be resolved before PR 1 merges.

### P0 (blocks first commit)

| # | Finding | File Reference | Fix |
|---|---------|---------------|-----|
| P0-1 | `@worker.task` wraps full body in `session.begin()` | `worker.py:445` | Create `long_running_task` or separate session factory |
| P0-2 | `timeout=1800` not in decorator signature | `worker.py:431` (only `with_user`, `channel_type`) | ARQ `job_timeout=1800` in worker config |
| P0-3 | It's `prompt`, not `instructions` | `assistant.py:281`, `completion_service.py:198` | `prompt_override` throughout all code/docs |
| P0-4 | Non-streaming misses `tool_calls_metadata` | `tenant_model_adapter.py:415` | Populate in non-stream branch |
| P0-5 | Security level is nested `.security_classification.security_level` | `completion_model.py:31` | Fix all validation paths |

### P1 (blocks production)

| # | Finding | File Reference | Fix |
|---|---------|---------------|-----|
| P1-6 | Step 1 defaults to `previous_step` = empty | flow_steps schema | Validate: step_order=1 → reject `previous_step` |
| P1-7 | Hidden filter breaks `get_space_by_assistant` | `space_repo.py:832,1329` | Filter ONLY in list/UI, NOT by-ID |
| P1-8 | SessionProxy breaks in long-running | `container.py:1367` | Runner uses own `session_factory` |
| P1-9 | `delete_file` enforces owner check | `file_service.py:93` | Add system-level GC path |

### P2 (defer to V2)

| # | Finding | Fix |
|---|---------|-----|
| P2-10 | `output_type=image\|audio` not feasible non-stream | V1: text\|json\|pdf\|docx only |

---

## 1. Data Model

### 1.1 Existing Table Change: Assistants

Add `hidden: bool = False` column. Migration: `ALTER TABLE assistants ADD COLUMN hidden BOOLEAN DEFAULT FALSE NOT NULL;`

Default list filter: `WHERE hidden = false`. Flow builder queries `WHERE hidden = true AND space_id = ?`.

### 1.2 Flow Tables

```
flows
├── id, created_at, updated_at              (BasePublic)
├── name: str
├── description: str | None
├── published: bool                         (default False)
├── insight_enabled: bool                   (default False)
├── data_retention_days: int | None
├── metadata_json: JSONB | None
│     {
│       "form_schema": [
│         {"id": "namn", "label": "Namn", "type": "text", "required": true},
│         {"id": "ritning", "label": "Ritning", "type": "image", "required": true}
│       ],
│       "widget_config": {"widget_theme": "blue", "widget_welcome_text": "Hej!"}
│     }
│     Form field types: text | number | select | image | audio | document | file
├── space_id → spaces.id                    (CASCADE)
├── user_id → users.id                      (CASCADE)
├── tenant_id → tenants.id                  (CASCADE)
├── icon_id → icons.id                      (SET NULL)
└── steps: relationship → flow_steps        (viewonly, order_by step_order)

flow_steps
├── id, created_at, updated_at              (BasePublic)
├── flow_id → flows.id                      (CASCADE)
├── assistant_id → assistants.id            (CASCADE)
│     Hidden (inline builder) or pre-existing (linked)
├── step_order: int
├── user_description: str | None
├── input_source: str                       (default "previous_step")
│     flow_input | previous_step | all_previous_steps | http_get | http_post
│     ⚠️ P1-6: If step_order=1, reject previous_step/all_previous_steps at API time.
│     Step 1 should default to flow_input.
├── input_type: str                         (default "any")
│     text | json | image | audio | document | file | any
├── output_mode: str                        (default "pass_through")
│     pass_through | http_post
├── output_type: str                        (default "text")
│     V1: text | json | pdf | docx
│     V2: image | audio (deferred — non-stream completion returns text only,
│         generated media requires stream/event handling per P2-10)
├── output_classification_override: int | None
│     Use `is not None` check (Python: `0 or x` evaluates to x)
├── mcp_policy: str                         (default "inherit")
│     inherit | restricted
├── input_config: JSONB | None
│     http_get:  {"url": "...", "headers": {...}, "timeout_seconds": 10}
│     http_post: {"url": "...", "headers": {...}, "body": "{...}", "timeout_seconds": 10}
│     URL and body support {{variable}} interpolation
│     ⚠️ HEADERS: Encrypted at rest if encryption utility exists in codebase.
│       Otherwise plaintext with documented risk (admin-only, least-privilege tokens).
│       Check for existing encryption before deciding. V2: mandate Fernet.
├── output_config: JSONB | None
│     Same encryption policy as input_config headers
└── UNIQUE(flow_id, step_order)

flow_step_mcp_tools                         (membership-based allowlist)
├── flow_step_id → flow_steps.id (CASCADE)  PK
└── mcp_server_tool_id → mcp_server_tools.id (CASCADE) PK

flow_runs
├── id, created_at, updated_at              (BasePublic)
├── flow_id → flows.id                      (CASCADE)
├── user_id → users.id                      (SET NULL)
├── tenant_id → tenants.id                  (CASCADE)
├── input_payload_json: JSONB | None
│     {"text": "...", "file_ids": [...], "form_data": {"namn": "Anna", ...}}
├── output_payload_json: JSONB | None
├── error_message: str | None
├── job_id → jobs.id                        (SET NULL)
│     Status derived from job.status (no separate column)
└── step_results: relationship, job: relationship

flow_step_results
├── id, created_at, updated_at              (BasePublic)
├── flow_run_id → flow_runs.id              (CASCADE)
├── step_order: int
├── assistant_id → assistants.id            (SET NULL)
├── input_payload_json: JSONB | None
├── output_payload_json: JSONB | None
│     {"text":"...","file_ids":[...],"generated_file_ids":[...],"webhook_delivered":false}
├── num_tokens_input: int | None
├── num_tokens_output: int | None
├── status: str (default "pending")
├── error_message: str | None
├── flow_step_execution_hash: str | None
│     SHA256 over execution-critical fields only:
│       assistant_id + prompt text + completion_model.id
│       + mcp_policy + sorted(mcp_tool_allowlist when restricted)
│       + input_source + canonical_json(input_config)
│       + output_mode + output_type + canonical_json(output_config)
│       + output_classification_override
│     Excludes cosmetic fields (`user_description`, labels/icons, timestamps).
└── tool_calls_metadata: JSONB | None
│     ⚠️ CODEBASE NOTE: Verify that tool_calls_metadata is populated for
│     non-streaming calls in tenant_model_adapter.py. There may be a gap
│     where streaming paths capture this but non-streaming (our path) does not.
```

### 1.3 Module Registry Table

```
module_registry
├── id, created_at, updated_at
├── name: str, module_id: str (unique)
├── internal_url: str, health_endpoint: str (default "/health")
├── last_health_check_at: datetime | None
├── last_health_status: str (healthy | unhealthy | unknown)
├── enabled: bool (default True)
├── module_version: str | None
├── image_digest: str | None
├── module_api_contract: str | None
├── core_compat_min: str | None
├── core_compat_max: str | None
├── compat_status: str | None                 (compatible | incompatible | unknown)
├── release_notes_url: str | None
├── tenant_id → tenants.id, metadata_json: JSONB | None
```

**Migration:** 1 ALTER (assistants.hidden) + 6 CREATE TABLE.

---

## 2. Flow Repository (Own Aggregate Root)

**CRITICAL (from Codex review):** Flow is NOT saved via `space_repo._set_flows()`. The Space aggregate saves all nested entities recursively — adding flows would cause massive write latency as every Space update resaves all flow steps, MCP tools, and configs.

**File:** `backend/src/intric/flows/infrastructure/flow_repo.py`

```python
class FlowRepository:
    """Dedicated repository for Flow aggregate. NOT inside SpaceRepository."""

    async def create(self, flow: Flow) -> Flow: ...
    async def get(self, flow_id: UUID) -> Flow: ...
    async def get_by_space(self, space_id: UUID) -> list[Flow]: ...
    async def get_sparse_by_space(self, space_id: UUID) -> list[FlowSparse]: ...
    async def update(self, flow: Flow) -> Flow: ...
    async def delete(self, flow_id: UUID) -> None: ...
    async def get_step_result(self, flow_run_id, step_order) -> FlowStepResult | None: ...
    async def save_step_result(self, flow_run_id, result: FlowStepResult) -> None: ...
```

**Space integration:** Space entity holds `flows: list[FlowSparse]` for UI listing only (name, id, published, description). Loaded via `FlowRepository.get_sparse_by_space()` called during space hydration — NOT via `_set_flows()`.

**Modify** `space_factory.py`: After building space, fetch `FlowSparse` list from `FlowRepository` and attach.
**Modify** `space_repo.py`: Do NOT add `_set_flows`. Only read FlowSparse references.

---

## 3. Assistant Changes

### `execute_once()` with `prompt_override`

```python
async def execute_once(
    self,
    assistant_id: UUID,
    question: str,
    user: "UserInDB",                  # REQUIRED: worker has no HTTP context
    mcp_servers: list["MCPServer"] | None = None,
    file_ids: list[UUID] | None = None,
    tracing_metadata: dict | None = None,
    prompt_override: str | None = None,  # resolved {{variables}} — matches completion_service API
) -> ExecuteOnceResult:
    # NOTE (P1-7): get_space_by_assistant MUST include hidden assistants.
    # The hidden filter applies ONLY in list/UI endpoints, not by-ID lookups.
    space = await self.space_repo.get_space_by_assistant(assistant_id=assistant_id)
    assistant = space.get_assistant(assistant_id=assistant_id)

    actor = self.actor_manager.get_space_actor_from_space(space=space, user=user)
    if not actor.can_read_assistant(assistant=assistant):
        raise UnauthorizedException()
    if not space.can_ask_assistant(assistant=assistant):
        raise UnauthorizedException("Security classification mismatch")

    files = await self.file_service.get_files_by_ids(file_ids) if file_ids else []

    response, datastore_result = await assistant.ask(
        question=question,
        session=None, files=files,
        stream=False, require_tool_approval=False,
        mcp_servers_override=mcp_servers,
        prompt_override=prompt_override,      # P0-3: matches completion_service.get_response(prompt=...)
    )

    return ExecuteOnceResult(
        output=response.completion.text,
        num_tokens_input=response.total_token_count,
        num_tokens_output=count_tokens(response.completion.text),
        tool_calls_metadata=response.tool_calls_metadata,
        # P0-4: If tool_calls_metadata is None here, check tenant_model_adapter.py
        # non-stream branch — it may not populate this field. Fix in PR 2.
    )
```

**Hidden assistant filter (P1-7 — CRITICAL):**
- Default assistant **list** endpoints: add `WHERE hidden = false`
- `get_space_by_assistant()` (space_repo.py:1329): **MUST include hidden** — flow execution needs these
- `space.get_assistant(id)`: **MUST include hidden** — by-ID lookups always return all
- Only filter hidden in UI/list paths, NEVER in by-ID execution paths
**Cleanup job (hourly):** Delete hidden assistants with no `flow_step` reference.

---

## 4. Variable Resolver

**File:** `backend/src/intric/flows/application/variable_resolver.py`

```python
import re, json
from typing import Any

def build_context(flow_run_params, all_results: list) -> dict:
    context = {
        "flow_input": {
            "text": flow_run_params.input_data or "",
            **(flow_run_params.form_data or {}),
        }
    }
    for result in all_results:
        out = result.output_payload_json or {}
        step_ctx = {
            "output": out.get("text", ""),
            "file_ids": ",".join(out.get("file_ids", [])),
        }
        try:
            parsed = json.loads(out.get("text", ""))
            if isinstance(parsed, dict):
                step_ctx["output"] = parsed
        except (json.JSONDecodeError, ValueError):
            pass
        context[f"step_{result.step_order}"] = step_ctx
    return context

def interpolate(text: str, context: dict, json_safe: bool = False) -> str:
    """Replace {{path.to.value}} with resolved values.

    json_safe=True: JSON-escape values (handles quotes, backslashes, newlines)
    to prevent payload corruption when interpolating inside JSON strings.
    """
    def replacer(match):
        path = match.group(1).split(".")
        value: Any = context
        for key in path:
            if isinstance(value, dict):
                value = value.get(key, match.group(0))
            else:
                return match.group(0)
        if isinstance(value, (dict, list)):
            return json.dumps(value, ensure_ascii=False)
        result = str(value)
        if json_safe:
            # Escape characters that break JSON strings
            result = result.replace('\\', '\\\\').replace('"', '\\"')
            result = result.replace('\n', '\\n').replace('\r', '\\r')
            result = result.replace('\t', '\\t')
        return result
    return re.sub(r'\{\{(\w+(?:\.\w+)*)\}\}', replacer, text)
```

**Usage in runner:**
- System prompt: `interpolate(prompt, context, json_safe=False)` — free text
- `input_config.url`: `interpolate(url, context, json_safe=False)` — URL encoding handled by httpx
- `input_config.body`: `interpolate(body, context, json_safe=True)` — JSON context, must escape
- `output_config.url`: `interpolate(url, context, json_safe=False)`

---

## 5. SequentialFlowRunner

### Worker Transaction Isolation

**CRITICAL (from Codex review):** The ARQ `@worker.task` must NOT hold an auto-transaction during flow execution. LLM calls take 60+ seconds — holding a DB connection that entire time exhausts the Postgres pool.

**Pattern:** The runner does NOT receive a database session from the worker. Instead, it requests short-lived sessions from a session factory only when persisting results.

```python
class SequentialFlowRunner:
    def __init__(self, assistant_service, flow_repo, file_service,
                 session_factory, doc_generator=None):
        self.assistant_service = assistant_service
        self.flow_repo = flow_repo
        self.file_service = file_service
        self.session_factory = session_factory  # creates short-lived DB sessions
        self.doc_generator = doc_generator

    async def _persist_step_result(self, flow_run_id, step_result):
        """Open a short-lived DB session, save, commit, close."""
        async with self.session_factory() as session:
            await self.flow_repo.save_step_result(
                flow_run_id, step_result, session=session
            )
            await session.commit()
        # Session closed — connection returned to pool immediately

    async def execute(self, flow, flow_run_id, flow_run_params, user):
        previous_result = None
        all_results = []
        context = build_context(flow_run_params, [])

        for step in sorted(flow.steps, key=lambda s: s.step_order):
            # ── Idempotency check (ARQ retry safety) ──
            existing = await self._get_step_result(flow_run_id, step.step_order)
            if existing and existing.status == "completed":
                if step.output_mode == "http_post" and step.output_config:
                    delivered = (existing.output_payload_json or {}).get("webhook_delivered", False)
                    if not delivered:
                        url = interpolate(step.output_config["url"], context)
                        headers = self._decrypt_headers(step.output_config.get("headers", {}))
                        await self._safe_http_post(url, headers, existing.output_payload_json.get("text", ""))
                        existing.output_payload_json["webhook_delivered"] = True
                        await self._persist_step_result(flow_run_id, existing)
                previous_result = existing
                all_results.append(existing)
                context = build_context(flow_run_params, all_results)
                continue

            current_text, current_file_ids = "", []
            try:
                # ── 1. Resolve inputs ──
                current_text, current_file_ids = await self._resolve_inputs(
                    step, flow_run_params, previous_result, all_results, context)

                # ── 2. Token pre-check ──
                token_count = count_tokens(current_text)
                model_limit = step.assistant.completion_model.token_limit
                if model_limit and token_count > model_limit:
                    raise FlowStepError(
                        f"Step {step.step_order}: Input ({token_count} tokens) > "
                        f"model context ({model_limit} tokens)")

                # ── 3. Variable interpolation in system prompt ──
                resolved_prompt = interpolate(
                    step.assistant.prompt or "", context, json_safe=False)

                # ── 4. MCP tools ──
                effective_mcp = self._resolve_step_mcp_tools(step)

                # ── 5. Execute (NO open DB transaction during this call) ──
                result = await self.assistant_service.execute_once(
                    assistant_id=step.assistant.id,
                    question=current_text,
                    user=user,
                    mcp_servers=effective_mcp,
                    file_ids=current_file_ids,
                    tracing_metadata={"flow_run_id": str(flow_run_id), "step": step.step_order},
                    prompt_override=resolved_prompt,
                )

                # ── 6. Post-process ──
                out_text, out_file_ids = await self._process_output(step, result)

                # ── 7. Persist (short-lived transaction: milliseconds) ──
                step_result = FlowStepResult(
                    step_order=step.step_order,
                    assistant_id=step.assistant.id,
                    input_payload_json={"text": current_text, "file_ids": current_file_ids},
                    output_payload_json={
                        "text": out_text, "file_ids": out_file_ids,
                        "generated_file_ids": out_file_ids,
                        "webhook_delivered": False,
                    },
                    num_tokens_input=result.num_tokens_input,
                    num_tokens_output=result.num_tokens_output,
                    tool_calls_metadata=result.tool_calls_metadata,
                    status="completed",
                )
                try:
                    await self._persist_step_result(flow_run_id, step_result)
                except Exception:
                    for fid in out_file_ids:
                        await self.file_service.delete_file(fid)
                    raise

                # ── 8. Webhook ──
                if step.output_mode == "http_post" and step.output_config:
                    try:
                        url = interpolate(step.output_config["url"], context)
                        headers = self._decrypt_headers(step.output_config.get("headers", {}))
                        await self._safe_http_post(url, headers, out_text)
                        step_result.output_payload_json["webhook_delivered"] = True
                        await self._persist_step_result(flow_run_id, step_result)
                    except Exception as e:
                        raise FlowStepError(f"Step {step.step_order}: webhook failed: {e}")

                yield step_result
                previous_result = step_result
                all_results.append(step_result)
                context = build_context(flow_run_params, all_results)

            except Exception as e:
                failed = FlowStepResult(
                    step_order=step.step_order, assistant_id=step.assistant.id,
                    input_payload_json={"text": current_text, "file_ids": current_file_ids},
                    status="failed", error_message=str(e))
                await self._persist_step_result(flow_run_id, failed)
                raise
```

### MCP Tool Resolution

```python
def _resolve_step_mcp_tools(self, step):
    if step.mcp_policy == "inherit":
        return None
    allowed_ids = {tid for tid in step.mcp_tool_allowlist}
    servers = []
    for orig in step.assistant.mcp_servers:
        # ⚠️ CODEBASE CHECK: Determine if MCP servers are Pydantic models or domain entities.
        # If Pydantic v2 models → use orig.model_copy(deep=True) (deepcopy breaks internal refs)
        # If plain domain entities → use copy.deepcopy(orig) (model_copy doesn't exist)
        # Verify by checking: hasattr(orig, 'model_copy')
        server = _safe_copy(orig)
        for tool in server.tools:
            tool.is_enabled_by_default = tool.id in allowed_ids
        servers.append(server)
    return servers

def _safe_copy(obj):
    """Copy entity safely regardless of whether it's Pydantic or domain entity."""
    if hasattr(obj, 'model_copy'):
        return obj.model_copy(deep=True)
    else:
        import copy
        return copy.deepcopy(obj)
```

### Input Resolution (abbreviated — full logic in FLOW_EXECUTION_ENGINE.md)

Handles: `flow_input`, `previous_step`, `all_previous_steps`, `http_get`, `http_post`.
`http_post` input: body interpolated with `json_safe=True`.

### SSRF / HTTP / Output Processing (unchanged from v3.1)

Block private IPs + configurable `ALLOWED_INTERNAL_CIDRS`. Header injection blocklist. 3 retries. 1MB cap. Content-type routing. `Idempotency-Key` on webhooks. JSON extraction with markdown stripping.

---

## 6. Graph Endpoint (Compliance/Transparency)

**File:** `backend/src/intric/flows/presentation/flow_router.py`

```python
@router.get("/flows/{id}/graph")
async def get_flow_graph(id: UUID, flow_service: FlowService = Depends()):
    flow = await flow_service.get(id)

    nodes = [{"id": "input", "label": f"Formulär: {flow.name}",
              "type": "input",
              "fields": [f.get("id") for f in (flow.metadata_json or {}).get("form_schema", [])]}]

    for step in sorted(flow.steps, key=lambda s: s.step_order):
        nodes.append({
            "id": f"step_{step.step_order}",
            "label": step.user_description or f"Steg {step.step_order}",
            "type": "llm",
            "model": step.assistant.completion_model.name if step.assistant else None,
            "input_source": step.input_source,
            "input_type": step.input_type,
            "output_type": step.output_type,
            "has_webhook": step.output_mode == "http_post",
            "has_http_input": step.input_source in ("http_get", "http_post"),
            "mcp_policy": step.mcp_policy,
        })

    nodes.append({"id": "output", "label": "Resultat", "type": "output"})

    edges = []
    for step in sorted(flow.steps, key=lambda s: s.step_order):
        sid = f"step_{step.step_order}"
        if step.step_order == 1:
            if step.input_source in ("flow_input", "http_get", "http_post"):
                edges.append({"source": "input", "target": sid})
        if step.input_source == "previous_step" and step.step_order > 1:
            edges.append({"source": f"step_{step.step_order - 1}", "target": sid})
        elif step.input_source == "all_previous_steps":
            for prev in flow.steps:
                if prev.step_order < step.step_order:
                    edges.append({"source": f"step_{prev.step_order}", "target": sid,
                                  "style": "dashed", "label": "aggregated"})
            edges.append({"source": "input", "target": sid, "style": "dashed"})

    last_step = max(s.step_order for s in flow.steps) if flow.steps else 0
    edges.append({"source": f"step_{last_step}", "target": "output"})

    return {"nodes": nodes, "edges": edges}
```

---

## 7. Permissions + Space (unchanged pattern, new boundary)

**Permissions:** `FLOW = "flow"` in `SpaceResourceType`. Same role matrix.
**Space:** Holds `FlowSparse[]` references (read-only). NOT saved via `_set_flows`.

---

## 8. API Key v2 + Audit (unchanged from v3.1)

`flows` in `ResourcePermissions`. Scope resolution: flow_id → space_id. Audit with truncated payloads.

---

## 9. Worker Task

**P0-1 (Codex):** `worker.py:445` — `@worker.task` wraps the full task in `session.begin()`. This means any LLM call (60+ seconds) holds a DB connection open.

**P0-2 (Codex):** `worker.py:431` — `@worker.task` decorator only accepts `with_user` and `channel_type`, NOT `timeout`.

**Fix: Create `long_running_task` pattern** that does not auto-open a session. The runner uses `session_factory` for short-lived persist calls only. Timeout is configured at ARQ worker level (`job_timeout=1800` in ARQ settings), not in the decorator.

```python
# Option A: New decorator that skips session.begin()
@worker.long_running_task(channel_type=ChannelType.FLOW_RUN_UPDATES)
async def run_flow(params: FlowRunParams, container, worker_config):
    """NO session.begin() wrapper. Runner manages own DB sessions."""
    runner = container.get(SequentialFlowRunner)
    flow = await container.get(FlowService).get(params.flow_id)
    user = await container.get(UserService).get(params.user_id)
    try:
        last_result = None
        async for step_result in runner.execute(flow, params.flow_run_id, params, user):
            last_result = step_result
        await container.get(FlowRunService).complete(
            params.flow_run_id, output=last_result.output_payload_json if last_result else None)
    except Exception as e:
        await container.get(FlowRunService).fail(params.flow_run_id, str(e))
        raise

# Option B: If modifying worker.py is too invasive, use existing @worker.task
# but have the runner create its own async session factory that's independent
# of the task's session scope. The task's session is only used for the initial
# flow/user fetch, then runner uses its own sessions for step persistence.
```

**P1-8 (Codex):** Container's `SessionProxy` pattern (`container.py:1367`) assumes request-scoped sessions. The runner must use explicit `session_factory` for all DB ops during long-running execution, never the ambient `SessionProxy`.

**ARQ timeout:** Set `job_timeout=1800` in ARQ worker config (`arq.py`), not in decorator.

---

## 10. Cleanup Jobs

**Active cancellation:** `arq_redis.abort_job()` on DELETE.
**Zombie cleanup (hourly):** 2h threshold.
**Hidden assistant cleanup (hourly):** Delete orphans with no flow_step reference.
**Data retention:** `delete_old_flow_runs()` with `generated_file_ids` GC.
**P1-9 (Codex):** `FileService.delete_file` enforces owner check (`file_service.py:93`). Flow cleanup job running as system can't delete user-uploaded flow files. **Fix:** Add `delete_file_system(file_id, tenant_id)` — tenant-scoped system GC path that bypasses user ownership but requires tenant authorization.

### Flow Validation (API time)

**P1-6 (Codex):** Step 1 with `input_source="previous_step"` = empty input (no previous step exists).
**Fix:** On flow create/update, validate:
- If `step_order == 1`: reject `previous_step` and `all_previous_steps`. Default to `flow_input`.
- If `step_order > 1` and `input_source == "flow_input"`: allowed (step re-reads original form data).
- If `step_order > 1` and `input_source == "http_get|http_post"`: allowed (external fetch).

---

## 11: Module Template (unchanged from v3.1)

---

## 12: Widget (unchanged from v3.1)

---

## 13: Auth Broker + Ticket Handoff (v3.9)

**Production default:** Auth Broker with one-time ticket exchange. No wildcard shared cookie domain.

### 13.1 Auth Broker Endpoints

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/api/v1/auth/module-initiate` | GET | None (browser redirect) | Start module auth. Params: `module_client_id`, `return_to`, `handoff_state`. Backend stores state in Redis and sends compact `state_id` to IdP. |
| `/api/v1/auth/callback` | GET | OIDC callback | **Conditional branch:** if state payload contains `module_client_id` → mint one-time ticket and redirect to module callback (`ticket` + `handoff_state`). If not → return JSON `{ "access_token": "..." }` for core login flow (unchanged). |
| `/api/v1/auth/ticket-exchange` | POST | `Authorization: Bearer sk_<module_key>` | Exchange one-time ticket for module-scoped JWT. Body: `{ "ticket": "<ticket>" }`. |
| `/api/v1/auth/module-refresh` | POST | `Authorization: Bearer sk_<module_key>` | Refresh expiring module-scoped JWT. Body: `{ "token": "<module_jwt>" }`. |
| `/api/v1/auth/module-logout` | POST | Module-scoped JWT | Revoke module session, clear module cookie, and update revocation watermark. |

### 13.2 Module Client Configuration

`module_clients` table (V1 managed via seed/sysadmin API; no end-user admin UI in this release):

```
module_clients
├── id, created_at, updated_at
├── tenant_id → tenants.id (CASCADE)
├── client_id: str (unique per tenant)          e.g., "speech-to-text"
├── callback_url: str                           Exact-match only. e.g., "https://taltilltext.eneo.sundsvall.se/auth/callback"
├── active: bool (default True)
├── allowed_scopes: JSONB | None                Optional permission constraints for module-scoped JWTs
├── module_api_contract: str                    e.g., "core-api-v1"
└── sk_key_id → api_keys.id (SET NULL)          Links to the module's sk_ key for exchange/refresh auth
```

### 13.3 OIDC State + Ticket Lifecycle (Redis)

```python
# module-initiate
state_id = secrets.token_urlsafe(24)
state_payload = {
    "tenant_id": str(tenant.id),
    "module_client_id": module_client_id,
    "handoff_state": handoff_state,
    "return_to": return_to,
    "nonce": nonce,
    "pkce_verifier": pkce_verifier,
}
await redis.set(f"oidc_state:{state_id}", json.dumps(state_payload), ex=600)  # 10m TTL

# callback
state_json = await redis.getdel(f"oidc_state:{state_id}")  # single use
if not state_json:
    raise HTTPException(401, "Invalid or expired state")

# ticket mint
ticket = secrets.token_urlsafe(32)
await redis.set(f"module_ticket:{ticket}", json.dumps(ticket_payload), ex=30)  # 10-30s TTL

# ticket exchange
ticket_json = await redis.getdel(f"module_ticket:{ticket}")  # single use
if not ticket_json:
    raise HTTPException(401, "Ticket expired or already consumed")
```

### 13.4 Module JWT Claims, Auth Patterns, and Refresh

**Module-scoped JWT claims:** `sub`, `tenant_id`, `aud`, `scope`, `auth_time`, `iat`, `exp`, `jti`.

**Backend call patterns (explicit):**
- **User-context calls (module BFF acting for logged-in user):** `Authorization: Bearer <module-scoped-jwt>`
- **System-context calls (ticket-exchange, refresh, health, anonymous widget):** `Authorization: Bearer sk_<module_key>`

Backend authorization logic must not trust custom user-id headers; identity comes from signed JWT claims.

**Refresh contract (`/api/v1/auth/module-refresh`):**
- Validate JWT signature and issuer.
- Validate module key binding: `sk_` key must map to JWT `aud`.
- Validate grace window: `now < exp + MODULE_REFRESH_GRACE_SECONDS`.
- Validate absolute session: `now - auth_time < MODULE_MAX_SESSION_HOURS`.
- Validate revocation watermark: `iat > logout_after`.
- Validate user/tenant still active.
- Return fresh 15-minute JWT preserving original `auth_time`.
- On failure: module BFF clears cookie and redirects to login.

**Backend implementation note (decode trap):** `module-refresh` receives tokens that may have just expired. Do not use strict `get_current_user` dependency for this path. Decode with signature validation and manual expiry enforcement (`verify_exp=False` or equivalent), then apply grace-window and absolute-timeout checks explicitly.

### 13.5 Security Controls

| Control | Implementation |
|---------|---------------|
| OIDC state transport | Compact `state_id` in URL; full state payload in Redis (10m TTL) |
| OIDC state consume | Atomic `GETDEL` in Redis (single use) |
| Callback CSRF | Module BFF verifies `handoff_state` against temporary `__Host-handoff` cookie |
| Ticket TTL | 10–30 seconds (starts when backend mints ticket) |
| Ticket consume | Atomic `GETDEL` in Redis |
| Ticket binding | `tenant_id` + `module_client_id` verified on exchange |
| Exchange/refresh auth | Module `sk_` key required in Authorization header |
| JWT audience | `aud` = `module_client_id`, enforced by backend middleware |
| Redirect allowlist | Exact-match from `module_clients.callback_url` (no wildcards) |
| Cache headers | `Cache-Control: no-store` on callback responses |
| Referrer policy | `Referrer-Policy: no-referrer` on callback routes |
| Open redirect | Reject any `return_to` not matching registered callback URL |

### 13.6 Backwards Compatibility

Core web login flow remains unchanged:
- Callback checks stored state payload for `module_client_id`
- If absent → existing JSON token response for core frontend
- If present → module ticket redirect path

### 13.7 Acceptance Criteria

- [ ] Module login succeeds via broker handoff (`handoff_state` validated)
- [ ] Ticket replay fails (consumed on first exchange)
- [ ] Ticket exchange with wrong `module_client_id` fails (403)
- [ ] Expired ticket fails (after TTL)
- [ ] OIDC state replay fails (state consumed on first callback)
- [ ] Module-scoped JWT cannot access core-only endpoints
- [ ] Refresh works within grace + absolute timeout windows
- [ ] Refresh fails after `auth_time` absolute timeout
- [ ] Refresh and API calls fail after `logout_after` revocation
- [ ] Core web login remains unchanged and functional

### 13.8 Migration from Legacy Shared Cookie

If any environment previously used `SESSION_COOKIE_DOMAIN` on `.eneo.sundsvall.se`:
1. Deploy broker endpoints alongside existing cookie auth
2. Set `MODULE_AUTH_STRATEGY=handoff` on all module containers
3. Remove `SESSION_COOKIE_DOMAIN` from backend config
4. Verify module logins work via broker
5. Remove any `eneo_sso` cookie references from module code

### 13.9 Implementation Gotchas

**1. Scrub `.env` files:** Remove `SESSION_COOKIE_DOMAIN=.eneo.sundsvall.se` from `.env`, `.env.example`, and templates.

**2. Callback URL cleanup:** Module callback route must clear `ticket` from URL via second redirect after successful exchange.

**3. `module_clients` management path:** Use DB seed/sysadmin API in V1. Do not build end-user admin CRUD in this release.

---

## 14. Frontend Implementation

### Flow Editor (Inline Builder)

**FlowStepCard.svelte:** Create new / Use existing toggle. Inline prompt, model, knowledge, MCP, I/O config. Hidden assistant lifecycle.

**FlowFormBuilder.svelte:** Edit `form_schema`. Add/remove/reorder fields.

**FlowRunForm.svelte:** Render form_schema as dynamic form for execution.

**Auto-save / Draft State:**
Building a multi-step flow takes time. The frontend auto-saves with debounced PATCH (500ms):
- First save: `POST /flows` creates with `published: false`
- Subsequent edits: `PATCH /flows/{id}` (debounced)
- Top-right indicator: "Sparad ✓" (green) / "Sparar..." (gray) / "Ej sparad" (yellow)
- Hidden assistants created immediately on "Add step" so prompts are always persisted

**Variable Picker ("Infoga variabel" button):**
Above the prompt textarea in each step, a `{ }` button opens a context-aware dropdown:
- Reads `form_schema` → shows: "Inmatning: Namn", "Inmatning: Personnummer", etc.
- Reads previous steps → shows: "Steg 1: Analys (output)", "Steg 2: Data (output)"
- If previous step output_type=json → expands sub-fields from example or schema
- Click inserts `{{flow_input.namn}}` or `{{step_1.output}}` at cursor position
- Users never type `{{...}}` syntax manually

**Model Capability Warnings:**
Yellow warning icon between steps when incompatible:
- Step outputs `image` → next model lacks vision → "⚠️ Modellen stöder inte bildanalys"
- Step outputs `audio` → next model lacks whisper → "⚠️ Modellen kan inte transkribera ljud"
- Step outputs `json` → next expects `image` → "⚠️ Steg 1 producerar JSON, Steg 2 förväntar sig bild"
- Check model capabilities via existing model metadata (vision_support, audio_support flags)

**Step compatibility matrix** (frontend validation, not backend-enforced):
- text/json output → text, json, any input ✅
- file outputs (image, audio, pdf, docx) → matching type, file, any ✅
- text output → image input ⚠️ warning

**Reuse Last Test Data ("Återanvänd senaste testdata"):**
On the Run tab, a ghost button fetches the user's most recent `FlowRun` for this flow (`GET /flow-runs?flow_id=X&limit=1`), takes `input_payload_json`, and pre-fills the form fields + file dropzones. **Zero backend changes.** Eliminates re-typing/re-uploading during iterative flow debugging.

**Semantic Step Names in Visualizer:**
`user_description` field on `flow_steps` is exposed as a bold title input at the top of each step card. The graph endpoint uses this as the node label: *Formulär → Transkribering → Sammanfattning → Webhook* instead of *Step 1 → Step 2 → Step 3*. Already supported in schema — just surface prominently in UI.

### Översikt Tab (Flow Preview + Compliance Export)

**New tab** alongside "Redigera" (Edit) and "Resultat" (Results).

**Uses:** `@xyflow/svelte` (Svelte Flow) in **read-only mode**.

```
frontend/apps/web/src/lib/features/flows/components/
├── FlowVisualizer.svelte       ← Svelte Flow canvas (read-only)
├── FlowNodeInput.svelte        ← Green node: form input fields
├── FlowNodeLLM.svelte          ← Orange node: step with model + I/O badges
├── FlowNodeOutput.svelte       ← Yellow node: final result
└── FlowExportButtons.svelte    ← JSON + PNG + SVG download buttons
```

**Implementation:**
1. On mount: fetch `GET /api/v1/flows/{id}/graph`
2. Map `nodes` → Svelte Flow nodes with custom components
3. Map `edges` → Svelte Flow edges (solid for direct, dashed for aggregated)
4. Auto-layout: `dagre` library for automatic node positioning
5. Minimap enabled (bottom-right, matching Svelte Flow default)
6. Pan/zoom enabled, node selection disabled (read-only)

**Node styling:**
- Input node: green (#c8e6c9), shows form field names
- LLM step node: orange (#ffcc80), shows model name + I/O type badges
- Output node: yellow (#fff9c4)
- Webhook indicator: small outbound arrow icon on node
- HTTP input indicator: small inbound arrow icon on node
- Classification badge: "K3" / "K2" / "K1" colored chip if security_enabled

**Stateful visualizer (when viewing a completed FlowRun):**
When Översikt is opened from a specific run result (not the editor), pass `?run_id=` to the graph endpoint. The endpoint enriches nodes with execution data:
- Completed steps → green nodes with execution time ("2.3s") and token count
- Failed steps → red nodes with error summary
- Skipped steps (retry) → gray nodes
- This makes debugging visual — instantly see where and why a flow failed

**Graph endpoint with run data** (`GET /api/v1/flows/{id}/graph?run_id=xxx`):
```python
if run_id:
    step_results = await flow_repo.get_step_results(run_id)
    for node in nodes:
        if node["type"] == "llm":
            result = next((r for r in step_results if f"step_{r.step_order}" == node["id"]), None)
            if result:
                node["status"] = result.status
                node["execution_time_ms"] = (result.updated_at - result.created_at).total_seconds() * 1000
                node["tokens"] = {"input": result.num_tokens_input, "output": result.num_tokens_output}
                node["error"] = result.error_message
```

**Export buttons:**
- "Ladda ner flöde (JSON)" → full flow definition for compliance audit
- "Ladda ner diagram (PNG)" → uses `html-to-image` library (`toBlob()` on the Svelte Flow wrapper DOM ref). Captures exact layout and CSS without backend rendering.
- "Ladda ner diagram (SVG)" → SVG export for print/reports

**Compliance Audit Report (V1.5 — add after DocumentGeneratorService lands in PR 7):**
"Ladda ner granskningsrapport" button generates a PDF containing:
1. Flow name, creator, creation/modification dates
2. Embedded PNG of the Svelte Flow diagram
3. Complete form schema (what data is collected)
4. Each step: system prompt text, model name, classification level, I/O types, webhook URLs
5. If generated from a specific run: execution times, token counts, pass/fail per step
This is what municipal auditors need when evaluating automated AI decision-making.

**Package:** `npm install @xyflow/svelte dagre @dagrejs/dagre html-to-image`

### Admin: Module Registry UI (unchanged)

---

## 15. IoC Container

- `FlowRepository` (dedicated, NOT inside SpaceRepository)
- `FlowFactory`, `FlowRunFactory`
- `FlowAssembler`, `FlowRunAssembler`
- `FlowService`, `FlowRunService`, `ModuleService`
- `SequentialFlowRunner(assistant_service, flow_repo, file_service, session_factory)`
- `HealthChecker(module_service)`

---

## 16. PR Sequence

### PR 1: Data + Domain Foundations
- Alembic migration: 1 ALTER + 6 CREATE
- Flow domain entities/factories
- **FlowRepository** (own aggregate root, NOT in space_repo)
- Module registry entity/service
- Space: FlowSparse references only (read via FlowRepository)
- Permissions, hidden assistant filter
- IoC wiring, audit types

### PR 2: Flow APIs + Execution Engine
- Routers/models/assemblers + API Key v2 wiring
- **Graph endpoint** (`GET /flows/{id}/graph`)
- **Resume endpoint** (`POST /flow-runs/{id}/resume`) with deterministic execution-hash checks
- `execute_once()` + `mcp_servers_override` + `prompt_override`
- `_safe_copy()` for MCP entity cloning
- SequentialFlowRunner with: **manual session factory**, variable resolver (with JSON-safe escaping), idempotent steps, token pre-check
- SSRF + http_get + http_post input
- Classification validation + egress check
- Header encryption (if existing utility) or documented plaintext

### PR 3: Worker + Jobs + WebSocket + Cleanup
- ARQ task (**explicit transaction isolation**, 30 min timeout)
- WebSocket channel
- Job cancellation, zombie cleanup, orphan assistant cleanup

### PR 4: Frontend — Flow Builder + Översikt
- intric-js endpoints + types
- Navigation, routes
- Inline builder (FlowStepCard, FlowStepList, FlowFormBuilder, FlowRunForm)
- Variable autocomplete, step compatibility indicators
- **Översikt tab: Svelte Flow visualizer** (read-only)
- **Export buttons:** JSON, PNG, SVG
- API Key dialog: Flöden card
- Translations

### PR 5: Module Infrastructure + Auth Broker
- Template, health registry, compose overlay
- **Auth Broker endpoints:** `module-initiate`, `ticket-exchange`, `module-refresh`, `module-logout`
- **Callback branching:** conditional JSON (core) / 302+ticket (module) in `/api/v1/auth/callback`
- **OIDC state transport:** Redis-backed `state_id` + single-use `GETDEL`
- **Module callback CSRF:** `handoff_state` cookie roundtrip verification in module template
- **Module client registration:** `module_clients` table + seed/sysadmin API path (no end-user admin CRUD in V1)
- **Compatibility gate:** contract-first check on `module_api_contract`; advisory core version bounds
- **One-time ticket:** Redis storage with atomic consume, TTL, client binding
- **Module-scoped JWT:** `aud` claim, constrained permissions, backend middleware enforcement
- **Session lifecycle:** 15m token TTL + refresh grace + `auth_time` absolute timeout + `logout_after` revocation
- **Security headers:** `Cache-Control: no-store`, `Referrer-Policy: no-referrer` on callbacks
- **Redirect allowlist:** exact-match validation from `module_clients.callback_url`

### PR 6: STT + Widget Modules
- Both modules, Dockerfiles, e2e test

### PR 7: Hardening + Document Generation
- Retention, file GC, DocumentGeneratorService
- Compatibility override endpoint with full audit trail
- tool_calls_metadata verification for non-streaming
- Tests, docs, CHANGELOG

---

## 17. Verification Checklist

### Aggregate Boundary
- [ ] FlowRepository handles all Flow CRUD (not space_repo)
- [ ] Space only holds FlowSparse references
- [ ] Space update does NOT trigger flow saves

### Transaction Isolation
- [ ] Worker task does NOT hold DB connection during LLM calls
- [ ] `_persist_step_result` opens + commits + closes in milliseconds
- [ ] 3 concurrent flow runs don't exhaust connection pool

### Entity Cloning
- [ ] `_safe_copy()` works for MCP servers (test both Pydantic + domain entity paths)
- [ ] No DetachedInstanceError from SQLAlchemy-loaded objects

### Compliance Export
- [ ] `GET /flows/{id}/graph` returns correct node/edge structure
- [ ] Översikt tab renders Svelte Flow diagram
- [ ] JSON export downloads complete flow definition
- [ ] PNG/SVG export captures diagram

### Resume Determinism
- [ ] `flow_step_execution_hash` computed from execution-critical fields only
- [ ] Cosmetic edits (`user_description`, labels, timestamps) do NOT force full rerun
- [ ] Upstream logic change forces full rerun from step 1
- [ ] No upstream change resumes from first failed step

### Auth Broker
- [ ] Module login succeeds via broker ticket handoff (`handoff_state` verified)
- [ ] OIDC state replay fails (`state_id` consumed once via `GETDEL`)
- [ ] Ticket replay fails (consumed on first exchange)
- [ ] Ticket exchange with wrong module_client_id fails
- [ ] Expired ticket rejected
- [ ] Redirect to unregistered callback URL rejected
- [ ] Module-scoped JWT cannot access core-only endpoints
- [ ] Refresh succeeds within grace window and absolute timeout
- [ ] Refresh fails after absolute timeout (`auth_time`)
- [ ] Refresh fails when `iat <= logout_after`
- [ ] Core web login unchanged and functional
- [ ] Logout clears module-local cookie only

### (All v3.1 checks also apply — inline builder, form schema, variables, I/O types, security, widget, modules)

---

## 18. Assumptions

- Flow is its own aggregate root (not nested in Space)
- Worker manages own DB sessions (no auto-transaction)
- MCP server type verified at implementation time (model_copy vs deepcopy)
- Webhook header encryption depends on existing codebase utilities
- tool_calls_metadata capture verified for non-streaming path
- Svelte Flow is read-only visualization (no flow building/importing)
- Graph endpoint is lightweight (no separate table, computed from flow data)
- Auth Broker + Ticket Handoff is the production default (no shared cookie)
- Redis is available for ticket storage and atomic consume-once
- No IdP per-module redirect URI registration required (broker centralizes this)
- Module-scoped JWT prevents blast-radius escalation from compromised modules
- Docker Compose is the primary deployment model for municipal operations
- All v3.1 assumptions also apply
