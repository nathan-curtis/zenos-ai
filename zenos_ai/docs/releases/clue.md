# Release Notes — 2026.6.0 'Clue'

**Released:** 2026-05-17
**Branch:** `feat/2026.6.0`
**Base:** 2026.5.0 'Fry's Grandpa'
**UAT:** Nyx (H:\)

---

*The AI knows which room. Every room. And what's connected to it.*

---

## Summary

Before this release, ZenOS-AI knew your home the way a stranger knows a city — by reading about it. It could see your entities, query your integrations, and reason about what was happening. But if you asked "what's adjacent to the living room?" or "which door leads outside?" or "where's the fire extinguisher?" — it was guessing. There was no spatial ground truth.

Clue closes that. The Room Manager gives the system a first-class spatial model: rooms with portals between them, adjacency derived from connections, exits ranked by priority, safety equipment cataloged, emergency routing built in. The home is no longer a flat list of entities. It's a place.

That model ripples outward. `home_overview` now carries a weather snapshot, a live plant summary, and the current home mode — the AI has a full situational picture before it types the first word of a response. The emergency snapshot knows which rooms are primary shelter and which have exterior portals. The lighting engine knows what bleeds through an open archway. The security manager knows which cameras cover which rooms.

Then there's Plant Manager. The home finally has a nervous system readout: live watts, billing accumulation, water flow, gas consumption, water heater status, circuit-level breakdown. All of it label-first, all of it graceful when a sensor is missing.

And then the rest: Media Manager for whole-home audio/video. ZenShade for covers. Security Manager replacing the old alarm panel stub. Calendar and Todo as standalone tools. Cortex v39 with the Home First directive — the AI leads with what's happening in the home, not with what you just asked. Office split into its component tools. The calderaspas plugin is gone and SpaMaster is the replacement.

Six new tools. Twenty-plus updated. One release that changes what the AI knows it's standing in.

Before beta shipped, the foundation got swept too. FileCabinet normalization touched every tool in the stack — one writer, one reader, one normalized struct. The Ninja Summarizer gained a dual-seed path so KFCs can pull from Room Manager or any tool directly. AlertManager grew a face: Friday can now list, fire, and clear alerts as a first-class agent action. The engine underneath changed. The spatial model stays the headline.

In Clue, the whole game is figuring out which room. We're done figuring.

---

## Room Manager (RoomReg) — New (v1.42.0)

The spatial intelligence hub. Everything room-aware in ZenOS-AI reads from here.

### What it stores

Room topology lives in the household cabinet under `room_topology` — a dict keyed by HA area_id. Each room entry holds:

- `portals[]` — passable connections (door/archway/passage), with `sound_tx`, `light_tx`, `exterior`, `exit`, `emerg_exit`, `exit_priority`
- `windows[]` — windows with direction, type, `drop_ft`, `emerg_exit`
- `adjacent[]` — auto-derived from portals on every `mode=link` call; never build by hand
- `boundary_links[]` — non-passable shared boundaries (walls, floor/ceiling) for acoustic and thermal reasoning
- `safety[]` — fire extinguishers, first aid kits, AEDs, hazmat items by room
- `description`, `area_note`, `walls`, `exterior`, `shelter_eligible`, `grocy_location_id`, `rally_point`

### Key modes

| Mode | What it does |
|------|-------------|
| `get` | Full room brief: topology + live HA state. Add `context_slices=+light,+climate,+media,+topo,+inventory,+tasks,+calendar,+security` |
| `set` | Write or patch room topology. Auto-applies `room_layout` label to the HA area. |
| `link` | Bidirectional adjacency + interior portal. `adjacent[]` derives from this — never write it manually. |
| `area_create` | Create a new HA area, apply `room_layout`, and init topology in one call. |
| `home_overview` | Whole-house snapshot: signal{}, alerts{}, domain_summaries{}, floors[], weather{}, plant{}, home_mode, utility_index{} |
| `emergency` | Crisis snapshot: guidance, exits[], shelter{primary/secondary/avoid}, safety_equipment[], hazards[], likely_occupied[], location, rally_point. Pass `area=` for room-scoped `from_room{}` block. |
| `utility` | Manage utility_index: provider names, emergency contacts, service entry locations. |

