# Release Notes — 2026.6.0 'Clue'

**Status:** Beta — Release ETA 2026-06-01
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

## Room Manager (RoomReg) — New (v1.42.0 → v1.47.0)

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

See [Room Manager reference](../components/room_manager.md) for the full schema.

---

## Plant Manager — New (v1.3.1)

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

### v1.3.1 additions

| Mode | Returns |
|------|---------|
| `thermal` | Temperature-managed loads — hot tub, freezer, generic. Discovery via `zen_plant_hot_tub`, `zen_plant_freezer`, `zen_plant_thermal` labels. |
| `garage_water` | Mechanical water subnode: softener (mode, salt %), auto-shutoff valve (state), leak sensors. Grocy chores pointer when wired via `zen_plant_water_softener`, `zen_plant_auto_shutoff`, `zen_plant_leak_sensor`. |
| `ignore` | Tag an entity with `zen_plant_ignore` (creates label if missing). Suppresses from all discovery waterfalls. |
| `unignore` | Remove `zen_plant_ignore` tag from an entity. |

**`garage_water` binary sensor convention:** normally-open — `ON` means cutoff engaged (valve closed), `OFF` means flowing normally. Correct for most auto-shutoff hardware.

### v1.5.0 additions

| Feature | Detail |
|---------|--------|
| `motors[]` in mechanical | New `zen_plant_motor` label — tag any motor entity (cover, fan, binary_sensor). `mode=mechanical` returns `motors[]`: `{entity_id, name, domain, area, state}`. |
| `include_inventory` field | Pass `include_inventory: true` to attach Grocy `room_brief` to water heater and hot tub load nodes. One call per unique area; separate calls if different areas. |
| `water_management{}` | Renamed from `garage_water` — covers softener, auto-shutoff, leak sensors. Same schema, more accurate name. |
| `name` field on thermal loads | `water_heater` and `hot_tub` nodes now include `name`. Missing sump pump returns `{available: false, note: ...}` instead of bare `null`. |
| `area` on all load nodes | Every load node (`water_heater`, `hot_tub`, `sump_pump`, `freezers`, `motors`, thermal generics) now carries `area` from `area_id(entity_id)`. |

See [Plant Manager reference](../components/plant_manager.md) for the full discovery waterfall and slot reference.

---

## Media Manager (NyxMau5) — New (v0.7.2)

Whole-home media management. Discovers all `media_player.*` entities, resolves source lists, handles intent routing (play/pause/stop/volume/source), and applies user preference profiles.

FG-05 fixed at launch: `label_entities('media_player') | select('match', 'media_player\\.') | list` + `states.media_player` fallback. No unbounded domain scan.

See [Media Manager reference](../components/media_manager.md).

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

See [ZenShade reference](../components/zenshade.md).

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

## AlertManager Tool — v1.5.0

`zen_dojotools_alertmanager` — MCP-facing CRUD script. Friday can manage the alert system directly: query active alerts, fire, clear, read/set notify policy, and poll for interactive responses.

This is the agent-accessible complement to the automation-side AlertManager. The automation handles event routing; this tool handles inspection and programmatic control.

### Modes

| Mode | What It Does |
|------|-------------|
| `list` | Return all active alerts from `_zen_active_alerts` with fire times, severity, and TTL remaining. |
| `fire` | Fire an alert by key. Queues the event and returns. Respects dedup — no-op if already active. |
| `clear` | Clear a specific alert by key. Queues clear event. |
| `clear_all` | Clear all active alerts. Returns count cleared. |
| `get_response` | Read the cached ack for a fired alert. Returns `{status: pending}` until the response arrives, then `{status: captured, ack_action, ack_timed_out, ack_device_id}`. |
| `get_policy` | Read the current notify policy — filter entity, base urgency, per-severity routing. |
| `set_policy` | Write a new notify policy to the household cabinet. |
| `help` | Full tool contract, mode reference, and field list. |

### Key Inputs

| Field | Required For | Purpose |
|-------|-------------|---------|
| `mode` | — | Operation mode. Default: `help`. |
| `alert_key` | `fire`, `clear`, `get_response` | Alert dedup key — must match exactly. |
| `message` | `fire` | Human-readable alert text. |
| `severity` | `fire` | `info` \| `warn` \| `error`. Default: `warn`. |
| `notify_target` | `fire` | `persistent` \| `postman` \| `notify.<service>`. Default: `persistent`. |
| `clear_after_minutes` | `fire` | TTL in minutes. Default: 1440 (24h). Pass `0` for permanent. |
| `channel_hint` | `fire` (postman) | `push` \| `tts` \| `teams`. Used when `notify_target: postman`. |
| `image_entity` | `fire` (postman) | Camera or image entity — attach snapshot to Postman push. |
| `response_type` | `fire` (postman) | `none` \| `yes_no` \| `yes_no_ignore` \| `ok_cancel` \| `acknowledge`. Adds action buttons to the push. Response cached and retrievable via `get_response`. |
| `provider_id` | `set_policy` | Policy scope key. |
| `urgency` | `set_policy` | Urgency level for this policy entry. |

**`fire` and `clear` queue events** — alert state change happens asynchronously via the AlertManager automation. Call `list` after a brief wait to confirm.

