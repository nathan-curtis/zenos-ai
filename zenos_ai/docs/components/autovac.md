# ZenOS-AI AutoVac

**Version:** 5.1.0
**Script:** `zen_dojotools_autovac`

---

## Overview

Zen DojoTools AutoVac is the autonomous robotic vacuum management surface for ZenOS-AI. It handles the full lifecycle: room scheduling and election, schedule-aware runs, post-dock analysis, and a complete ERP loop for consumable tracking via Grocy.

Key capabilities:

* Schedule-aware room election with per-room frequency, readiness gating, and occupancy-based skip
* Immediate room clean by name (`mode=clean`)
* Dry-run briefing before scheduled runs
* Post-dock map analysis via the camera tool (coverage + obstacle detection)
* Full consumables ERP: robot provisioning, stock status, reorder, replacement logging
* Wear sensor monitoring — alerts when parts are worn, triggers shopping and maintenance chores
* DnD, water-low, and battery guards
* Dual integration support: Roborock native (`vacuum.clean_area`) and legacy Xiaomi MiHome (`xiaomi_miio.vacuum_clean_segment`) — auto-detected at runtime

Zero hardcoded entity IDs. All entity discovery is label-based.

---

## Label Taxonomy

Apply these labels in HA to wire up the integration:

| Label | Required | Entity type | Purpose |
|-------|----------|-------------|---------|
| `autovac` | Yes | `vacuum.*` | The robot vacuum |
| `autovac` | Optional | `camera.*` | Cleaning map camera (enables post-dock analysis) |
| `autovac_dnd` | Optional | `binary_sensor.*` | DnD (do-not-disturb) sensor — blocks runs when `on` |
| `autovac_water_low` | Optional | `binary_sensor.*` | Water tank low sensor |
| `autovac_schedule` | Yes | Schedule entities | One per run slot (morning/evening, weekday/weekend) |
| `autovac_wear` | Optional | Roborock wear sensors | Wear % sensors — wired to consumables catalog at provision time |
| `autovac_calendar` | Optional | `calendar.*` | Calendar entities — events with "AUTOVAC HOLD" in summary/description block runs; others surfaced as soft warnings in briefing |
| `autovac_current_room` | Optional | `sensor.*` | Live current-room sensor |
| `Zen Household Cabinet` | Yes | `sensor.*` | Household cabinet (shared with other dojotools) |

Schedules are discovered dynamically from `autovac_schedule` label — add or remove schedule entities without changing the script.

---

## Modes

| Mode | Description |
|------|-------------|
| `setup` | Verify label wiring, init cabinet drawer, deploy KFC dojo entry — idempotent. `dry_run=true` previews without writing. |
| `status` | Full system snapshot: vacuum state, room decisions, election, schedules, run readiness |
| `configure` | View or update room and system config via cabinet |
| `queue` | Mark a room for the next run regardless of schedule |
| `dequeue` | Remove a room from the queue |
| `skip` | Skip a room on the next run |
| `unskip` | Clear skip flag |
| `ready` | Mark a room ready to clean (for `requires_ready` rooms) |
| `unready` | Clear ready flag |
| `mark_cleaned` | Manually record a room as cleaned today |
| `pause` | Pause all scheduled runs for today |
| `unpause` | Clear today's pause |
| `clean` | Immediate clean by room slug(s) — vacuum must be docked or idle |
| `run` | Schedule-triggered run — evaluates election, guards, dedup |
| `run_elected` | Run all currently elected rooms (bypasses schedule dedup) |
| `consumables` | ERP loop — stock, reorder, provision, replace, purchase |
| `check_wear` | Check one or all wear sensors; alerts and queues shopping if worn and out of stock |
| `analyze` | Post-dock map analysis — updates room last_cleaned, broadcasts completion event |
| `briefing` | Pre-run announcement ~30 min before a schedule fires. Checks `autovac_calendar` label — "AUTOVAC HOLD" events in the 4-hour run window block the run; other events appended as soft warnings in the message. |
| `handle_ack` | Process Postman push notification ack — go now, skip this run, or pause all day |
| `nightly_reset` | Reset daily run flags and pause state (call at midnight) |
| `morning_reset` | Clear `is_ready` flags after morning run starts |