### home_overview additions (v1.39.0–v1.42.0)

`home_overview` gained three new top-level fields:

- `home_mode` — current ZenOS home mode string (`sensor.zen_home_mode` or `input_select.zen_home_mode`)
- `weather{}` — condition, temperature, humidity, wind_speed, next_hour slot, sourced from `zen_weather` label or first available `weather.*`
- `plant{}` — live_kw, water_gal, and a pointer to zen_dojotools_plant for depth

`utility_index{}` is also injected from the household cabinet on every call.

### zen_rm_ignore label

Tag any entity with `zen_rm_ignore` to exclude it from all `home_overview` entity discovery passes — battery scan, alarm panel discovery, plant snapshot. For entities that exist but should not pollute whole-home rollups.

### Setup

```
zen_dojotools_room_manager  mode=setup  confirm_action=true
zen_dojotools_room_manager  mode=area_create  area_name="Kitchen"  description="Main cooking area"
zen_dojotools_room_manager  mode=set  area=kitchen  description="Open to living room"
zen_dojotools_room_manager  mode=link  area=kitchen  area_b=living_room  portal_type=archway
```

See [Room Manager reference](../room_manager.md) for the full schema.

---

## Plant Manager — New (v1.2.2)

Physical plant + energy manager. Every utility in the home, surfaced in one tool.

### Discovery model

Label-first. `zen_plant_*` override labels take first priority for every slot. Standard semantic labels (`utility_main`, `utility_meter`, `consumed_energy`, `water_usage`) are the primary discovery path. Integration-specific labels (`utility_billing` for billing/tariff sensors, `droplet` for Droplet water meters) are secondary fallbacks.

`zen_plant_ignore` suppresses any entity from all waterfalls.

### Modes

| Mode | Returns |
|------|---------|
| `overview` | All utilities in one call: electric, hvac, water, gas, mechanical |
| `electric` | Live watts, daily/weekly/monthly kWh, tariff (USD/kWh), grid fossil %, panel status |
| `hvac` | Climate units: mode, setpoint, current temp |
| `water` | Usage (gallons), billing rate, usage_since timestamp |
| `gas` | Live therms if `zen_plant_gas` labeled, else graceful N/A |
| `mechanical` | Water heater (mode, temps, hot water %, live W, daily kWh) + sump pump |
| `circuits` | Top circuits by lifetime Wh or live amps. `circuit_limit` (default 10), `sort_by=energy\|current` |
| `validate` | 15-slot resolution report — entity_id, pinned, raw_state, ok. Run this after setup. |

Apply `zen_plant_water_rate` or tag a `water_usage` sensor with a name matching `fee|rate|cost` to wire the water rate slot.

See [Plant Manager reference](../plant_manager.md) for the full discovery waterfall and slot reference.

---

## Media Manager (NyxMau5) — New (v0.7.2)

Whole-home media management. Discovers all `media_player.*` entities, resolves source lists, handles intent routing (play/pause/stop/volume/source), and applies user preference profiles.

FG-05 fixed at launch: `label_entities('media_player') | select('match', 'media_player\\.') | list` + `states.media_player` fallback. No unbounded domain scan.

See [Media Manager reference](../media_manager.md).

---

## Security Manager — New (v1.2.0)

Replaces `zen_dojotools_alarm_panel`. Renamed to match manager convention.

Modes: `read_state`, `query`, `arm`, `disarm`, `get_policy`, `set_alert_policy`. Discovery via `label_entities('alarm_panel')`.

`read_state` adds two new fields:
- `cameras_by_area` — `security_camera`-labeled cameras grouped by area_id (from RM `+security` slice)
- `lens` — cross-reference noting Security Manager and Room Manager +security as complementary views of the same state

Per-room zone inventory via RM `+security` context slice.

**Entity registry note:** If `zen_dojotools_security_manager` shows `unavailable` after reload, HA may be holding a stale entry for the old `zen_dojotools_alarm_panel` entity. Go to Settings → Devices & Services → Entities, find the old entry, delete it, reload scripts.