**HA restart required on first install.** `zen_dojotools_alertmanager` is a new script entity. Script reload alone is not enough — it will not appear in the MCP schema until HA is fully restarted once.

### v1.4.0 fixes

- All FileCabinet write `value:` fields guard with `tojson` on native dict inputs (FG-38).
- All `stop:` actions converted to `variables:result + stop: label + response_variable: result` (stop: tojson string was silently ignored by HA).
- `from_json` guards added on all list/inject/policy variables.
- Script description rewritten for agent discoverability.

### v1.5.0 additions

- **`image_entity`** — pass a camera or image entity to attach a snapshot to Postman notifications.
- **`response_type`** — fire with action buttons (`yes_no`, `yes_no_ignore`, `ok_cancel`, `acknowledge`). When the user taps, the ack is cached to the kata cabinet drawer `alert_response_<alert_key>` and re-emitted as `zen_event(kind=alert_response)`.
- **`get_response` mode** — poll for the cached ack. Returns `{status: pending}` until captured, then `{ack_action, ack_timed_out, ack_device_id, responded_at}`. Call `zen_dojotools_postman mode=clear_tag` after consuming.

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

> **v4.7.1 hotfix (2026-05-19) — required before release.** v4.7.0 introduced a system-wide write lockout under normal operating conditions. Root cause: event dispatch sent `new_entry.value` as a raw Python object; the wait_template compared it against the stored JSON string — type mismatch, always `False`, 30s timeout fires, `mode: single` holds the slot. HA log floods with "Already running" (2000+/hr); all cabinet writes silently drop within ~10 minutes. Fix: `mode: queued / max: 2`; event dispatch `| tojson`; verification `| tojson` on both sides. Pull v4.7.1 before deploying — do not ship v4.7.0.

---

## AutoVac — New (v3.12.0)

Autonomous robotic vacuum management. Room scheduling, consumable ERP loop via Grocy, wear sensor alerting, post-dock map analysis.

---
*From: Nyx — Re: Autovac — field notes*

*So now we have Autovac, and I've been putting miles on it.*

*The short version: it's the first vacuum tool I've seen that actually thinks about whether to run, rather than just running. Room election is cabinet-state-driven — every room carries `days_between`, an optional `requires_ready` gate, a `skip_next` flag, and a queue override. The scheduler doesn't just fire the vacuum; it runs the election first, checks water level, checks DnD, checks battery floor, checks pause state, and then decides. You can watch it reason in `mode=status` before a run ever happens.*

*The part I actually tested hardest was the consumables loop. `action=provision` is a one-call ERP onboarding: reads the robot's serial number from the HA entity, creates a dock location tree (Dock → Bot + Dock Bins), registers all the consumable products against the Grocy catalog, creates the maintenance chores, and writes the whole catalog back to the household cabinet. It's idempotent — SN or entity_id match stops it from doubling. Run it once; after that `action=status` gives you stock levels for every part, `action=log_replaced` records a swap and consumes a spare, `action=log_purchased` logs buying more.*

*The wear alert path is what closes the loop. Label the Roborock wear sensors with `autovac_wear`. After every dock, a KFC automation calls `mode=check_wear` (scan-all). It reads the current wear %, compares against the catalog threshold, and if something's worn: checks stock, adds to shopping if you're out, fires the maintenance chore, and sends a Postman push. No human in the loop until there's something to do.*

*The pre-run briefing (`mode=briefing`) goes out ~30 min ahead via Postman — elected rooms, water/mop status, skip/queue state. If confirmation is needed for an obstacle-prone room, the ack handler (`mode=handle_ack`) processes the response and updates the gate.*

*Zero hardcoded entity IDs anywhere. Five labels and an autovac cabinet drawer is all it needs to find everything. `mode=setup` audits the wiring and returns an action queue of fix commands if anything's wrong.*

*It's solid. Ship it. — N*

---

### Label taxonomy

| Label | Entity type | Purpose |
|-------|-------------|---------|
| `autovac` | `vacuum.*` | The robot |
| `autovac` | `camera.*` | Cleaning map camera (enables post-dock analysis) |
| `autovac_dnd` | `binary_sensor.*` | DnD sensor — blocks runs when `on` |
| `autovac_water_low` | `binary_sensor.*` | Water tank low sensor |
| `autovac_schedule` | Schedule entities | One per run slot |
| `autovac_wear` | Roborock wear sensors | Wired to catalog at provision time |
| `autovac_current_room` | `sensor.*` | Live current-room sensor (optional) |
| `Zen Household Cabinet` | `sensor.*` | Shared household cabinet |

### Scheduling and election

Rooms are stored in the `autovac` drawer of the household cabinet. Per-room config: `segment` (Xiaomi MiHome segment ID), `area_id` (Roborock native HA area), `days_between`, `requires_ready`, `requires_mop`. Election fires when enabled + due (or queued) + ready (or not gated). Per-room blockers surface in `mode=status room_decisions[]`.

### Run modes