---

## Room Configuration

Rooms are stored in the `autovac` drawer of the `Zen Household Cabinet`. Add and configure them via `mode=configure`:

```yaml
# View current room config
zen_dojotools_autovac:
  mode: configure
  room: living_room

# Create or update a room
zen_dojotools_autovac:
  mode: configure
  room: living_room
  config: '{"enabled": true, "segment": 16, "days_between": 3, "requires_ready": false}'
```

**Room config fields:**

| Field | Description |
|-------|-------------|
| `enabled` | Include in election (default: true) |
| `segment` | Roborock segment ID (required for xiaomi_miio integration) |
| `area_id` | HA area slug (required for roborock native integration when slug ≠ room key) |
| `days_between` | Minimum days between cleans (default: 7) |
| `requires_ready` | If true: room won't run until explicitly marked ready via `mode=ready` |
| `requires_mop` | If true: triggers water-low checks before running |
| `notes` | Free text (logged to status output) |

**System config** (no `room=` parameter):

```yaml
zen_dojotools_autovac:
  mode: configure
  config: '{"water_low_action": "stop", "enabled": true, "minimum_battery": 20}'
```

| Field | Values | Description |
|-------|--------|-------------|
| `water_low_action` | `warn` / `stop` | Warn only, or block runs requiring mop |
| `enabled` | bool | Master enable/disable |
| `minimum_battery` | int (%) | Don't start a run below this battery % |

---

## Room Election

A room is elected for the next run when all of these are true:
- `enabled: true`
- `skip_next: false`
- Either: `days_since_cleaned >= days_between`, OR `queued: true`
- Either: `requires_ready: false`, OR `is_ready: true`
- Water is not low (if `requires_mop: true` and `water_low_action: stop`)

`mode=status` returns `room_decisions[]` — per-room elected/blocker/reason breakdown.

---

## Run Modes

### `mode=run` (schedule-triggered)

Called by a schedule automation. Guards:
- System disabled → skip
- Paused today → skip
- DnD active → skip
- Battery below minimum → skip
- Vacuum not docked or idle → skip
- No rooms elected → skip
- Already run for this schedule slot today → skip (dedup by schedule entity_id)

```yaml
zen_dojotools_autovac:
  mode: run
  schedule_entity_id: schedule.autovac_weekday_morning
```

### `mode=run_elected`

Runs all currently elected rooms — bypasses schedule dedup. Use for manual trigger or after a dry_run preview. Supports `dry_run: true`.

### `mode=clean` (immediate)

Clean specific rooms by slug, bypassing election:

```yaml
zen_dojotools_autovac:
  mode: clean
  rooms: living_room,kitchen
```

---

## Consumables ERP

`mode=consumables` manages the robot's consumable parts catalog through Grocy. The catalog is stored in the `autovac` cabinet drawer under `grocy_catalog`.

For end-to-end commissioning, including Postman policy setup and Grocy prerequisites, start with [AutoVac First Setup](../getting_started/autovac_first_setup.md). For the shared inventory contract used by AutoVac and SpaMaster, see [Grocy Inventory Component](../plugins/grocy.md).

### Actions

| Action | Description |
|--------|-------------|
| `provision` | Bootstrap robot as Grocy asset with full parts catalog. Idempotent via serial number + entity_id cross-reference. |
| `status` | Stock status for all cataloged parts — `ok`, `low`, or `out` per part |
| `add_to_shopping` | Queue all `low`/`out` parts to the Grocy shopping list |
| `log_replaced` | If `chore_id` set: executes chore (chore handles stock internally). If no chore: consumes the installed unit from `installed_location_id`. Always: opens next spare via `stock_open_item` — Grocy moves it to the install slot (`move_on_open` + `to_location_id` wired at provision time). |
| `log_purchased` | Add purchased spares to stock at the spare storage location |

### Provisioning

`action=provision` is the ERP bootstrap call. It:

1. Discovers the robot (entity + serial number via `autovac` label)
2. Resolves or creates Grocy locations (bot bin, dock bins, spare storage) using the HA area the vacuum entity belongs to
3. Creates a "Robot Machine" product in Grocy (tagged `autovac`)
4. Calls `provision_bom` — handles unit resolution, product create/find, meta update (`to_location_id`, `move_on_open`, `due_days_after_open`), HA label tagging (`autovac,autovac_part`), chore create/find, chore tagging (`autovac,autovac_chore`), and installed stock seed — all in one call per part
5. Maps `autovac_wear` labeled sensors to catalog part keys
6. Sends a Postman notification to log any additional spares on hand

After provisioning, `catalog_find_by_tag ha_labels=autovac` returns all robot products and chores in one call. Use `autovac_part` or `autovac_chore` to narrow.

```yaml
zen_dojotools_autovac:
  mode: consumables
  action: provision
  model_preset: roborock_s8_pro_ultra
  config: '{"robot_name": "Swiffer", "spare_storage_location": "Kitchen - Cabinet Left of Dishwasher"}'
```

`force: true` in config skips the idempotency check and re-provisions (existing stock untouched).

### Storage vs Consume Locations

Two distinct location types must be understood when provisioning:

| Location type | What it is | How it's set |
|---------------|-----------|--------------|
| **Spare storage** | Where uninstalled spare parts physically live (e.g. a kitchen cabinet) | `spare_storage_location` in config — **always pass this** |
| **Consume / installed** | Where parts are consumed from when in use (robot bin, dock bins) | Auto-created from `area_id(vacuum_entity)` — no config needed |

Per-category overrides are available if dock parts and wear parts live in different places:

```yaml
config: '{
  "robot_name": "Rosie",
  "spare_storage_location": "Kitchen - Cabinet Left of Dishwasher",
  "storage_location_dock_parts": <grocy_location_id>,
  "storage_location_wear_parts": <grocy_location_id>
}'
```

If `spare_storage_location` is omitted, all parts default to the dock location. This causes Plant `mode=managed` to show wrong storage and can produce false chore bleed from co-located machines.

### Model Presets

| Key | Model |
|-----|-------|
| `roborock_generic_dock` | Generic auto-empty dock |
| `roborock_generic_dock_ultra` | Generic Auto-Empty Dock Ultra |
| `roborock_generic_dock_nomop` | Generic dock, no mop capability |
| `roborock_s7_plus` | S7+ with Auto-Empty Dock |
| `roborock_s7_maxv_ultra` | S7 MaxV Ultra |
| `roborock_s8_plus` | S8+ with Auto-Empty Dock |
| `roborock_s8_pro_ultra` | S8 Pro Ultra |
| `roborock_q7_max_plus` | Q7 Max+ |

Presets define the parts catalog: SKUs, wear sensor keys, wear thresholds, storage location categories, chore periods, and chore names. Loaded via `!include` from `.autovac_presets/`.

---

## Wear Monitoring

`mode=check_wear` checks Roborock wear percentage sensors against catalog thresholds.

**Scan-all path** (no `wear_entity` provided): checks every cataloged part with a mapped wear sensor. Called post-dock by automation via the `vacuum.docked` label trigger (HA 2025.12+).

**Single-entity path**: checks one specific wear sensor by entity_id.

When a part is worn (`wear_pct <= threshold`):
- If spare stock > 0: sends Postman push — "X is worn, spare on hand, run `action=log_replaced`"
- If spare stock == 0: adds to Grocy shopping list + sends higher-urgency push

If a `chore_id` is set in the catalog for the part, the linked maintenance chore is executed automatically.

---

## Post-Dock Analysis

`mode=analyze` runs after the vacuum docks. Called by automation with `trigger_id=docked` or `trigger_id=error`.

What it does:
1. If a camera entity is labeled `autovac`: calls the camera tool to analyze the cleaning map (coverage, obstacles, completion assessment)
2. Updates `last_cleaned` for all rooms that ran
3. Clears `current_run` from the cabinet
4. Fires `zen_event kind: autovac_run_complete` with status, rooms_cleaned, area m², duration, and camera analysis

---