---

## ZenShade (Covers) — New (v0.2.2)

Cover management with tilt support and ZenLux sync.

- Tilt control: bit-128 precision (0=closed, 128=privacy at 45°, 255=open). `privacy` mode sets 45°.
- Barrier exclusion: garage doors, entry doors, and gate covers are automatically excluded from scene targets
- `sync_shades` mode: ZenLux calls this to synchronize cover positions with lighting scenes
- Scene modes: `open`, `close`, `privacy`, `blackout`, `sync`

See [ZenShade reference](../zenshade.md).

---

## OOBE — v4.2.0

Room setup is now RM-native. The old pattern (one filecabinet drawer per room) is gone.

### What changed

The AI now builds a full Room Manager topology during OOBE — not a flat profile list. For each room:

1. Calls `mode=area_create` for rooms not yet in HA (creates area + applies `room_layout` in one call)
2. Calls `mode=set` to register existing rooms (auto-applies `room_layout` — no manual HA UI step)
3. Calls `mode=link` for each connection — `adjacent[]` derives from portals automatically
4. Collects exterior exits, emergency exits, rally point, and safety equipment
5. Closes with `mode=home_overview` spatial sanity check before sealing

New close-out step: the AI instructs the user to set the **ZenOS: Persona** helper (`input_text.zenos_persona_name`) to the agent name just configured, then start a fresh conversation for persona handoff.

Integration mapping: cameras now get `security_camera` label (not just `camera`) — required for RM +security slice and Security Manager `cameras_by_area`.

---

## Cortex v39 — Home First

Supersedes v38 (Kata First).

The flagship behavioral change: the AI now leads with what's happening in the home before answering a question. HOME OVERVIEW is surfaced in the context frame. Cruise director calibration aligns proactive guidance with the operator's actual home state rather than with the last kata.

Load: `zen_admintools_prompt_loader: cortex_version: latest`

---

## AlertManager — v1.3.0

- **Severity labels** — alerts now carry a severity tag in the label namespace. Alert discovery queries include severity filtering.
- **Fire-once dedup** — `_zen_active_alerts` drawer tracks active alert keys. Duplicate fires for the same key are suppressed until cleared.
- **`clear_after_minutes` default** — `alert_fire` now defaults to **1440 minutes (24h)** if `clear_after_minutes` is not passed. Pass `clear_after_minutes: 0` for permanent / manual-clear-only alerts. `expires_at` is stamped on every entry.
- **GC sweep** — DojoTools Core step 4c sweeps `_zen_active_alerts` on every GC cycle. Fires `alert_clear` for entries whose `expires_at` has passed.

---

## AlertManager Tool — v1.0.1 (New)

`zen_dojotools_alertmanager` — new MCP-facing CRUD script. Friday can now manage the alert system directly: query active alerts, fire, clear, and read or set notify policy.

This is the agent-accessible complement to the automation-side AlertManager. The automation handles event routing; this tool handles inspection and programmatic control.

### Modes

| Mode | What It Does |
|------|-------------|
| `list` | Return all active alerts from `_zen_active_alerts` with fire times, severity, and TTL remaining. |
| `fire` | Fire an alert by key. Queues the event and returns. Respects dedup — no-op if already active. |
| `clear` | Clear a specific alert by key. Queues clear event. |
| `clear_all` | Clear all active alerts. Returns count cleared. |
| `get_policy` | Read the current notify policy — filter entity, base urgency, per-severity routing. |
| `set_policy` | Write a new notify policy to the household cabinet. |
| `help` | Full tool contract, mode reference, and field list. |

### Key Inputs

| Field | Required For | Purpose |
|-------|-------------|---------|
| `mode` | — | Operation mode. Default: `help`. |
| `alert_key` | `fire`, `clear` | Alert dedup key — must match exactly. |
| `message` | `fire` | Human-readable alert text. |
| `severity` | `fire` | `info` \| `warn` \| `error`. Default: `warn`. |
| `notify_target` | `fire` | `persistent` \| `postman` \| `notify.<service>`. Default: `persistent`. |
| `clear_after_minutes` | `fire` | TTL in minutes. Default: 1440 (24h). Pass `0` for permanent. |
| `provider_id` | `set_policy` | Policy scope key. |
| `urgency` | `set_policy` | Urgency level for this policy entry. |

