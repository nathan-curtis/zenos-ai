# Custom Templates — ZenOS-AI 2026.4.0 'Ectoplasm'

*Jinja templates powering prompt assembly, context construction, and query processing*

---

ZenOS-AI's cognitive surface runs on three Jinja template libraries. These live in `custom_templates/zenos_ai/` and are loaded by Home Assistant at startup.

---

## Templates

### `zen_os_1.jinja` — Prompt Engine & Macro Library

The core cognitive assembly engine. Everything Friday knows about herself, her household, and the current state of the home is compiled here.

**Entry point:** `render_prompt(ai_entity)` — single macro that runs the full pipeline and returns Friday's system prompt. This is what `conversation_agent_prompt_template.yaml` calls.

Key capabilities:

* Full persona resolution — jacket, core, companion, environment schema
* Highlander cabinet resolution via resolver sensors
* Identity roster (`identity_roster()`) and manifest loader (`identity_manifest_loader()`)
* Home overview: presence, active components, home mode, quiet/work hours
* Wake scene assembly from essence environment block
* Prompt integrity check (`prompt_health_check()`) — schema, signature, manifest presence
* Prompt length audit (`prompt_length_audit()`) — per-section character counts, 9 sections
* `envelope()` — canonical OS-level response shape (Zammad #10297, pilot). Separates execution `status` from domain-level `result` state. `zen_health_report` and `zen_dojotools_locks` are the first two real consumers
* Flynn detection chain: `zen_flynn_override` → blank persona → explicit `flynn` label → resolver error → `prompt_system_flynn()`
* `prompt_system_flynn()` — hardcoded fallback prompt, zero cabinet dependencies, always works

**Used by:** `conversation_agent_prompt_template.yaml`, `zen_dojotools_supersummary`, `zen_dojotools_profile`, `zen_dojotools_identity`, `sensor.zen_prompt_health`, `sensor.zen_prompt_length`

→ **[Full reference](zen_os1_jinja.md)**

---

### `zen_query.jinja` — ZenQuery Filter Engine

The ZQ-1 query language implementation. Provides deterministic, label-graph-driven entity filtering with forward (query) and reverse (redaction) modes.

Used internally by HyperIndex and the Index tool for hypergraph traversal, adjacency expansion, and ACL-aware filtering.

→ **[Full reference](zen_query_jinja.md)**

---

### `zenos_cabinets.jinja` — Cabinet Macro Library

The canonical safe-read layer for all ZenOS cabinet I/O in Jinja. Encapsulates the FG-38 two-round normalization pattern for drawer access — every template that reads a cabinet drawer imports this library.

Key macros:

* `cabinet_drawer_value(entity_id, drawer_key, fallback)` — extracts the `.value` field from a named drawer, FG-38 safe
* `cabinet_drawer(entity_id, drawer_key, fallback)` — full drawer dict including `value`, `timestamp`, `meta`
* `cabinet_variables(entity_id, fallback)` — full `variables` attribute dict, safe getter
* `cabinet_volume_info / cabinet_guid / cabinet_acls` — VolumeInfo metadata helpers
* `volume_drawer_value(volume, drawer_key, fallback)` — value extraction on a pre-fetched dict (use inside loops)
* `cabinet_guid_new_v4()` — RFC 4122 UUID v4 generator

**Used by:** `zen_os_1.jinja`, `zenos_health.jinja`, `dojotools_filecabinet`, `dojotools_identity`, and all other templates that read cabinet drawers.

→ **[Full reference](zenos_cabinets_jinja.md)**

---

### `zen_target_resolve.jinja` — Shared DojoTools Targeting Core (Zammad #10308)

Common core for DojoTools setter/getter target resolution — entity/label/room resolution was hand-rolled per tool before this file existed. `zen_dojotools_locks` is the first adopter and reference implementation; `zen_dojotools_covers` and every other setter/getter still hand-roll their own.

Key macros:

* `resolve_room_entities(room, domain_pattern)` — every entity in a room, RM-aware: unions native HA area assignment with Room Manager's own label-based room tagging, not area alone. Fixes a real gap the pre-existing per-tool implementations had — a helper labeled into a room without native area assignment (common for timers/input_selects, RM's own convention) was invisible to room-targeting before this.
* `resolve_targets(entity_id, room, label, domain_pattern)` — unified precedence resolver for the three addressing modes (entity_id > label+room > label > room)
* `entity_extended_notes(entity_id, exclude_labels)` — surfaces any label carrying a real `label_description()` as agent-facing guidance, e.g. a lock labeled with an instruction to always confirm out loud before unlocking
* `discover_targets(...)` — bundles resolve+enrich into one call for any tool's discover mode

All macros return JSON strings, same convention as `zenos_cabinets.jinja` — callers do `| from_json` on the result.

**Two real bugs found building this, both generalizable to any macro-backed resolver:**
1. HA's NativeEnvironment auto-coerces a template output of `"[]"` to a real Python empty list before `variables:` sees it — `| from_json` on that crashes with `invalid input '[]'`. Guard every crossing: `(_raw | from_json) if _raw is string else _raw`.
2. `selector: area:` crashes MCP calls with a bare `list index out of range` whenever the caller's string doesn't cleanly fuzzy-match an existing HA area — confirmed it never even reaches the script's `sequence:`. Any field meant to accept both a native area and an RM label must use `selector: text:` instead and let the macro's own resolution validate it.

**Used by:** `zen_dojotools_locks`

---

## Other Files in `custom_templates/zenos_ai/`

| File | Purpose |
|---|---|
| `conversation_agent_prompt_template.yaml` | Paste this into your conversation agent's system prompt in HA. Three lines — imports `zen_os_1.jinja` and calls `render_prompt()`. |
| `command_interpreter.jinja` | Library v1 command dispatch engine. Routes `~COMMANDS~` syntax to subsystem handlers. **Retiring at GA** — individual commands are migrating to index-supported constructs. No new commands should be added here. |
| `library_index.jinja` | Library index — registered command domains and their handlers. |
| `zenos_health.jinja` | Health sensor macro library — `required_labels()`, `slots_all()`, `slot_to_label()`, cabinet state helpers, `is_warmup()`. Imports `zenos_cabinets.jinja` for drawer reads. |
| `flynn_onboarding.jinja` | Flynn onboarding macros including `active_notification()` — surfaces highest-priority Flynn persistent notification into the prompt. |

---

## How Templates Load

HA loads all files under `custom_templates/` at startup. Import them in YAML or other Jinja with:

```jinja
{%- import 'zenos_ai/zen_os_1.jinja' as zen -%}
{{ zen.render_prompt(states('input_text.zenos_persona_name')) }}
```

Templates are **not** hot-reloaded on file change — restart HA or trigger a template reload after edits.

---

→ **[Documentation Hub](../readme.md)**
→ **[zen_os_1.jinja full reference](zen_os1_jinja.md)**