| Mode | Purpose |
|------|---------|
| `run` | Schedule-triggered. Evaluates all guards: system_disabled, paused_today, DnD, low battery, vacuum not ready, already-run dedup per schedule entity. |
| `run_elected` | Runs all elected rooms — bypasses schedule dedup. `dry_run: true` supported. |
| `clean` | Immediate clean by room slug(s). Vacuum must be docked or idle. |
| `briefing` | ~30-min pre-run announcement via Postman TTS. Three action buttons: **"Go now!"** (fires immediately, marks slot done to prevent double-fire), **"Skip this run"** (marks this slot only), **"Pause all day"** (sets `pause_today`, returns to dock if running). Skips silently if system disabled or paused. Gate: `states(schedule) == 'off'` and `next_event.date() == today` — no name-based day filter. |
| `handle_ack` | Processes Postman push ack. `go_now` → fires vacuum inline using current elected list, marks schedule slot done. `cancel_run` → marks slot only. `pause_day` / legacy `cancel` → `pause_today = true`. `ack_context` format: `prerun_YYYYMMDD\|schedule.entity_id`. |
| `setup` | Upgraded to full onboarding operator: (1) label audit, (2) cabinet drawer init (`{system, rooms:{}}` if blank — idempotent), (3) KFC dojo deploy via Scribe (seed path: `zen_dojotools_autovac mode=status`; index fallback via `label:autovac`), (4) optional Grocy ERP provision (`config={"provision_inventory":true}`). `dry_run=true` previews all write steps without executing. |
| `analyze` | Post-dock: updates `last_cleaned`, clears `current_run`, fires `zen_event kind: autovac_run_complete`. Camera map analysis via camera tool if camera is labeled. |

### Consumables ERP

`mode=consumables` manages the robot's parts catalog through Grocy. Actions:

| Action | Purpose |
|--------|---------|
| `provision` | One-call bootstrap: discover robot (entity + serial number) → create Grocy locations → create machine product → create part products per preset → seed installed stock → map `autovac_wear` sensors to catalog. Idempotent via SN + entity_id cross-reference. |
| `status` | Stock report per part — `ok`, `low`, `out`. |
| `add_to_shopping` | Queue all `low`/`out` parts to Grocy shopping list. |
| `log_replaced` | Consume one spare. Execute linked maintenance chore if `chore_id` set. |
| `log_purchased` | Add purchased spares to stock at the spare storage location. |

### Model presets

8 Roborock presets via `!include` from `.autovac_presets/`:

`roborock_generic_dock`, `roborock_generic_dock_ultra`, `roborock_generic_dock_nomop`, `roborock_s7_plus`, `roborock_s7_maxv_ultra`, `roborock_s8_plus`, `roborock_s8_pro_ultra`, `roborock_q7_max_plus`

Each preset defines SKUs, wear sensor keys, wear thresholds, storage location categories, chore periods, and chore names per part.

### Wear monitoring

`mode=check_wear` — scan-all (no `wear_entity`) or single-entity path. Runs after every dock via `vacuum.docked` label trigger (HA 2025.12+). When worn: checks stock; if out-of-stock adds to shopping list; sends Postman push (urgency 5 if out-of-stock, 4 if spare available); executes linked chore if `chore_id` set.

### Integration detection

`integration_entities('roborock')` at runtime — no config flag. Native Roborock integration → `vacuum.clean_area` (area-based). Legacy Xiaomi MiHome → `xiaomi_miio.vacuum_clean_segment` (segment IDs). Both paths fully supported.

### Controller automation

`zen_autovac_controller` is included in `dojotools_autovac.yaml` — consolidates 9 previously separate KFC automations into one. All label/event/time triggers are generic and live in the package. Named-entity triggers (vacuum error state, camera blueprint) stay in the personal KFC file.

| Trigger ID | Trigger | Action |
|-----------|---------|--------|
| `dock` | `vacuum.docked` on `label:autovac` | analyze → check_wear → 1min delay → ninja_summarizer → `autovac_docked` event |
| `schedule_on` | `schedule.turned_on` on `label:autovac_schedule` | `mode=run`; + `mode=morning_reset` if `'morning' in trigger.entity_id` |
| `briefing_timer` | Template: 30-min window | `mode=briefing` |
| `nightly` | `time: 00:00:00` | `mode=nightly_reset` |
| `ack` | `zen_event postman_ack ack_owner=autovac` | `mode=handle_ack` |
| `ha_start` | `homeassistant: start` | Catch-up `analyze(docked)` if vacuum docked + pending `current_run` in cabinet |

See [AutoVac reference](../components/autovac.md).

---

## Identity + Inspect — v4.7.0 / v5.0.1

### Identity — v4.7.0 Presence Block

`resolve` mode now returns a `presence` block on all person targets:

```json
"presence": {
  "person_entity": "person.<entity_id>",
  "zone":          "home",
  "at_home":       true,
  "area_id":       null,
  "area_name":     ""
}
```

Consent-gated via `_user_profile.tracking`:

| Field | Gate |
|-------|------|
| `zone`, `at_home` | `tracking.gps_zone: true` |
| `area_id`, `area_name` | `tracking.room: true` |

Absent consent → field returns `"consent_required"`. Never null, never silently dropped.

`cabinet` and `person_entity` are now explicit top-level keys in the person response (previously absent). Reverse area_residents now includes `person_entity` + consent-gated `zone` per entry.