**`fire` and `clear` queue events** — alert state change happens asynchronously via the AlertManager automation. Call `list` after a brief wait to confirm.

**HA restart required on first install.** `zen_dojotools_alertmanager` is a new script entity. Script reload alone is not enough — it will not appear in the MCP schema until HA is fully restarted once.

---

## Ninja Summarizer — v4.3.0 — Dual-Seed Architecture

KFCs can now define their own context source, bypassing HyperIndex when a richer tool gives better data.

Before: every component drove context acquisition through the same path — label-based HyperIndex (step 4). Works well for entity-centric components. Falls short when the right context source is a tool, not a label query.

After: new **step 3c**. A KFC can define `seed` (concept-first) or `area_seed` (location-first). When either is present, step 3c fires the configured tool with configured params, step 4 (HyperIndex) is skipped via `_seed_used: true`, and the monk receives the tool's richer output directly.

### `seed` — concept-first

```json
{
  "seed": {
    "tool": "zen_dojotools_room_manager",
    "params": {"mode": "get", "area": "garage", "context_slices": "+topo,+security"}
  }
}
```

Fixed params, fires once per run. Use when the context source is domain-specific and doesn't change per invocation.

### `area_seed` — location-first

```json
{
  "area_seed": {
    "tool": "zen_dojotools_room_manager",
    "params": {"mode": "get", "area": "{{area_id}}", "context_slices": "+topo,+light,+climate"}
  }
}
```

`{{area_id}}` is filled at runtime from the Ninja Summarizer's `area_id` input field. One KFC definition, N room-scoped runs. This is the **per-area rollup pattern**: parent KFC rolls up whole-home, child KFCs fire per-area with `area_seed`.

### Whitelist gate

Step 3c checks `zen_summarizer_seed_whitelist` in syscab before firing any seed tool.

- Shape: `{allowed_tools: ["zen_dojotools_index", ...], note: "..."}`
- **`zen_dojotools_index` is whitelisted by default** — existing KFCs unaffected.
- Unwhitelisted tool → emits `seed_tool_blocked` zen_event (warn) → falls through to HyperIndex. Summarizer still runs; operator gets visibility via the event.
- To add Room Manager: `zen_admintools_summarizer_seed action_type: add tool: zen_dojotools_room_manager` (not MCP-exposed — run via HA script action).

**If the summarizer pipeline is running but seeds are always falling through to HyperIndex, force-run `zen_admintools_reset_template` to seed the whitelist drawer into syscab, then add your tools.**

### Backward compatibility

No `seed` and no `area_seed` → step 3c is skipped, step 4 runs as before. All existing KFCs continue unchanged.

---

## FileCabinet Normalization — Global Architecture

Every tool in ZenOS-AI now flows through two canonical I/O paths:

- **Scripts and automations** — all cabinet reads and writes go through `zen_dojotools_filecabinet`
- **Templates** — all cabinet access goes through `zenos_cabinets.jinja` CABS macros

And all cabinet writes now produce a normalized struct:

```json
{
  "value": <payload>,
  "timestamp": "2026-05-18T10:00:00",
  "meta": {}
}
```

`timestamp` is always written. You can suppress it, but the struct is preserved. `meta` is reserved for future use (ACL tagging, versioning, drawer-level metadata).

The old pattern: most tools wrote values directly as raw JSON. Some wrote strings. Some wrote dicts. Reads guessed. Guards were inconsistent. One writer, one reader, one struct closes that.

### What changed under the hood

- **Read guards** — `x if x is not string else (x | from_json)` applied throughout. Native Python lists returned by HA's template engine are no longer double-parsed. JSON-encoded strings still parse correctly. This fixes FG-38 across the stack.
- **`zenos_cabinets.jinja`** — fallback handles legacy non-structured drawers: `drawer.get('value', drawer if drawer else fallback)`. Pre-normalization drawers degrade gracefully instead of returning None.
- **`zen_os_1.jinja`** — guards on `_slot` and `_alerts` reads.
- **FileCabinet v4.7.0** — `set_timestamp` defaults to `true`. Writes always produce the normalized struct. `_` prefix reads no longer silently strip the underscore when `force_action` is omitted.
- **~21 files touched** — FC normalization sweep across the full DojoTools package.

