# Zen DojoTools Summarizers — v5.1.0

*Ninja Summarizer + SuperSummary — the KF4 action pipeline — MCP-exposed*

---

## Overview

The summarizer package is the KF4 action pipeline. It runs continuously on a schedule, reads each component's context via HyperIndex, writes structured kata to the Kata cabinet, and synthesizes that knowledge into the `zen_summary` drawer that loads Friday's prompt context. Where a component's kata calls for action — a notification, an event, a dispatch — the pipeline fires it.

Two scripts ship in this package:

| Script | Role |
|---|---|
| `zen_dojotools_ninja_summarizer` | Per-component kata writer — reads one KFC component's context and writes its kata drawer |
| `zen_dojotools_supersummary` | Whole-home synthesizer — reads all active kata drawers and writes `zen_summary` |

**Pipeline order:** Ninja Summarizer → per-component kata drawers → SuperSummary → `zen_summary` → Friday's prompt

Both scripts are **MCP-exposed** for on-demand runs. The Scheduler drives them automatically.

```mermaid
flowchart LR
  Scheduler["Scheduler / force event"]
  KFC["Dojo KFC drawer"]
  Seed{"seed or area_seed?"}
  SeedTool["Whitelisted seed tool"]
  Index["Index / HyperIndex"]
  Monk["ai_task.generate_data"]
  FileCabinet["FileCabinet write"]
  Kata["Kata drawer"]
  SuperSummary["SuperSummary"]
  Prompt["Prompt context"]

  Scheduler --> KFC --> Seed
  Seed -- "Yes" --> SeedTool --> Monk
  Seed -- "No" --> Index --> Monk
  Monk --> FileCabinet --> Kata --> SuperSummary --> Prompt
```

---

## Kill Switches

Three `input_boolean` entities control the pipeline. Fresh installs default the master switch **off** so a new user does not accidentally run continuous background inference before choosing a local/background AI task.

| Entity | Default | Purpose |
|---|---|---|
| `input_boolean.zen_summarizers_enabled` | off on fresh install | Master gate — turns off both summarizers immediately |
| `input_boolean.zen_ninja_summarizer_enabled` | on | Ninja Summarizer individual kill switch |
| `input_boolean.zen_supersummarizer_enabled` | on | SuperSummary individual kill switch |
| `input_boolean.zen_action_emission_enabled` | off | Allows Ninja to emit `suggested_act_event` kinds onto the event bus. Operator-only — AI cannot write this boolean. `zen_summarizer_act_whitelist` is a separate per-kind gate. |

Master is checked first. If the master is off, both summarizers exit regardless of their individual switches. Turning any switch off is non-destructive — no schedules, automations, or cabinet data are touched.

**Auto-refire on re-enable:** `automation.zen_pipeline_autofire_on_enable` fires the appropriate force event within seconds of any switch being turned back on — no waiting for the next scheduled run. See [Scheduler readme](zen_dojotools_scheduler_readme.md) for trigger IDs.

> **Warning:** Do not point `input_text.zenos_ai_task_entity` at a paid inference API. The Ninja Summarizer fires multiple times per hour; the SuperSummary fires a minimum of 4 times per hour. Use a locally-hosted model for background summarization. Your frontline conversation agent (demand-only) does not carry this risk.

---

## Ninja Summarizer

**Script:** `zen_dojotools_ninja_summarizer`

Summarizes a single Kung Fu Component. Called by the Scheduler for each component that subscribes to the current trigger ID.

### Input Fields

| Field | Required | Description |
|---|---|---|
| `kung_fu_component_id` | Yes | KFC component ID (will be slugified). Matches the drawer key in the Dojo cabinet. |
| `query` | No | Override the default summarization query. Use with caution. |
| `post_to_kata_cabinet` | No | Write result to Kata cabinet? Default: `false`. |
| `supplemental_prompt` | No | Extra data or instructions appended to the monk prompt. |
| `force` | No | Bypass the run governor (dedup burnout window). For admin overrides or emergency on-demand runs. Default `false`. |
| `area_id` | No | HA area ID. When set and the KFC defines `area_seed`, the `{{area_id}}` slot in `area_seed.params` is filled at runtime. Used for per-area rollup patterns. |
| `index_call_override` | No | Override the component's configured index call with a custom dict DSL for this single run. Does not persist. |
| `parent_component_id` | No | Component slug to treat as the logical parent of this run (for compound/nested component patterns). Echoed in the response for correlation. |
| `staleness_minutes` | No | If the component's last kata drawer is newer than this value, skip the monk call and return the cached kata. Sets `expires_after` on the kata write. Default: uses `staleness_minutes` from the KFC drawer if present, else no staleness check. |
| `caller_token` | No | Opaque pass-through token for correlation. |

### What It Does

