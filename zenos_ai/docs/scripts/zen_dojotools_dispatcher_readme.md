# Zen DojoTool Dispatcher — v5.3.0

**File:** `packages/zenos_ai/dojotools/dojotools_dispatcher.yaml`
**Automations:** `zen_dojotool_dispatcher` and supporting routers (single dispatch automation as of v5.2.0 — see below)

---

## Overview

The Dispatcher is ZenOS-AI's event-driven inter-tool communication layer. Instead of tools calling each other directly via `action: script.*` (which hard-faults if the script is missing or renamed), all tool-to-tool calls go through the dispatcher via the event bus. Unknown tools return a structured error. Nothing crashes.

**Three benefits over direct script calls:**

1. **Fault isolation** — unregistered tools return `dojotool_dispatch_error` rather than killing the sequence.
2. **Version routing** — `tool + version` resolves to a specific script. Tools can be renamed or versioned without touching callers.
3. **Observability** — every inter-tool call is visible on the event bus at no extra cost.

Direct `action: script.*` calls still work and are unaffected. Migrate opportunistically.

---

## Single-Automation Architecture (v5.2.0+)

Before v5.2.0, this was a two-automation design: `zen_dojotool_dispatcher` handled explicit field-mapped tools directly and forwarded everything else via a `zen_event(kind: dojotool_route)` to a second automation, `zen_dojotool_tool_router`, which held the generic arms. As of **v5.2.0 (2026.6.0)**, that split was merged into one automation with one routing table — the `dojotool_route` event and the Tool Router automation are both gone. If you see either mentioned anywhere (comments, old handoffs, memory), it's describing the pre-merge architecture.

```
Caller
  └─ fires zen_event(kind: dojotool_call)
       └─ zen_dojotool_dispatcher
            ├─ Registered arm (choose block, one per tool) → calls backing script → fires dojotool_return
            └─ default (no matching arm) → fires zen_event(kind: dojotool_dispatch_error), error: unknown_tool
```

Listens for `zen_event(kind: dojotool_call)`. The registry is a single `choose` block — one arm per `tool` + `version` pair. Every arm maps payload fields to script fields, calls the script, captures `response_variable: _result`; the automation then fires `dojotool_return` once at the end for every path (including the `default` unknown-tool fault) with the accumulated `_result`.

**Adding a new tool:** add one `choose` arm mapping `_payload.get(field, default)` to the backing script's fields. That's the entire API surface — no second automation, no routing event to fire.

---

## Caller Pattern

```yaml
# 1. Generate a correlation_id (use guidgen or a UUID)
- action: script.zen_dojotools_calculator
  data:
    operator: guidgen
  response_variable: _guid_result

# 2. Arm wait_for_trigger on correlated return
# 3. Fire the dojotool_call event
- event: zen_event
  event_data:
    event:
      kind: dojotool_call
      tool: zen_dojotools_filecabinet
      version: "1"           # optional, defaults to "1"
      correlation_id: "{{ _guid_result.result }}"
      payload:
        action_type: read
        volume_entity_id: "{{ my_cabinet }}"
        key: my_drawer
        caller_token: "{{ caller_token }}"

# 4. Receive correlated return
- wait_for_trigger:
    - trigger: event
      event_type: zen_event
      event_data:
        event:
          kind: dojotool_return
          correlation_id: "{{ _guid_result.result }}"
  timeout: '00:00:30'
```

The `correlation_id` field is required. A call without it returns `dojotool_dispatch_error` with `error: missing_correlation_id` and stops.

---

## Event Reference

| Event | Direction | Key Fields |
|---|---|---|
| `dojotool_call` | Caller → Dispatcher | `tool`, `version`, `correlation_id`, `payload` |
| `dojotool_return` | Dispatcher → Caller | `correlation_id`, `tool`, `version`, `result` |
| `dojotool_dispatch_error` | Dispatcher → Bus | `tool`, `version`, `correlation_id`, `error`, `message` |

All events are `zen_event` type. Filter by `event_data.event.kind`.

---

## Tool Registry

One flat `choose` block, one arm per `tool` + `version` pair — no tiering, no duplicate arms:

| Tool | Notes |
|---|---|
| `zen_dojotools_provisioner` v1 | Full cabinet provisioning field set |
| `zen_admintools_cabinetadmin_factory` v1 | Cabinet factory — confirm_action gate |
| `zen_admintools_cabinetadmin` v1 | Cabinet admin — mode/schema_version |
| `zen_dojotools_filecabinet` v1 | Large field set including ACL, label, expiry |
| `zen_dojotools_inspect` v1 | Entity inspection — output_fields, infer, drawer_key |
| `zen_dojotools_query` v1 | Entity query — filter_json, target_entities |
| `zen_dojotools_index` v1 | Index build — index_command, filter_json (JSON-safe deserialization) |
| `zen_dojotools_manifest` v1 | Manifest rebuild — show_hidden, show_stacks, extended |
| `zen_admintools_summarizer_act` v1 | Routes to `zen_admintools_prompt_loader` mode=whitelist |
| `zen_dojotools_urgency_handler` v1 | **Stub** — registered, handler not yet implemented |
| `zen_dojotools_todo` v1 | To-do list CRUD |
| `zen_dojotools_calendar` v1 | Calendar read/write |
| `zen_dojotools_security_manager` v1 | Security panel — arm/disarm/read_state |
| `zen_dojotools_postman` v1 | Canonical notification dispatch (`resolve_and_dispatch` default mode) |
| `zen_dojotools_room_manager` v1 | Room topology — portals, occupants, boundary links |
| `zen_dojotools_labels` v1 | Label CRUD — target_entities, label_list |
| `zen_dojotools_covers` v1 | Cover control — position, tilt, scene |
| `zen_dojotools_climate` v1 | Climate control — temperature, hvac_mode, fan |
| `zen_dojotools_spamaster` v1 | Spa master controller — full field set |
| `zen_dojotools_announce` v1 | TTS announcement |
| `zen_dojotools_autovac` v1 | Vacuum management |
| `zen_stack_firefly` v1 | Firefly III Lens Bus provider — direct dispatch arm alongside its registry-based Lens Bus routing |
| `zen_stack_battery` v1 | Battery Notes Lens Bus provider (2026.7.1) — same dual-path shape as firefly above |

**`zen_dojotools_notification_router` was removed entirely in v5.3.0 (2026-07-04)** — no backing script existed for it. A caller using that legacy tool name now falls through to `default` (`unknown_tool`). Use `zen_dojotools_postman` directly instead.

---

## Supporting Routers (same file)

### `zen_identity_manifest_router`
Listens for `zen_event(kind: identity_manifest_rebuild)` (fired by Provisioner and CabinetAdmin). Calls `zen_dojotools_manifest` to rebuild `zen_library_manifest` in the household cabinet. Mode: `queued` — provision/deprovision sequences serialize cleanly. Fire-and-forget.

### `zen_label_mutation_router`
Listens for `zen_event(kind: label_mutation)` (fired by Labels on any successful write). Calls `zen_dojotools_index (mode: build_compact_index)` to rebuild `_compact_index`. Also rebuilds on HA start and at 06:00, 12:00, 18:00, 00:00. Mode: `queued` — burst label ops don't race each other.

### `zen_scheduler_drain_router`
Trickle recovery for shed pipeline components. Fires `summary_force` for the most-stale shed-eligible component when queue depth stays below `drain_below` for `s_min_minutes`. Starvation guard at `/10` minutes forces recovery if any component's kata exceeds `s_max_minutes`. Config in `zen_scheduler_config` drawer of household root cabinet. Mode: `single`.

### `zen_flynn_health_resummarize`
Fires `summary_force` for `zen_system` whenever `sensor.zen_flynn_health` changes state, debounced 30s. Ensures kata reflects Flynn recovery or degradation without waiting for the next scheduler cycle. Mode: `single`.

### `zen_cabinet_vi_autorepair`
Catches `cabinet_vi_degraded` events emitted by Inspect when a cabinet VolumeInfo is stored as a JSON string. Calls `zen_admintools_cabinetadmin mode=repair_volumeinfo`. Emits `cabinet_vi_repaired` on success, `cabinet_vi_repair_failed` on error. Mode: `queued, max: 5`.

---

## Caveats

- **`correlation_id` is mandatory.** Calls without it are rejected with `dojotool_dispatch_error` before touching the registry.
- **`zen_dojotools_urgency_handler` is a stub.** It is registered and will accept calls, but the backing implementation does not exist yet. Response: `status: stub`.
- **`zen_dojotools_notification_router` no longer exists.** Removed entirely in v5.3.0 — there was never a backing script behind it, only the dispatcher arm. A caller using the old tool name gets `unknown_tool` now. Use `zen_dojotools_postman` directly.
- **The dispatcher runs `mode: parallel, max: 20`.** Twenty concurrent dojotool calls can be in flight simultaneously. Each receives its own correlated return.
- **`zen_stack_firefly` and `zen_stack_battery` have direct dispatcher arms** in addition to being Lens Bus registry providers. `lens_bus_autoreg.md` states stack providers don't need a dispatcher arm (the registry is the routing mechanism) — these two have one anyway, likely predating or running alongside that pattern. Not a conflict in practice: the arm here is for direct `dojotool_call` invocation, the Lens Bus registry is for `stacks_by_anchor` consumer routing. Worth a look if you're touching either path.