---

## Other Tool Updates

| Tool | Version | Key Changes |
|------|---------|-------------|
| **Dispatcher** | v1.1.0 | Postman Tier 2, infra escalation hard deny, Covers + Climate Tier 2, security_manager route, spamaster route |
| **ZenLux** | v0.5.1 | sync_shades integration, burnout timer, RM hold gate, media controller awareness |
| **Climate Manager** | v1.1.0 | `topology_context` in GET: open doors/windows, area sensors, RM HVAC bleed portals, natural vent advisory |
| **Grocy** | v4.10.0 | `units_add` idempotent, `update_product_meta` unit support, hazmat/safety RM integration, `chores_by_area` dual discovery |
| **Ectoplasm** | v4.6.1 | `floor_assign`/`unassign` REST path, `area_id` reserved word fix, error surface improvements |
| **Index / ZQ-1** | v4.9.1 | `+rm` pipeline, `area_entities()` fix, `filter_json` fix |
| **ZQ-1 filter engine** (`zen_query.jinja`) | v4.5.7 | `friendly_name_regex` filter — `regex_search()` against `friendly_name` attribute; SEED or FILTER mode. Complements `entity_id_regex` (added v4.6.0). |
| **Labels** | v4.6.0 | `target_areas` support, `add/remove_label_to_area` |
| **Camera** | v1.4.0 | `ai_task` gate for look/scan, `sendto` field expansion, **3h result expiry** for look/scan modes (was 24h). Lens pattern: Security Manager + RM +security are complementary, not competing. |
| **Scribe** | v1.4.0 | `seed` and `area_seed` input fields added. Parsed into draft and publish payloads. Publish preserves existing seed values when inputs blank. Help updated with `context_source_guide`, `per_area_rollup_pattern`, `tuning_guide`. Also: replaces KungFu Writer. |
| **Ninja Summarizer** | v4.3.0 | Dual-seed architecture — new step 3c, `area_id` input field, seed whitelist gate (`zen_summarizer_seed_whitelist` in syscab). **MCP-exposed.** See [Ninja Summarizer section](#ninja-summarizer--v430--dual-seed-architecture) above. |
| **SystemTools** | v4.5.9 | `ha_reload_all` and `ha_reload_scripts` now deferred via `zen_event(kind: deferred_script_reload / deferred_reload_all)`. Closes the WONT FIX asyncio `InvalidStateError` from `__remove_future` cancellation. All four reload modes now config-check gated. Ships with Scheduler v4.5.5 (hard dependency). **MCP-exposed.** |
| **Scheduler** | v4.5.5 | Two new event triggers: `deferred_script_reload` and `deferred_reload_all`. Required companion to SystemTools v4.5.9. Must ship together. Automation-driven — not MCP-exposed. |
| **AdminTools** | v4.6.1 | KFC schema `v1.4.0`: `seed` and `area_seed` fields added to `kfc_template`. `zen_admintools_reset_template` now seeds `zen_summarizer_seed_whitelist` into syscab (Flynn gate-3). New `zen_admintools_summarizer_seed` management script (list/add/remove/reset). **Admin-only — not MCP-exposed.** |
| **FileCabinet** | v4.7.0 | Global normalization — see [FileCabinet Normalization section](#filecabinet-normalization--global-architecture) above. **MCP-exposed.** |
| **SpaMaster** | v3.3.0 | Replaces calderaspas entirely. Generic spa management, ESPHome device discovery, scene/chemistry/log modes, preset library. |
| **Identity** | v4.5.6 | VolumeInfo decode guard, profile autosign, provision_member (from Fry's Grandpa — carried forward) |
| **DojoTools Core** | v4.5.6 | `_zen_active_alerts` TTL sweep (step 4c) |
| **Office** | v5.0.0 | Todo + Calendar removed — now standalone `dojotools_todo.yaml` and `dojotools_calendar.yaml`. Office carries Teams + Mail only. |
| **Flynn** | v4.5.6 | Household membership + identity manifest at bootstrap, warmup timer |
| **Calendar** | v1.11.0 | Split from office.yaml. Standalone HA Calendar domain CRUD. |
| **Todo** | v2.1.0 | Split from office.yaml. `action` alias for `action_type`; `list_id` alias for `list_name` (accepts entity_id or friendly name). Invalid `action_type` now returns help instead of noop. |

---

## HALMark Audit — 2026.6.0

Full audit of all 64 YAML files in `packages/zenos_ai/`.

### Fixed this release

| Finding | File | Fix |
|---------|------|-----|
| FG-07: Hardcoded expansion cabinet IDs | `dojotools_identity.yaml` | `label_entities('expansion_cabinet')` + `range(1,6)` fallback |
| FG-05: `states.sensor` unbounded scan | `dojotools_manifest.yaml` | `label_entities('zenos_cabinet') \| select('match','sensor\\.') \| list` + fallback. Root cause: missing domain filter let non-sensor entities pass the guard, manifest returned empty. |
| FG-05: Battery scan | `dojotools_room_manager.yaml` | `label_entities('battery_sensor') \| select('match','sensor\\.') \| list` + fallback |
| FG-05: `states.media_player` scan | `dojotools_media_manager.yaml` | `label_entities('media_player') \| select('match','media_player\\.') \| list` + fallback |
| FG-05/FG-19: `states.todo` loop | `dojotools_todo.yaml` | Entity ID pool + `state_attr()` reads |
| FG-05: `states.light` loop | `dojotools_lights.yaml` | `area_entities()` pre-scan outside main loop — builds on/unavail lists before iterating |
| FG-07: Hardcoded entity ID in water rate fallback | `dojotools_plant.yaml` | Fallback removed; `zen_plant_water_rate` label is now the correct path |
| FG-07: Integration-specific label name | `dojotools_plant.yaml` | Provider label → `utility_billing` throughout |
| FG-38: Unguarded `from_json` on FileCabinet drawer reads | All 21 affected tools | `x if x is not string else (x | from_json)` guard applied throughout. Native Python list/dict values no longer double-parsed. |
| SystemTools asyncio `InvalidStateError` | `dojotools_systemtools.yaml` + `dojotools_scheduler.yaml` | Deferred event pattern — script fires `zen_event(kind: deferred_script_reload)` and exits. Scheduler automation handles reload from automation context. All four reload modes now config-check gated. Previously documented WONT FIX — closed. |

### WONT FIX — documented

**FG-09: FileCabinet delete gate** — adding `confirm: true` to the delete path would break every automated delete in the OS (GC sweeps, alert_clear, deprovision, cabinetadmin repair). Accepted architectural deviation. The confirm-gate pattern applies to interactive/AI-generated ad-hoc deletes, not structured tool call chains.

### Accepted remaining FG-05 (small-domain patterns)

`states.calendar` in calendar/office, `states.zone` / `states.person` in index, `states.fan` in spa_manager — inherently bounded domains, already guarded with `| list`.

---

## Breaking Changes

**`zen_dojotools_alarm_panel` is gone.** The script entity does not exist. New name: `zen_dojotools_security_manager`. Update any KFCs or automations calling the old name.

**`calderaspas` plugin is gone entirely.** `plugins/calderaspas/` deleted. Old script `zen_dojotools_spa_manager` is gone. New: `zen_dojotools_spamaster`. Update any references. If `zen_dojotools_spamaster` shows `unavailable` after reload, delete the stale entity from Settings → Entities.

**`zen_dojotools_kungfu_writer` retired.** Replaced by `zen_dojotools_scribe`. `zen_dojotools_kungfu_loader` (deploys factory KFCs) is a separate, unrelated tool that ships unchanged — do not confuse the two.

**Office v5.0.0 — Todo + Calendar split.** The scripts still exist and still work, but they now live in `dojotools_todo.yaml` and `dojotools_calendar.yaml` instead of `dojotools_office.yaml`. No call changes required.

**AlertManager `alert_fire` default TTL.** Any fire without `clear_after_minutes` now auto-expires after 24h. Pass `clear_after_minutes: 0` explicitly if the alert must persist until manually cleared.

---

## Files Changed

| File | Change |
|------|--------|
| `dojotools/dojotools_room_manager.yaml` | New — v1.42.0. Full spatial topology tool. |
| `dojotools/dojotools_plant.yaml` | New — v1.2.2. Physical plant + energy manager. |
| `dojotools/dojotools_media_manager.yaml` | New — v0.7.2. Whole-home media management. |
| `dojotools/dojotools_security_manager.yaml` | New — v1.2.0. Replaces dojotools_alarm_panel.yaml (deleted). |
| `dojotools/dojotools_calendar.yaml` | New — v1.11.0. Split from office.yaml. |
| `dojotools/dojotools_todo.yaml` | New — v2.1.0. Split from office.yaml. `action`/`list_id` aliases; invalid action_type returns help. |
| `dojotools/dojotools_covers.yaml` | New — v0.2.2. ZenShade cover manager. |
| `dojotools/dojotools_kungfu_loader.yaml` | Restored — was incorrectly omitted from branch. Factory KFC deployer. |
| `dojotools/dojotools_admintools.yaml` | Cortex v39 (Home First); dispatcher spamaster route |
| `dojotools/dojotools_alertmanager.yaml` | v1.3.0: severity labels, fire-once dedup, `clear_after_minutes` default 1440, GC sweep. v1.0.1: new `zen_dojotools_alertmanager` MCP CRUD tool appended. |
| `dojotools/dojotools_camera.yaml` | v1.4.0: `ai_task` gate, `sendto` expansion, 3h result expiry, lens cross-reference |
| `dojotools/dojotools_climate.yaml` (utilities) | v1.1.0: topology_context in GET |
| `dojotools/dojotools_core.yaml` | v4.5.6: `_zen_active_alerts` TTL sweep (step 4c) |
| `dojotools/dojotools_dispatcher.yaml` | v1.1.0: Postman Tier 2, spamaster + security_manager routes; full spamaster payload passthrough (scene, lights, jets, audio, chemistry, cover) |
| `custom_templates/zenos_ai/zenos_cabinets.jinja` | FC normalization: legacy drawer fallback `drawer.get('value', drawer if drawer else fallback)` |
| `custom_templates/zenos_ai/zen_os_1.jinja` | FC normalization: `_slot` and `_alerts` read guards |
| `custom_templates/zenos_ai/zen_query.jinja` | ZQ-1 v4.5.7: `friendly_name_regex` filter added |
| `dojotools/dojotools_ectoplasm.yaml` | v4.6.1: floor REST path, area_id fix |
| `dojotools/dojotools_grocy.yaml` | v4.10.0: idempotent units_add, RM integration, chores_by_area |
| `dojotools/dojotools_identity.yaml` | v4.5.6: FG-07 expansion cabinet fix, VolumeInfo guard |
| `dojotools/dojotools_index.yaml` | v4.9.1: +rm pipeline, area_entities fix, filter_json fix |
| `dojotools/dojotools_labels.yaml` | v4.6.0: target_areas, add/remove_label_to_area |
| `dojotools/dojotools_lights.yaml` | FG-05: area_entities() pre-scan |
| `dojotools/dojotools_manifest.yaml` | FG-05: domain filter on label_entities |
| `dojotools/dojotools_office.yaml` | v5.0.0: todo + calendar removed |
| `dojotools/dojotools_postman.yaml` | Ack loop, actionable notifications, image support |
| `dojotools/dojotools_filecabinet.yaml` | v4.7.0: FC normalization — always-wrap write struct, `set_timestamp` default true, `_` prefix read fix |
| `dojotools/dojotools_summarizers.yaml` | v4.3.0: dual-seed step 3c, `area_id` input field, `_seed_used` gate on HyperIndex, seed whitelist check against `zen_summarizer_seed_whitelist` |
| `dojotools/dojotools_scribe.yaml` | v1.4.0: `seed` and `area_seed` input fields, updated help (context_source_guide, per_area_rollup_pattern, tuning_guide) |
| `dojotools/dojotools_admintools.yaml` | v4.6.1: seed_whitelist_seed, `reset_template` seeds `zen_summarizer_seed_whitelist` into syscab, new `zen_admintools_summarizer_seed` script |
| `dojotools/dojotools_systemtools.yaml` | v4.5.9: `ha_reload_all` and `ha_reload_scripts` deferred via `zen_event`; all four reload modes config-check gated |
| `dojotools/dojotools_scheduler.yaml` | v4.5.5: `deferred_script_reload` and `deferred_reload_all` event triggers added |
| `dojotools/dojotools_spamaster.yaml` | v3.3.0: replaces calderaspas |
| `dojotools/dojotools_zenlux.yaml` | v0.5.1: sync_shades, burnout timer, RM hold gate |
| `flynn_oobe.yaml` | v4.2.0: RM-native room setup, persona handoff step, security_camera label |
| `zenos_ai/docs/room_manager.md` | New — v1.42.0 full reference |
| `zenos_ai/docs/plant_manager.md` | New — v1.2.2 full reference |
| `zenos_ai/docs/media_manager.md` | New |
| `zenos_ai/docs/spamaster.md` | New — includes calderaspas migration note |
| `zenos_ai/docs/alertmanager.md` | Updated |
| `zenos_ai/docs/zenlux.md` | Updated |
| `zenos_ai/docs/zenshade.md` | New |
| `zenos_ai/docs/getting_started/oobe.md` | v4.2.0 RM-native room setup; security_camera label; persona handoff |
| `zenos_ai/docs/getting_started/first_run.md` | v2026.6.0; Room Manager in rooms step; persona handoff; security_camera |
| `zenos_ai/docs/readme.md` | v2026.6.0; Tool Reference section; 2026.6.0 What's New; roadmap entry |
| `zenos_ai/docs/_index.json` | v10: tools section, 7 releases, 2 scripts, 3 kung_fu, 1 custom_templates |

---

## Upgrade Notes

**Room Manager first-run:** Call `mode=setup confirm_action=true` once to deploy the KFC. Then use `mode=area_create` and `mode=set` to register rooms. `adjacent[]` is built from `mode=link` calls — do not write it manually.

**Plant Manager first-run:** Call `mode=validate` to see which slots are wired. Apply `zen_plant_*` labels to pin sensors that aren't resolving via generic labels. `utility_billing` is the generic label for billing/tariff sensors from utility integrations.

**Security Manager:** If the old `zen_dojotools_alarm_panel` entity is still in your HA registry, delete it manually after upgrading (Settings → Entities). The script is gone — HA holds a stale entry.

**SpaMaster:** Same as security manager — delete the stale `zen_dojotools_spa_manager` entity if it appears unavailable after reload.

**AlertManager auto-expiry:** Audit any existing `alert_fire` calls that rely on alerts persisting indefinitely. Add `clear_after_minutes: 0` if they must survive beyond 24 hours.

**Office split:** No immediate action needed — Todo and Calendar scripts work unchanged. Clean up any internal references to the old office.yaml definitions at your own pace.

**OOBE users (re-running setup):** The room setup step now uses Room Manager instead of filecabinet drawers. Existing room topology data written by the old OOBE is not migrated — if re-running on an existing install, the AI will discover HA areas as usual but will not import old room drawer contents.

**AlertManager Tool first install:** `zen_dojotools_alertmanager` is a new script entity. It will not appear in the MCP tool schema after a script reload — HA must be fully restarted once. After the initial restart, normal reloads apply.

**KFC schema v1.4.0:** The updated `kfc_template` drawer (with `seed` and `area_seed` fields) is deployed by Flynn on next warmup. Existing KFCs are unaffected — `seed` and `area_seed` are optional. No manual migration needed.

**SystemTools + Scheduler — must ship together:** `dojotools_systemtools.yaml` v4.5.9 and `dojotools_scheduler.yaml` v4.5.5 are a hard dependency pair. SystemTools fires `deferred_script_reload` and `deferred_reload_all` events; Scheduler handles them. If you're patching just one file, patch both.