1. **Kill switch check** — exits immediately if master or ninja switch is off
2. **AI task guard** — exits with error if `input_text.zenos_ai_task_entity` is unset or unavailable
3. **Resolve cabinets** — reads Dojo, Kata, and household cabinet entity IDs from resolver sensors
4. **Read Dojo drawer** — loads the component's KFC metadata (friendly name, label, command, tool, kata_key)
5. **`meta.enabled` check** — exits with `reason: meta_disabled` if the component's `meta.enabled` is `false`
6. **Run governor** — dedup burnout window check (see below). Exits with `reason: dedup_window` if blocked.
7. **Step 3c — Seed tool call (v4.3.0+)** — if the KFC defines `seed` or `area_seed`:
   - Resolves `_seed_tool`: `area_seed` takes priority when `area_id` input is set; falls back to `seed`.
   - **Whitelist check**: reads `zen_summarizer_seed_whitelist` from syscab (`allowed_tools` list). Default if drawer missing: `['zen_dojotools_index']`.
     - Whitelisted → fires `script.{{ _seed_tool }}`, sets `_seed_used: true`, step 8 skipped.
     - Not whitelisted → emits `seed_tool_blocked` zen_event (warn), falls through to step 8 (HyperIndex runs normally).
   - If neither `seed` nor `area_seed` is defined: step 3c is skipped, step 8 runs normally.
