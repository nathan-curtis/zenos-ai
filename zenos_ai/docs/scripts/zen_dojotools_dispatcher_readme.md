# Zen DojoTool Dispatcher — v5.1.0

**File:** `packages/zenos_ai/dojotools/dojotools_dispatcher.yaml`
**Automations:** `zen_dojotool_dispatcher`, `zen_dojotool_tool_router`, and supporting routers

---

## Overview

The Dispatcher is ZenOS-AI's event-driven inter-tool communication layer. Instead of tools calling each other directly via `action: script.*` (which hard-faults if the script is missing or renamed), all tool-to-tool calls go through the dispatcher via the event bus. Unknown tools return a structured error. Nothing crashes.

**Three benefits over direct script calls:**

1. **Fault isolation** — unregistered tools return `dojotool_dispatch_error` rather than killing the sequence.
2. **Version routing** — `tool + version` resolves to a specific script. Tools can be renamed or versioned without touching callers.
3. **Observability** — every inter-tool call is visible on the event bus at no extra cost.

Direct `action: script.*` calls still work and are unaffected. Migrate opportunistically.

---

## Two-Tier Architecture

HA does not support templated action names (`action: script.{{ _tool }}`). The dispatcher solves this with two automations in series.

```
Caller
  └─ fires zen_event(kind: dojotool_call)
       └─ zen_dojotool_dispatcher (Tier 1)
            ├─ Explicit branch (custom field-mapping needed) → calls script directly → fires dojotool_return
            └─ Generic arm (standard payload shape) → fires zen_event(kind: dojotool_route)
                 └─ zen_dojotool_tool_router (Tier 2)
                      └─ Explicit branch → calls backing script → fires dojotool_return
```

### Tier 1 — `zen_dojotool_dispatcher`

Listens for `zen_event(kind: dojotool_call)`. The registry is a `choose` block:

- **Explicit arms** handle tools that need custom field-mapping (e.g. `zen_dojotools_provisioner`, `zen_dojotools_filecabinet`). The arm maps payload fields to script fields, calls the script, captures `response_variable: _result`, then fires `dojotool_return`.
- **Generic arm** matches `^zen_dojotools_|^zen_admintools_` tools not listed above. Fires `zen_event(kind: dojotool_route)` with full routing info. Tier 2 handles dispatch.
- **Default (unknown tool)** fires `zen_event(kind: dojotool_dispatch_error)` with `error: unknown_tool`. No hard crash.

The generic arm is also the natural seam for a future event-bus audit logger — every generic dispatch emits `dojotool_route` with full metadata.

### Tier 2 — `zen_dojotool_tool_router`

Listens for `zen_event(kind: dojotool_route)`. Handles tools that need standard dispatch but not custom field-mapping. Fires `dojotool_return` directly after calling the backing script.

**Adding a new generic tool:** add one arm to the Tool Router. No dispatcher change required. The dispatcher generic arm catches it automatically.

**Adding a tool with custom field-mapping:** add an explicit arm in the Dispatcher instead.

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
| `dojotool_route` | Dispatcher → Tool Router | `tool`, `version`, `correlation_id`, `payload` |
| `dojotool_return` | Dispatcher or Router → Caller | `correlation_id`, `tool`, `version`, `result` |
| `dojotool_dispatch_error` | Dispatcher → Bus | `tool`, `version`, `correlation_id`, `error`, `message` |

All events are `zen_event` type. Filter by `event_data.event.kind`.

---

## Tool Registry

### Tier 1 Explicit Arms (Dispatcher)

Tools with custom payload mapping handled in `zen_dojotool_dispatcher`:

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
| `zen_dojotools_notification_router` v1 | Legacy notification router (deprecated — prefer Postman) |
| `zen_dojotools_urgency_handler` v1 | **Stub** — registered, handler not yet implemented |

### Tier 2 Arms (Tool Router)

Tools dispatched generically via `zen_dojotool_tool_router`:

| Tool | Notes |
|---|---|
| `zen_dojotools_inspect` v1 | Standard field-pass (duplicate arm for direct route path) |
| `zen_dojotools_todo` v1 | To-do list CRUD |
| `zen_dojotools_calendar` v1 | Calendar read/write |
| `zen_admintools_summarizer_act` v1 | Duplicate arm (prompt_loader whitelist) |
| `zen_dojotools_notification_router` v1 | Legacy (deprecated) |
| `zen_dojotools_security_manager` v1 | Security panel — arm/disarm/read_state |
| `zen_dojotools_postman` v1 | Canonical notification dispatch |
| `zen_dojotools_room_manager` v1 | Room topology — portals, occupants, boundary links |
| `zen_dojotools_labels` v1 | Label CRUD — target_entities, label_list |
| `zen_dojotools_covers` v1 | Cover control — position, tilt, scene |
| `zen_dojotools_climate` v1 | Climate control — temperature, hvac_mode, fan |
| `zen_dojotools_spamaster` v1 | Spa master controller — full field set |
| `zen_dojotools_announce` v1 | TTS announcement |
| `zen_dojotools_autovac` v1 | Vacuum management |

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
- **`zen_dojotools_notification_router` is deprecated.** The dispatcher still routes it for backwards compatibility. Prefer `zen_dojotools_postman`.
- **Routed tools fire `dojotool_return` from the Tool Router**, not from the Dispatcher. The Dispatcher skips its own return-fire step when `_route_externally: true`. Do not rely on the Dispatcher firing `dojotool_return` for tools that went through the generic arm.
- **The dispatcher runs `mode: parallel, max: 20`.** Twenty concurrent dojotool calls can be in flight simultaneously. Each receives its own correlated return.