## Pre-Run Briefing

`mode=briefing` fires ~30 minutes before a scheduled run. Skips silently if the system is disabled or paused. Called by `zen_autovac_controller` via template trigger.

It:
- Identifies which schedule is about to fire (gate: `states(s) == 'off'` and `next_event.date() == today`)
- Lists rooms that will run and rooms waiting on your OK (`requires_ready`)
- Warns about low water, low battery, or a vacuum that hasn't run in 3+ days
- Auto-marks non-`requires_ready` due rooms as `ready_to_clean`
- Dispatches via Postman TTS with three push actions

**Briefing push actions:**

| Button | ack_action | Effect |
|--------|-----------|--------|
| "Go now!" | `go_now` | Fires vacuum immediately using the current elected list; marks this schedule slot done to prevent double-fire |
| "Skip this run" | `cancel_run` | Marks this schedule slot done only — other runs today are unaffected |
| "Pause all day" | `pause_day` | Sets `pause_today = true`, clears at midnight. Stops an in-progress run if one is running. |

`ack_context` format: `prerun_YYYYMMDD|schedule.entity_id` — used by `handle_ack` to identify the specific slot.

Legacy `cancel` ack (old "Stop it!" button) maps to `pause_day` for backward compatibility.

---

## Controller Automation

`zen_autovac_controller` is included in `dojotools_autovac.yaml`. It is a single automation that routes all generic triggers. No per-schedule automations to wire up.

| Trigger ID | Trigger | Action |
|-----------|---------|--------|
| `dock` | `vacuum.docked` on label `autovac` | analyze(docked) → check_wear → 1min delay → ninja_summarizer → `autovac_docked` event |
| `schedule_on` | `schedule.turned_on` on label `autovac_schedule` | `mode=run`; + `mode=morning_reset` if `'morning' in trigger.entity_id` |
| `briefing_timer` | Template: 30-min window before any `autovac_schedule` next_event | `mode=briefing` |
| `briefing_manual` | `zen_event kind: autovac_prerun_briefing` | `mode=briefing` |
| `nightly` | `time: 00:00:00` | `mode=nightly_reset` |
| `ack` | `zen_event kind: postman_ack ack_owner: autovac` | `mode=handle_ack` |
| `ha_start` | `homeassistant: start` | Catch-up `analyze(docked)` if vacuum is docked with a pending `current_run` in cabinet |

**Named-entity triggers** (vacuum error state, camera blueprint) require hardcoded entity IDs and must stay in your personal KFC file. Example personal KFC structure:

```yaml
# kfc_trigger_autovac.yaml — personal file, do not commit to repo
automation:
  # Error state → analyze (install-specific entity ID)
  - id: 'autovac_analyze_on_error'
    triggers:
      - trigger: state
        entity_id: vacuum.your_robot
        to: error
    actions:
      - action: script.zen_dojotools_autovac
        data:
          mode: analyze
          trigger_id: error
```

---

## Dispatcher Registration

`zen_dojotools_autovac` is registered in `dojotools_dispatcher.yaml` as `zen_dojotools_autovac v1`. All 17 fields are passed through from `_payload`:

`mode`, `caller_token`, `room`, `config`, `rooms`, `dry_run`, `action`, `part`, `amount`, `wear_entity`, `preset`, `model_preset`, `schedule_entity_id`, `trigger_id`, `ack_action`, `ack_context`, `pm_tag`

---

## Dependencies

| Dependency | Purpose |
|------------|---------|
| `zen_dojotools_filecabinet` | Cabinet reads and writes (autovac drawer in household cabinet) |
| `zen_dojotools_inventory` | Grocy operations — stock, chores, shopping list |
| `zen_dojotools_grocy_advanced` | Direct REST calls for stock queries during ERP loop |
| `zen_dojotools_postman` | Briefing, wear alerts, provision notifications |
| `zen_dojotools_camera` | Post-dock map analysis |
| `Zen Household Cabinet` | `autovac` drawer — rooms + system config + grocy_catalog |
| `.autovac_presets/*.yaml` | Roborock model part catalogs (loaded via `!include`) |