7.5. **Step 3d — Resolve component_summary from label description (v4.6.0+)** — if `component_summary` is empty after reading the Dojo drawer (Scribe's `trim_description` path leaves it blank), the summarizer reads the base label description via `zen_dojotools_labels` and uses it as `component_summary`. This means the label description is the authoritative routing signal when the drawer field is trimmed — keep label descriptions accurate.
8. **Run HyperIndex** — queries the index using the component's configured index call. Skipped if `_seed_used`. Routing:
   - If the Dojo drawer has an `index_command` dict field: emits compound/recursive index call via `zen_indexer_request` event. Supports the full nested DSL: `{operator, index_1: {...}, index_2: {...}}`. Use for components whose context spans multiple independent label sets.
   - If the drawer has a `label` field: standard label-based index call in hypergraph mode.
   - If neither is present: HyperIndex step is skipped.
9. **Run library command** — if the component has a `command` field, dispatches it through `command_interpreter.jinja`
10. **Build monk prompt** — assembles query, kata template (structure), example, review data (index + library output + dojo drawer), and supplemental instructions
11. **Call ai_task.generate_data** — sends prompt to the configured AI task entity (the local LLM monk)
12. **Post to Kata cabinet** — if `post_to_kata_cabinet` is true and monk returned data, writes result to `kata_cabinet[component_slug]`
13. **Emit event** — fires `zen_dojotools_event_emitter` with kata/monk excerpt fields

### Response

```json
{
  "status": "success",
  "monk": { "data": { ... } },
  "post": false,
  "dojo_drawer": { ... },
  "filecabinet_write": {},
  "caller_token": ""
}
```

---

## Ninja Run Governor

The run governor prevents duplicate runs within a configurable burnout window. It fires between the `meta.enabled` check and the HyperIndex call.

**How it works:**

- Reads `burnout_seconds` from the `zen_ninja_config` drawer in the household cabinet. Default: **300 seconds (5 min)**.
- Reads `zen_ninja_last_run_<component_slug>` from the Kata cabinet — a timestamp drawer written at each successful run.
- If elapsed time since last run is less than `burnout_seconds`, the script stops and fires a `summarizer_run_blocked` event with `reason: dedup_window`.
- Set `force: true` to bypass the governor for admin overrides or emergency on-demand runs.

**Why it exists:** The Scheduler can fire multiple triggers in a short window (e.g., `ha_start` + `force_summary` near-simultaneously, or rapid home mode changes). Without the governor, a component could run 4–5 times in under a minute, burning through LLM capacity and producing stale-input summaries before state settles.

**Configuring `burnout_seconds`:** Write to the `zen_ninja_config` drawer in the household cabinet via FileCabinet:
```yaml
action: script.zen_dojotools_filecabinet
data:
  action_type: update
  volume_entity_id: sensor.<household_cabinet>
  key: zen_ninja_config
  value:
    burnout_seconds: 180
```

**Event on block:**
```json
{
  "kind": "summarizer_run_blocked",
  "component": "<component_slug>",
  "reason": "dedup_window",
  "elapsed_secs": 42,
  "burnout_secs": 300,
  "severity": "info"
}
```

---

## SuperSummary

**Script:** `zen_dojotools_supersummary`

Synthesizes all active component kata drawers into a single `zen_summary` — the canonical home state that loads into Friday's prompt every session.

### Input Fields

| Field | Required | Description |
|---|---|---|
| `query` | No | Override the default summarization query. |
| `post_to_kata_cabinet` | No | Write `zen_summary` to Kata cabinet? Default: `false`. |
| `supplemental_prompt` | No | Extra instructions appended to the prompt. |
| `force` | No | Bypass the `super_burnout_seconds` cooldown governor. Default `false`. |
| `caller_token` | No | Opaque pass-through token for correlation. |

### What It Does

1. **Kill switch check** — exits immediately if master or supersummarizer switch is off
2. **AI task guard** — exits with error if `input_text.zenos_ai_task_entity` is unset
3. **Resolve cabinets** — reads Dojo and Kata cabinet entity IDs from resolver sensors
4. **Load schema** — reads `zen_template.value.structure` from Kata cabinet (the output schema the monk must fill)
5. **Build prompt context** — assembles identity card, manifest, capsule, home overview, and last `zen_summary`
6. **Collect active components** — iterates all Dojo drawers, selects those where `meta.enabled` is true (or absent) and whose `kata_key` exists in the Kata cabinet
7. **Load per-component kata values** — reads each active component's kata drawer value
8. **Build monk prompt** — assembles query, schema (zen_template structure), example, and review data (all component kata values + prompt context)
9. **Call ai_task.generate_data** — sends prompt to the AI task entity
10. **Write `zen_supersummary_status`** — records run status + component count to Kata cabinet
11. **Post `zen_summary`** — if `post_to_kata_cabinet` is true and monk returned data, writes to Kata cabinet as `ZEN_SUMMARY`
12. **Emit event** — fires `zen_dojotools_event_emitter`

### Pipeline Tier Split (v5.1.0)

v5.1.0 separates components into three pipeline tiers. Tier is set by the `pipeline_tier` field in the KFC Dojo drawer:

| Tier | Values | How SuperSummary handles it |
|------|--------|----------------------------|
| `direct` | default when field absent | Full `component_data` included in the monk prompt |
| `ambient` | `ambient` | Excluded from direct `component_data`. Pre-digested by **Trapper Keeper** into a compact breadcrumb + navigation index. Urgency ≥ 4 promotes the component to `active_components`. |
| `system` | `system` | Excluded from direct `component_data`. Summary provided via a separate system_summary block. |

**Ambient urgency promotion rule:**
- Urgency 0–3 → ambient_nav breadcrumb only
- Urgency 4–5 → also added to `attention` list
- Urgency 6–10 → promoted to `active_components` (same as direct tier)

**Trapper Keeper** is SuperSummary's ambient pre-processor. It reads all ambient-tier component kata drawers, extracts a compact breadcrumb (component name, urgency, one-line summary), and builds a navigation index of which ambient components are available and how to reach them via a targeted Ninja run. The Trapper Keeper output is injected into the SuperSummary monk prompt as a navigation aid rather than full data.

### Active Component Selection

A component is included in SuperSummary if:
- Its Dojo drawer has a `kata_key` field (non-empty)
- `meta.enabled` is `true` or absent (switchless KFCs are always included)
- Its `kata_key` exists as a drawer in the Kata cabinet

Components with `meta.enabled: false` are excluded. Components with no `meta` block are included.

### Response

```json
{
  "active_component_ids": ["security_manager", "media_manager"],
  "monk": { "data": { ... } },
  "post": true,
  "filecabinet_write": { "status": "ok" },
  "caller_token": ""
}
```

---

## SuperSummary Run Governor

SuperSummary has its own cooldown window separate from the Ninja governor. Controlled by `super_burnout_seconds` in the `zen_ninja_config` drawer in the household cabinet. Default: **600 seconds (10 min)**.

If SuperSummary ran less than `super_burnout_seconds` ago, the call returns immediately with `reason: dedup_window`. Set `force: true` to bypass.

## Size Guard and Context Budget

SuperSummary enforces two guards before the monk call:

- **Size guard:** if the assembled prompt exceeds **200,000 bytes**, SuperSummary aborts with `status: error, reason: prompt_size_exceeded`. Reduce component count or trim kata drawer values.
- **Context budget:** `max_context_tokens` (from `zen_ninja_config`, default **28,000**) caps the total token estimate. If the assembled prompt would exceed the budget, ambient/system-tier components are dropped before direct-tier components.

## Forcing Runs

Use Scheduler force events to trigger on demand. See [Scheduler readme](zen_dojotools_scheduler_readme.md) for full syntax.

| Event kind | What runs |
|---|---|
| `summary_force` | Both Ninja dispatch + SuperSummary (delay skipped) |
| `ninja_force` | Ninja dispatch only (ha_start scope, delay skipped) |
| `supersummary_force` | SuperSummary only |

---

## Scheduler Integration

The Scheduler drives the summarizers automatically. Default trigger wiring:

| Trigger | What fires |
|---|---|
| `every_10_minutes` | SuperSummary |
| `quarter_hour` | Ninja dispatch (per component subscriber) |
| `ha_start` + `daily_midnight` | Both |
| `force_summary` | Both (delay skipped) |

Components subscribe to triggers via `trigger_subscriptions` in their Dojo drawer. See [Scheduler readme](zen_dojotools_scheduler_readme.md) and [KFC docs](../../kung_fu/readme.md).

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Summaries stale | `sensor.zen_supersummary_health` → `monk_status` |
| Ninja not firing | `sensor.zen_summarizer_health` → `last_timestamp` and `ai_task_entity` |
| Kill switch `disabled` state | One of the three kill switches is off — intentional. Turn back on; pipeline auto-restarts. |
| Empty or malformed output | Check inference server logs for context length errors. Models under ~4B parameters are not reliable as summarization backends. |
| Component not included in SuperSummary | Check `meta.enabled` in its Dojo drawer. Also verify `kata_key` is set and a matching drawer exists in Kata cabinet. |
| Seed always falls through to HyperIndex / `seed_tool_blocked` events | `zen_summarizer_seed_whitelist` drawer is missing from syscab, or the tool is not in `allowed_tools`. Run `zen_admintools_reset_template` to seed the default whitelist, then add tools via `zen_admintools_summarizer_seed action_type: add tool: <script_name>`. |

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `sensor.zen_dojo_cabinet_resolved` | Canonical Dojo cabinet entity |
| `sensor.zen_kata_cabinet_resolved` | Canonical Kata cabinet entity |
| `sensor.zen_default_household_cabinet_resolved` | Household cabinet (Ninja — household prefs) |
| `input_text.zenos_ai_task_entity` | LLM monk entity for ai_task.generate_data |
| `input_boolean.zen_summarizers_enabled` | Master kill switch |
| `input_boolean.zen_ninja_summarizer_enabled` | Ninja kill switch |
| `input_boolean.zen_supersummarizer_enabled` | SuperSummary kill switch |
| `script.zen_dojotools_filecabinet` | Kata cabinet writes |
| `script.zen_dojotools_index` | HyperIndex queries (Ninja) |
| `command_interpreter.jinja` | Library command dispatch (Ninja) |
| `zen_os_1.jinja` | `zen_cabinets()`, `manifest_loader()`, `ai_capsule()` (SuperSummary prompt context) |
| `script.zen_dojotools_event_emitter` | Post-run event emission |
| `automation.zen_dojotools_scheduler` | Scheduled dispatch |

---

## Cross-References

- [Zen Summarizer Overview](../zen_summarizer/readme.md) — conceptual pipeline from Dojo to prompt
- [DojoTools Scheduler](zen_dojotools_scheduler_readme.md) — trigger IDs, component subscriptions, and force events
- [DojoTools Index](zen_dojotools_index_readme.md) — default Ninja context resolver
- [HyperIndex Overview](../zen_hyperindex/zen_hyperindex_overview.md) — SELECT -> FILTER -> COMPOSE graph model
- [DojoTools FileCabinet](zen_dojotools_filecabinet_readme.md) — Kata and status drawer writes
- [Script Modules](readme.md) — return path to the internal tool map

---

## Version History

| Version | Change |
|---------|--------|
| v5.1.0 | Pipeline tier split: `direct`/`ambient`/`system` tiers via `pipeline_tier` KFC field. Trapper Keeper pre-digests ambient-tier components (breadcrumb + navigation index). Ambient urgency promotion rule (0–3=breadcrumb only; 4–5=attention; 6–10=active_components). SuperSummary run governor (`super_burnout_seconds`, default 600s, `force` bypass). Size guard (>200K bytes → abort). Context budget guard (`max_context_tokens`, default 28K). Ninja: `index_call_override`, `parent_component_id`, `staleness_minutes` input fields. `response_variable: result` on all stop steps. Namespace scoping fix on `active_components` iteration. `ZEN_SUMMARY` key case fix. Event→action migration. `library_console` rename. Kata cabinet variable fix. |
| v4.6.0 | Step 3d — label description resolution: when `component_summary` is empty after reading the Dojo drawer (Scribe `trim_description` path), ninja resolves it from the base label description via `zen_dojotools_labels`. `zen_action_emission_enabled` boolean added (operator-only gate for `suggested_act_event` emission). `emission_cooldown_minutes` Dojo drawer field (default 60 min) gates per-component action event emission; emits `emission_suppressed` on cooldown. FG-38 `from_json` guards on all FileCabinet drawer reads. |
| v4.3.0 | Dual-seed architecture: new step 3c, `area_id` input field, `_seed_used` gate on HyperIndex, seed whitelist gate (`zen_summarizer_seed_whitelist`). Backward compatible — no seed = old behavior. |