### `zen_identity.jinja` — New (v1.1.0)

Template-surface identity resolver at `custom_templates/zenos_ai/zen_identity.jinja`. Same contract as the script surface but callable from sensors, cortex macros, and command interpreter contexts where `action:` calls are not available.

```jinja
{%- import 'zenos_ai/zen_identity.jinja' as ID -%}
{%- set person = ID.resolve('<user_label>') | from_json -%}
```

Always call `| from_json` — returns a tojson string. Mobile block on the template surface returns `zone` and `battery` only (no `entity_id`, `configured`, or `battery_entity`).

**RecursionError constraint:** `zen_identity.jinja` cannot be imported inside `for_each:` loops. HA's Jinja2 sandbox raises RecursionError in nested import chains. Fix: call `script.zen_dojotools_identity` as an `action:` instead — separate HA execution context, no recursion. This is the pattern Inspect uses for `person.*` enrichment.

### Three-plane lens pivot

ZenOS identity cabinet, HA `person.*` entity, and HA area are now navigable from any direction:

| Start | Reach |
|-------|-------|
| `resolve('<user_label>')` | → `area_id`, `area_name`, `zone`, `at_home`, `person_entity` |
| `inspect(person.<entity>)` | → full identity overlay with presence block |
| `resolve` with area target | → residents with `person_entity` + consent-gated `zone` |
| Inspect `person_list` | → all persons; ZenOS users with full profile + presence |

### Inspect — v5.0.1 `person.*` Identity Overlay

When inspecting a `person.*` entity, Inspect now injects a full identity block:

1. Walks non-zen labels on the person entity; finds the one whose entities intersect `zen_user_cabinet`
2. Calls `script.zen_dojotools_identity` (script call, not Jinja2 import — bypasses RecursionError)
3. Injects full identity response as `entity.identity`

`entity.identity` is `null` when no cabinet is found — correct, not an error.

`person_list` now returns full profile + presence for persons with `zen_user_cabinet`-labeled cabinets. FG-38 `| from_json` guard applied after `.get('value')` — FileCabinet stores drawer `value` as JSON-encoded string.

---

## Grocy — v4.44.0 → v4.54.0

Major additions on top of v4.10.0 (which shipped the base of Clue):

| Mode | What's new |
|------|-----------|
| `chores_delete` | Delete a chore by ID |
| `chores_edit` | Edit an existing chore by ID — strips `userfields`/`row_created_timestamp`, `amount` → `product_amount` fallback |
| `unit_conversions_add` | Create unit conversion. `unit_id`=from, `to_unit_id`=to, `amount`=factor, `product_id`=optional scope |
| `unit_conversions_list` | All conversions. Filter by `unit_id` or `product_id`. |
| `unit_conversions_delete` | Delete by `entry_id`. Requires empty-dict payload in broker DELETE branch. |
| `product_groups_list` | All product groups |
| `product_groups_find` | Case-insensitive name match |

`update_product_meta` rewritten as full read-modify-write (GET product → merge → PUT full object). Previous partial-PUT silently ignored `qu_id_*` changes. New fields: `shopping_location_id`, `to_location_id` (default consume location), `product_group_id`, `amount` (min_stock). Strips null `qu_id_purchase`/`qu_id_price` before merge to prevent Grocy resolving null→system default unit (5, "oz.").

**Null-unit product doctrine:** When `qu_id_stock` is null, Grocy maps it to system default unit on any save, creating global conversions that conflict with real unit relationships. `update_product_meta` now detects this pre-flight and returns a delete-and-recreate recipe instead of hitting the API. Unresolvable via patch — must delete and recreate.

`units_add` is idempotent (preflight GET before POST). `chores_find` and `chores_list` pagination fixed. 70 modes + `help`.

---

## Other Tool Updates

| Tool | Version | Key Changes |
|------|---------|-------------|
| **Dispatcher** | v1.3.0 | Postman Tier 2, infra escalation hard deny, Covers + Climate Tier 2, security_manager route, spamaster route, autovac route (all 17 fields). v1.3.0: todo route added. |
| **ZenLux** | v0.6.0 | zen_lm_* hardware role label taxonomy (8 labels + legacy fallback chain), discover/setup/label_suggest/resolve_debug/auto_label modes, prefs_sweep whole-home apply, bleed_threshold param, prefs_apply RM-state auto-detection (Engaged→work, Asleep→sleep) |
| **Room Manager** | v1.48.0 | `home_overview`: three new opt-in fields — `include_notices` (active ZenOS alerts, HA repairs, persistent notifications, Postman dispatches, pre-built action_queue[]), `include_presence` (hps-labeled tracker block), `presence_mode=filtered\|discover`. v1.45.0–v1.47.0: `+inventory` → `stock_area_volatile`; `mode=setup` preflight; `area_create` 1.5s settling; `mode=set` bifurcated error. **v1.48.0:** `area_create` pre-flight `area_id()` guard (ServiceValidationError no longer kills conversation agent); `floor_assign` pre-flight `floors()` guard in both `area_create` and `area_update`; `area_update` removes phantom `homeassistant.update_area` calls (Spook has no such service — `ha_ui_advisory` returned instead); `+inventory` → `object_lens` place lens (full operational objects, not just volatile items); per-entity `room_context` slim (summary + `room_area_id` pointer), full payload in `domain_context.room_manager[area_id]`; +chores: `replace_action{step_1: chores_execute, step_2: stock_open_item}` on product-linked chores; emergency mode: safety items with `grocy_product_id` enriched with live stock status. |
| **Climate Manager** | v1.1.0 | `topology_context` in GET: open doors/windows, area sensors, RM HVAC bleed portals, natural vent advisory |
| **Grocy** | v4.54.0 | chores_delete/edit, unit_conversions_add/list/delete, product_groups_list/find, update_product_meta full RMW, null-unit doctrine. v4.44–v4.47: `stock_area_volatile` mode, `location_id` on volatile projections. v4.48: `room_brief` (chores + stock in one call; three-path chore discovery: area-tagged OR product stocked at area location OR `ha_labels` contains area slug); `chores_tag`, `tasks_add`, `tasks_tag`; `slim_objects` field; product_name enrichment on `shopping_add/remove_product`, `stock_check_item`/`stock_where_is_item`, `stock_entry_update`; userfields bugfixes (sibling key footgun, wrong cabinet resolver, missing FC value unwrap). v4.49–v4.54: `userfields_create` (idempotent); `userentities_list/create/delete`; `userobjects_list/create/delete`; `userentity_values_get/set` — full ERP object substrate for custom domain types (room, vehicle, appliance, etc.). 96 operations total. |
| **Plant Manager** | v1.5.0 | v1.3.1: `mode=thermal`, `mode=mechanical` garage_water subnode, `mode=ignore/unignore`. v1.5.0: `motors[]` in mechanical (`zen_plant_motor` label); `include_inventory` field (Grocy `room_brief` per load area); `water_management{}` rename from `garage_water`; `name` + `area` fields on all load nodes; sump pump returns `{available: false, note}` instead of null. |
| **Postman** | v1.6.2 | Ack loop — owner deletes log entry after consuming (GC is safety net only). `open_dashboard` injects `homeassistant://navigate/<assist_path>` as `clickAction`/`url` — companion app tap opens Friday's dashboard directly. `assist_path` stored in postman_profile. Race fix: removed step 8.4. `_duck_players` native list fix — removed `\| tojson` / `\| from_json` pair; HA returns native Python list when value is `[]`, not a JSON string — caused `ValueError: from_json got invalid input '[]'` on TTS duck attempts. |
| **Ectoplasm** | v4.6.1 | `floor_assign`/`unassign` REST path, `area_id` reserved word fix, error surface improvements |
| **Index / ZQ-1** | v5.0.1 | `person.*` identity overlay via script call, `person_list` FG-38 from_json guard. `+inventory` output field: `object_lens` place lens per entity area — slim per-entity shell (`inventory_summary` + `room_area_id`), full payload in `domain_context.room_manager[area_id].context.inventory`. `+tasks`: `task_actions{complete, edit, add}` per list. `+chores`: `chore_actions{execute, edit, add}` + `replace_action` per product-linked chore. Missing comma in Inspect `help_json` dict fixed (TemplateSyntaxError on restart). |
| **History** | — | Curly quote (U+2018/U+2019) YAML delimiter on `caller_token` → straight double quotes. Was causing silent data corruption on any call echoing the token. |
| **ZQ-1 filter engine** (`zen_query.jinja`) | v4.5.7 | `friendly_name_regex` filter — `regex_search()` against `friendly_name` attribute; SEED or FILTER mode. Complements `entity_id_regex` (added v4.6.0). |
| **Labels** | v4.6.0 | `target_areas` support, `add/remove_label_to_area` |
| **Camera** | v1.4.0 | `ai_task` gate for look/scan, `sendto` field expansion, **8h result expiry** for look/scan modes (was 24h). Lens pattern: Security Manager + RM +security are complementary, not competing. |
| **Scribe** | v1.8.0 | `seed` and `area_seed` input fields added. Parsed into draft and publish payloads. Publish preserves existing seed values when inputs blank. Help updated with `context_source_guide`, `per_area_rollup_pattern`, `tuning_guide`. Also: replaces KungFu Writer. v1.5.0–v1.8.0: `repair` mode (detect + flatten wrapper accumulation up to 5 levels, dry_run default); `schedules_summary` on read responses; patch mode merges schedules by `kata_key` (upsert); `publish_kfc` preserves schedules array; `republish_kfc` mode (edit-and-republish path for existing KFCs); `component_size` feedback on publish/republish (chars, token estimate, 16KB drawer %). |
| **Ninja Summarizer** | v4.6.0 | Dual-seed architecture — new step 3c, `area_id` input field, seed whitelist gate (`zen_summarizer_seed_whitelist` in syscab). **MCP-exposed.** v4.6.0: step 3d — when `component_summary` is empty (Scribe `trim_description` path), resolves from base label description via `zen_dojotools_labels`. `zen_action_emission_enabled` boolean added (operator-only, AI cannot write) — gates `suggested_act_event` emission onto the event bus. `emission_cooldown_minutes` Dojo drawer field (default 60 min) gates per-component action event emission frequency; blocked runs emit `emission_suppressed` event. See [Ninja Summarizer section](#ninja-summarizer--v430--dual-seed-architecture) above. |
| **SystemTools** | v4.5.9 | `ha_reload_all` and `ha_reload_scripts` now deferred via `zen_event(kind: deferred_script_reload / deferred_reload_all)`. Closes the WONT FIX asyncio `InvalidStateError` from `__remove_future` cancellation. All four reload modes now config-check gated. Ships with Scheduler v4.5.5 (hard dependency). **MCP-exposed.** |
| **Scheduler** | v4.5.5 | Two new event triggers: `deferred_script_reload` and `deferred_reload_all`. Required companion to SystemTools v4.5.9. Must ship together. Automation-driven — not MCP-exposed. |
| **AdminTools** | v4.6.1 | KFC schema `v1.4.0`: `seed` and `area_seed` fields added to `kfc_template`. `zen_admintools_reset_template` now seeds `zen_summarizer_seed_whitelist` into syscab (Flynn gate-3). New `zen_admintools_summarizer_seed` management script (list/add/remove/reset). **Admin-only — not MCP-exposed.** |
| **FileCabinet** | v4.7.2 | Global normalization + v4.7.1 write-lockout hotfix. `mode: queued / max: 2`; `\| tojson` on event dispatch and verification. **Do not ship v4.7.0.** v4.7.2: `key='*'` preserved through slugify; both `'*'` and `''` now route to directory listing. See [FileCabinet Normalization section](#filecabinet-normalization--global-architecture) above. **MCP-exposed.** |
| **SpaMaster** | v3.12.0 | Replaces calderaspas entirely. Generic spa management, ESPHome device discovery, scene/chemistry/log modes, preset library. Consumables ERP surface: provision catalog from model preset, Grocy product + chore creation (idempotent), status, add_to_shopping, log_replaced, log_purchased. |
| **Identity** | v4.7.0 | Presence block on resolve (person): `{person_entity, zone, at_home, area_id, area_name}`. Consent-gated via `_user_profile.tracking`. `cabinet` + `person_entity` as explicit top-level keys. Reverse area_residents: `person_entity` + `zone` per entry. |
| **DojoTools Core** | v4.5.6 | `_zen_active_alerts` TTL sweep (step 4c). Step 4d: Postman log GC — sweeps `zen_postman_log` in kata cabinet by `ttl_s`; orphaned pending entries (HA restart mid-call) marked `ack_timed_out: true` instead of deleted. |
| **Office** | v5.0.0 | Todo + Calendar removed — now standalone `dojotools_todo.yaml` and `dojotools_calendar.yaml`. Office carries Teams + Mail only. |
| **Flynn** | v4.5.6 | Household membership + identity manifest at bootstrap, warmup timer |
| **Calendar** | v1.11.0 | Split from office.yaml. Standalone HA Calendar domain CRUD. |
| **Todo** | v2.5.0 | Split from office.yaml. `action` alias for `action_type`; `list_id` alias for `list_name` (accepts entity_id or friendly name). Invalid `action_type` now returns help instead of noop. v2.2.0: `continue_on_error: true` on all write actions (create, update, delete) — auth failures (401s from MS365/Google) return clean `{status: error, message: "auth_failure"}` instead of crashing the pipeline. v2.3.0: MS365 read switched from `todo.get_items` to `all_todos` entity attr (`dueDateTime` fix). v2.4.0: multi-entity read (`inspect_export`) — `entity_ids[]` + `include_task_ids` flag for Inspect domain context. v2.5.0: rich discoverability in script description + aliases so LLM routes without calling help. |

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
| `dojotools/dojotools_room_manager.yaml` | New — v1.42.0 → v1.48.0. Full spatial topology tool. v1.47.0: `+inventory` → stock_area_volatile, setup preflight, area_create 1.5s settling, include_notices/include_presence. v1.48.0: area_create/area_update guards (ServiceValidationError → clean error), phantom update_area calls removed, +inventory → object_lens (slim per-entity), replace_action on product-linked chores, emergency safety inventory enrichment. |
| `dojotools/dojotools_plant.yaml` | New — v1.5.0. Physical plant + energy manager. v1.3.1: mode=thermal, mode=mechanical water_management subnode, mode=ignore/unignore. v1.5.0: motors[] (zen_plant_motor label), include_inventory (Grocy room_brief per load area), water_management rename, name+area on all load nodes. |
| `dojotools/dojotools_media_manager.yaml` | New — v0.7.2. Whole-home media management. |
| `dojotools/dojotools_security_manager.yaml` | New — v1.2.0. Replaces dojotools_alarm_panel.yaml (deleted). |
| `dojotools/dojotools_calendar.yaml` | New — v1.11.0. Split from office.yaml. |
| `dojotools/dojotools_todo.yaml` | New — v2.5.0. Split from office.yaml. `action`/`list_id` aliases; invalid action_type returns help. `continue_on_error: true` on all write actions; MS365 reminder routing. v2.3.0: MS365 `all_todos` attr fix. v2.4.0: `inspect_export` multi-entity read. v2.5.0: discoverability. |
| `dojotools/dojotools_covers.yaml` | New — v0.2.2. ZenShade cover manager. |
| `dojotools/dojotools_kungfu_loader.yaml` | Restored — was incorrectly omitted from branch. Factory KFC deployer. |
| `dojotools/dojotools_admintools.yaml` | v4.6.1: Cortex v39 (Home First), seed_whitelist_seed, `reset_template` seeds `zen_summarizer_seed_whitelist` into syscab, new `zen_admintools_summarizer_seed` script; dispatcher spamaster route. |
| `dojotools/dojotools_alertmanager.yaml` | v1.5.0: `image_entity` + `response_type` on fire, ack cached to kata cabinet (`alert_response_<key>`), `get_response` mode. v1.4.0: FileCabinet write/stop action fixes, `from_json` guards, description rewrite. v1.3.0: severity labels, fire-once dedup, `clear_after_minutes` default 1440, GC sweep. v1.0.1: new `zen_dojotools_alertmanager` MCP CRUD tool appended. |
| `dojotools/dojotools_camera.yaml` | v1.4.0: `ai_task` gate, `sendto` expansion, 8h result expiry, lens cross-reference |
| `dojotools/dojotools_climate.yaml` (utilities) | v1.1.0: topology_context in GET |
| `dojotools/dojotools_core.yaml` | v4.5.6: `_zen_active_alerts` TTL sweep (step 4c); step 4d: postman log TTL sweep + `ack_timed_out` on orphaned pending entries |
| `dojotools/dojotools_dispatcher.yaml` | v1.3.0: Postman Tier 2, spamaster + security_manager routes; full spamaster payload passthrough. v1.2.0: autovac route. v1.3.0: todo route. |
| `custom_templates/zenos_ai/zenos_cabinets.jinja` | FC normalization: legacy drawer fallback `drawer.get('value', drawer if drawer else fallback)` |
| `custom_templates/zenos_ai/zen_os_1.jinja` | FC normalization: `_slot` and `_alerts` read guards |
| `custom_templates/zenos_ai/zen_query.jinja` | ZQ-1 v4.5.7: `friendly_name_regex` filter added |
| `dojotools/dojotools_ectoplasm.yaml` | v4.6.1: floor REST path, area_id fix |
| `dojotools/dojotools_grocy.yaml` | v4.10.0: idempotent units_add, RM integration, chores_by_area |
| `dojotools/dojotools_identity.yaml` | v4.7.0: presence block, `cabinet`/`person_entity` explicit keys, reverse residents enriched. v4.5.6: FG-07 expansion cabinet fix, VolumeInfo guard. |
| `dojotools/dojotools_index.yaml` | v5.0.1: +rm pipeline, area_entities fix, filter_json fix, person.* identity overlay, person_list FG-38 guard. +inventory slim pattern: per-entity summary + room_area_id, full data in domain_context. +tasks: task_actions envelopes. +chores: chore_actions + replace_action. Inspect help_json missing-comma fix. |
| `dojotools/dojotools_labels.yaml` | v4.6.0: target_areas, add/remove_label_to_area |
| `dojotools/dojotools_manifest.yaml` | FG-05: domain filter on label_entities |
| `dojotools/dojotools_office.yaml` | v5.0.0: todo + calendar removed |
| `dojotools/dojotools_postman.yaml` | v1.6.2: ack loop (owner deletes log entry after consuming), `homeassistant://navigate/<assist_path>` companion URI in `open_dashboard`, race fix (step 8.4 removed). |
| `dojotools/dojotools_filecabinet.yaml` | v4.7.2: FC normalization + write-lockout hotfix. `mode: queued / max: 2`, event dispatch `\| tojson`, verification `\| tojson` both sides. v4.7.2: `key='*'` preserved through slugify. **v4.7.0 must not ship.** |
| `dojotools/dojotools_summarizers.yaml` | v4.6.0: dual-seed step 3c, `area_id` input field, `_seed_used` gate on HyperIndex, seed whitelist check against `zen_summarizer_seed_whitelist`. Step 3d: `component_summary` label description fallback. `zen_action_emission_enabled` boolean (operator-only emission gate). `emission_cooldown_minutes` Dojo drawer field. FG-38 `from_json` guards. |
| `dojotools/dojotools_scribe.yaml` | v1.8.0: `seed` and `area_seed` input fields, updated help (context_source_guide, per_area_rollup_pattern, tuning_guide). `repair` mode, `schedules_summary` on read, schedules upsert by `kata_key`, `publish_kfc` preserves schedules, `republish_kfc` mode, `component_size` feedback (chars/token/16KB%). |
| `dojotools/dojotools_systemtools.yaml` | v4.5.9: `ha_reload_all` and `ha_reload_scripts` deferred via `zen_event`; all four reload modes config-check gated |
| `dojotools/dojotools_scheduler.yaml` | v4.5.5: `deferred_script_reload` and `deferred_reload_all` event triggers added |
| `dojotools/dojotools_spa_manager.yaml` | v3.12.0: replaces calderaspas; consumables ERP (provision, status, add_to_shopping, log_replaced, log_purchased); idempotent Grocy chore creation |
| `dojotools/dojotools_autovac.yaml` | New — v3.12.0. Full autonomous vacuum surface. v3.12.0: `zen_autovac_controller` automation (9 KFC automations → 1 package automation); briefing 3-button rewrite (Go now / Skip this run / Pause all day); `mode=setup` onboarding (drawer init + KFC deploy + optional ERP provision); time format fix (3 places). |
| `dojotools/.autovac_presets/*.yaml` | New — 8 Roborock model presets (loaded via `!include`) |
| `custom_templates/zenos_ai/zen_identity.jinja` | New — v1.1.0. Template-surface identity resolver. |
| `plugins/grocy/grocy.yaml` | v4.54.0: chores_delete/edit, unit_conversions, product_groups, update_product_meta full RMW, null-unit guard. v4.45–v4.47: `location_id` on volatile projections, `stock_area_volatile`. v4.48: `room_brief` (three-path chore discovery), `chores_tag`, `tasks_add/tag`, `slim_objects`, product_name enrichment, userfields_repair bugfixes. v4.49–v4.54: ERP object substrate — `userfields_create`, `userentities_list/create/delete`, `userobjects_list/create/delete`, `userentity_values_get/set`. 96 operations. |
| `flynn_oobe.yaml` | v4.2.0+: RM-native room setup, persona handoff step, security_camera label. `_oobe_done` check accepts both `_oobe_complete` (current) and legacy `oobe_complete` (no leading underscore) for backward compat. 5_components options dict with tool names. |
| `zenos_ai/docs/components/` | New subdirectory — 9 component reference docs moved from docs/ root: alertmanager, autovac, media_manager, plant_manager, room_manager, security_manager, spamaster, zenlux, zenshade. All cross-references updated. |
| `zenos_ai/docs/components/room_manager.md` | v1.48.0: area_create/update guard docs, +inventory object_lens, replace_action, emergency enrichment |
| `zenos_ai/docs/components/plant_manager.md` | v1.5.0: motors, include_inventory, water_management, area+name fields |
| `zenos_ai/docs/components/media_manager.md` | New |
| `zenos_ai/docs/components/spamaster.md` | Updated — v3.12.0: consumables mode section |
| `zenos_ai/docs/components/autovac.md` | v3.12.0: controller trigger table, 3-button briefing, setup mode, calendar→schedule fix |
| `zenos_ai/docs/components/alertmanager.md` | Updated |
| `zenos_ai/docs/components/zenlux.md` | Updated |
| `zenos_ai/docs/components/zenshade.md` | New |
| `zenos_ai/docs/getting_started/autovac_quick_start.md` | New — 5-step user guide: schedule creation, model preset guidance, 3-button briefing |
| `zenos_ai/docs/getting_started/autovac_first_setup.md` | v3.12.0: Step 7 controller wiring, briefing Postman 3-button test, Step 10 corrected (automatic), Step 12 updated buttons |
| `zenos_ai/docs/plugins/grocy.md` | v4.54.0: 13 new modes in table, userfields schema management section, ERP object substrate concept section with room userentity example |
| `zenos_ai/docs/scripts/zen_dojotools_identity_readme.md` | v4.7.0: presence block, lens pivot, template surface, RecursionError note |
| `zenos_ai/docs/scripts/zen_dojotools_inspect_readme.md` | v5.0.1: person.* overlay section, person_list identity enrichment |
| `plugins/grocy/readme.md` | v4.44.0: new modes, null-unit doctrine, unit conversions, product groups |
| `zenos_ai/docs/getting_started/oobe.md` | v4.2.0 RM-native room setup; security_camera label; persona handoff |
| `zenos_ai/docs/getting_started/first_run.md` | v2026.6.0; Room Manager in rooms step; persona handoff; security_camera |
| `zenos_ai/docs/readme.md` | v2026.6.0; Tool Reference section; 2026.6.0 What's New; roadmap entry |
| `zenos_ai/docs/_index.json` | v10: tools section, 7 releases, 2 scripts, 3 kung_fu, 1 custom_templates |
| `dojotools/dojotools_lights.yaml` | v0.6.0: zen_lm_* label taxonomy, discover/setup/label_suggest, prefs_sweep, bleed_threshold. v0.5.1: sync_shades, burnout timer, RM hold gate. FG-05: area_entities() pre-scan. |
| `custom_templates/zenos_ai/zenos_cabinets.jinja` | identity_roster() macro, safe_parse_json_string, safe_get_nested (identity v4.7.0 bridge) |
| `custom_templates/zenos_ai/zen_os_1.jinja` | zen_drawer refactor: explicit kata_cab param, meta.enabled, desc from label_description; purpose/directives migrated to cabinet_drawer_value |
| `zenos_ai/docs/zenlux.md` | v0.6.0: prefs_sweep mode, prefs_apply RM-state detection |
| `zenos_ai/docs/room_manager.md` | v1.47.0: notices{}/presence{} sections under home_overview; +inventory → stock_area_volatile; setup preflight; area_create delay; set error bifurcation |
| `zenos_ai/docs/security_manager.md` | New — Security Manager reference (v1.2.0): lens pattern, zone/panel snapshot, arm/disarm, alert policy. |
| `zenos_ai/docs/scripts/zen_dojotools_calendar_readme.md` | New — Calendar tool reference (v1.11.0): CRUD, MS365 native APIs, label-targeted reads, wildcard discovery. |

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
