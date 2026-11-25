# ZenOS-AI AutoVac

**Version:** 5.14.2
**Script:** `zen_dojotools_autovac`

> **2026.12.0 (5.8.0 to 5.12.0):** a bare call answers help (it answered `status`). `mode=summary` is a scored read for the `autovac` component, behind the new read cert `zenos.autovac.status`. Schedules can be added, edited and removed from the tool. The pre-run briefing push and the run confirmations are settings, off on a fresh setup and kept on for an upgraded one. A run can be stopped (`mode=stop`) and read (`mode=run_state`), the new option `run_on_away` (off by default) runs what's due once the house goes Away, and the vacuum's two security settings show and edit here. See [The autovac component](#the-autovac-component), [Editing schedules](#editing-schedules), [Notices](#notices), [Run on Away](#run-on-away) and [Security options](#security-options).

> **2026.12.0 (5.13.0 to 5.14.2):** `mode=swap_robot` moves Autovac to a replacement vacuum and keeps the old one's records; exactly one vacuum may carry the `autovac` label. `clean` answers `dispatched`, and a run that never starts is reported. A drawer that exists but can't be read is never overwritten, and the controller reads the drawer through its mount. Maintenance tickets belong to the physical robot. New preset `roborock_qrevo_edge_3_pro`. See [Replacing the robot](#replacing-the-robot) and [Provisioning](#provisioning).

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
| `autovac_room_schedule` | Optional | Schedule entities | Room-scoped schedule — mapped to a room via matching `area_id` |
| `autovac_blackout` | Optional | Schedule entities | While `on`: blocks the room matching its `area_id`, or all rooms if the entity has no area |
| `autovac_mon` … `autovac_sun` | Optional | Schedule entities | Day-specific override — highest priority in day resolution |
| `autovac_weekday` | Optional | Schedule entities | Applies Mon–Fri when no day-specific label matches |
| `autovac_weekend` | Optional | Schedule entities | Applies Sat–Sun when no day-specific label matches |
| `Zen Household Cabinet` | Yes | `sensor.*` | Household cabinet (shared with other dojotools) |

Schedules are discovered dynamically from `autovac_schedule` label — add or remove schedule entities without changing the script.

---

## Modes

| Mode | Description |
|------|-------------|
| `help` | The default: a bare call answers help. |
| `setup` | Verify label wiring, init cabinet drawer, deploy KFC dojo entry — idempotent. `dry_run=true` previews without writing. `config={"create_labels":true}` creates every Autovac label Home Assistant doesn't have yet (the core five, `autovac_room_schedule`, `autovac_blackout`, `autovac_weekday`, `autovac_weekend`, `autovac_mon` to `autovac_sun`) and answers `label_objects_missing` for any it couldn't. Creating labels needs `zenos.registry.label_lifecycle_control` level 1 when an agent calls it. |
| `status` | Full system snapshot: vacuum state, room decisions, election, schedules, run readiness |
| `summary` | Scored, read-only seed for the `autovac` component. Needs `zenos.autovac.status`. See [The autovac component](#the-autovac-component). |
| `configure` | View or update room and system config via cabinet |
| `preset` | Apply a named config bundle to one or more rooms in one call. `preset=<name> rooms=<csv>` (or `room=<slug>`). No args: list all presets and their settings. Patches specified fields only — other settings preserved. Available: `obstacle_prone`, `mop_room`, `guest_room`, `high_traffic`, `twice_weekly`, `weekly`, `disabled`, `standard`. |
| `queue` | Mark a room for the next run regardless of schedule |
| `dequeue` | Remove a room from the queue |
| `skip` | Skip a room on the next run |
| `unskip` | Clear skip flag |
| `ready` | Mark a room ready to clean (for `requires_ready` rooms) |
| `unready` | Clear ready flag |
| `mark_cleaned` | Manually record a room as cleaned today |
| `pause` | Pause all scheduled runs for today |
| `unpause` | Clear today's pause |
| `clean` | Immediate clean by room slug(s) — vacuum must be docked or idle. Answers `dispatched`: the robot accepting the call is not the robot starting. A background check waits up to 90 seconds; if the robot never starts it clears the run and pushes "Run Didn't Start". |
| `run` | Schedule-triggered run — evaluates election, guards, dedup |
| `run_elected` | Run all currently elected rooms (bypasses schedule dedup) |
| `run_away` | Run what the next election would run, now, because the house is empty. The controller calls it five minutes after the home mode turns Away when `run_on_away` is on. See [Run on Away](#run-on-away). Requires `zenos.autovac.operate`. |
| `stop` | Cancel the run: empties the force-queue, marks the run cancelled and sends the vacuum home. See [Stopping a run](#stopping-a-run). Requires `zenos.autovac.operate`. |
| `run_state` | What the vacuum is doing: `active` (cleaning, returning or paused), `queued` (rooms waiting) or `idle`. Read, no cert. |
| `consumables` | ERP loop — stock, reorder, provision, replace, purchase |
| `check_wear` | Check one or all wear sensors; alerts and queues shopping if worn and out of stock |
| `analyze` | Post-dock map analysis — updates room last_cleaned, broadcasts completion event |
| `briefing` | Pre-run announcement ~30 min before a schedule fires. Checks `autovac_calendar` label — "AUTOVAC HOLD" events in the 4-hour run window block the run; other events appended as soft warnings in the message. |
| `handle_ack` | Process Postman push notification ack — go now, skip this run, or pause all day |
| `maintenance_set` | Mark the robot out of service. `input_json`: `{reason, ordered_part, eta, note}`. Opens a Radar ticket, or amends the open one. Requires `zenos.autovac.operate`. See [Maintenance status](#maintenance-status). |
| `maintenance_clear` | Return the robot to service by closing its open maintenance ticket. Requires `zenos.autovac.operate`. |
| `maintenance_get` | Current maintenance status: whether an open maintenance ticket exists, and its reason, ordered part, and ETA. |
| `swap_robot` | Replace the robot: `action=plan` (the default, read only), `commit`, `finish`, `rollback`, `status`. See [Replacing the robot](#replacing-the-robot). |
| `nightly_reset` | Reset daily run flags and pause state (call at midnight) |
| `morning_reset` | Clear `is_ready` flags after morning run starts |
| `schedule` | Day-resolved schedule summary — see [Scheduling](#scheduling) |
| `schedule_edit` | Replace the time blocks on days of an existing Autovac schedule. See [Editing schedules](#editing-schedules). |
| `schedule_add` / `schedule_remove` | Create or delete one Autovac schedule, labeled so the controller finds it. See [Editing schedules](#editing-schedules). |
| `schedule_blocks` | Raw `schedule.get_schedule` response for all `autovac_schedule` entities, enriched with labels/state/area |
| `kfc_manifest` | KFC dojo manifest — component summary, seed call, and drift/trigger config for the Ninja Summarizer |
| `stacks_by_anchor` | Lens Bus provider — `area_id` anchor → `cleaning_evidence`. See [Lens Bus Provider](#lens-bus-provider) |
| `register` | Lens Bus — register this tool as a `cleaning_evidence` provider in the cabinet's `lens_registry` |
| `unregister` | Lens Bus — remove this tool's `lens_registry` entry |
| `health` | Lens Bus — health check (vacuum entity live, cabinet reachable, rooms configured, inventory provisioned) |
| `inspect` | Lens Bus — static capability descriptor (consumes/returns/fields) for provider discovery |

**Before setup.** The modes that write state (`configure`, `preset`, `pause`, `unpause`, `queue`, `dequeue`, `clear`, `skip`, `unskip`, `ready`, `unready`, `mark_cleaned` and the two resets) answer `autovac_not_initialized` and point at `mode=setup` when the drawer or the cabinet is missing. Before 5.9.0 they answered ok with nothing stored.

`config` is accepted as JSON text or as an object (Home Assistant hands JSON with no `true`, `false` or `null` over as an object, which was ignored before 5.9.0). `mode=configure room=<slug>` with no config shows the room.

---

## The autovac component

`mode=summary` is a read-only seed for the `autovac` Kung Fu component, built on the Print Shop pattern ([the reference pattern](../developer_standards.md#a-complete-tool-and-component-the-reference-pattern)).

- **Cert:** `zenos.autovac.status`, level 1, stages `isl` (interactive, background summary, live drawer), in the `zenos_agent_advanced` bundle. Without it the component drops out of the agent's prompt.
- **What it returns:** `urgency` 0 to 5, `action_required` (urgency 3 or more), `reasons`, `vacuum_state`, `battery_pct`, `can_run_now`, `blockers`, `elected_rooms`, `due_rooms`, `overdue_rooms`, `next_schedule`, `dnd`, `paused_today`, `out_of_service`, and `watch_entities` (real entity IDs, from labels).
- **Scoring:** a vacuum error 4; low water that stops mop rooms (`water_low_action=stop`) 3; the vacuum unavailable, an open out-of-service ticket, or a room overdue by twice its interval, 2 each; low water that only warns, or a low battery, 1. `suggested_act_event` is always null: the controller already alerts on a vacuum error, and the pre-run briefing covers low water.
- **The component** seeds from `mode=summary`, names the cert in `meta.requires`, and subscribes to `force_summary`. The controller emits `summary_force` for it when the vacuum docks or errors; the component's 120-minute emission cooldown keeps that from repeating.

---

## Maintenance status

Out-of-service status has no cabinet drawer. It is derived on each call from whether an open Radar (Zammad) ticket tagged `autovac`, `<robot_slug>`, and `maintenance` exists, so the ticket is the single source of truth. `robot_slug` is the robot's `robot_name` from the `grocy_catalog` cabinet key (the same identity `consumables` and `check_wear` use), slugified.

The structured fields (`reason`, `ordered_part`, `eta`) are stored as `key=value` lines in the ticket's latest article. `status` and `health` both include a `maintenance` key. Only the modes that need it query Radar, so a Zammad outage does not block cleaning. `briefing` does not report maintenance status yet.

With a fleet registry (after a `swap_robot`), a ticket also carries an `avrobot_<key>` tag and is matched by the active robot's key, so a retired robot's open ticket never takes the new one out of service. Without a registry the robot-slug match stays. `swap_robot action=plan` lists the old robot's open maintenance tickets.

---

## Replacing the robot

`mode=swap_robot` moves Autovac from one vacuum to its replacement. Rooms, schedules and `last_cleaned` stay as they are, and nothing is cleaned during a swap.

| Action | What it does |
|---|---|
| `plan` | The default, read only. `new_entity_id=vacuum.<new>` (and `old_entity_id`, by default the vacuum Autovac uses now). Shows both robots, blockers, warnings and the `autovac` label moves. |
| `commit` | Needs `zenos.autovac.setup` and `confirm_action=true`. Turns Autovac off, files the old robot and its catalog as retired in the `autovac_fleet` drawer, moves the labels and clears the active catalog. `config={"label_map": {"<old entity>": "<new entity>"}}` pairs entities the names don't. |
| `finish` | `confirm_action=true`. Restores the prior enabled value and returns a `safe_to_decommission` checklist. |
| `rollback` | `confirm_action=true`. Labels and catalog go back on the old robot; Autovac stays off if the old robot is unavailable. |
| `status` | The registry and any swap in progress. |

Between `commit` and `finish`: give the new robot a Home Assistant area, map its segments to areas in Home Assistant, then run `mode=consumables action=provision model_preset=<key>` for its own catalog.

- **One robot at a time.** Exactly one vacuum may carry the `autovac` label. With two, Autovac uses neither and reports `duplicate_autovac_label`.
- **The `autovac_fleet` drawer** holds the robot registry and the swap record, apart from the `autovac` drawer (every write to that drawer replaces its root). History keeps the last 10 swaps, each a compact entry.
- **An unreadable drawer is never overwritten.** When the `autovac` drawer exists but a read doesn't come back as a real drawer, the read is retried once, then every writing mode refuses with `drawer_unreadable` and an `autovac_drawer_unreadable` event records the read status and mount traversal. Read-only modes still answer.

---

## Scheduling

`mode=schedule` resolves which `autovac_schedule` entity is authoritative right now, using a priority system so day-specific and room-scoped overrides win over generic weekday/weekend schedules:

| Priority | Match | Labels |
|----------|-------|--------|
| 1 (highest) | Today's specific day | `autovac_mon` … `autovac_sun` |
| 2 | Weekday/weekend class | `autovac_weekday`, `autovac_weekend` |
| 3 (lowest) | No day label at all | Generic `autovac_schedule` entity |

A schedule entity with no matching label for today is excluded from resolution entirely. Response includes `valid_today` (all candidates ranked), `next` (soonest upcoming fire, priority-aware), `in_window` (any candidate firing within `window_start`/`window_end` seconds — defaults 1770–2400s), `active` (currently-open window, if any), and `schedule_7day` (7-day forecast per day-of-week).

`mode=schedule_blocks` returns the raw `schedule.get_schedule` block/event data per `autovac_schedule` entity (area, notes, custom fields) plus per-entity metadata (labels, state, area, next_event) — used when a caller needs the underlying schedule structure rather than the resolved summary.

**Room-scoped and blackout schedules** (separate from the day-resolution system above): a schedule entity labeled `autovac_room_schedule` is mapped to a room by matching its `area_id` against each room's configured `area_id`. A schedule entity labeled `autovac_blackout` that is `on` blocks its matching room (or all rooms, if the entity has no area) — evaluated as a blocker in room election alongside `enabled`/`skip_next`/readiness/water checks.

### Editing schedules

All three need `zenos.autovac.setup`, preview with `dry_run=true`, and need Spook 5.11.0+ (storage schedules only). Autovac creates a schedule only when you call it; the controller never does.

- **`schedule_edit`** replaces the blocks on the named days of an existing Autovac schedule (global, per-room or blackout); other days are untouched, and `"blocks": []` clears a day. `schedule_entity_id` plus `config={"day":["tuesday"],"blocks":[{"from":"09:00:00","to":"10:00:00","data":{"rooms":"kitchen"}}]}`. Block `data` keys are limited to `rooms`, `skip_rooms` and `mode`, and room slugs must be rooms Autovac has. A schedule without an `autovac_schedule`, `autovac_room_schedule` or `autovac_blackout` label is refused. The result reads the schedule back.
- **`schedule_add`** takes `config={"name", "kind": "global|room|blackout", "scope": "weekday|weekend|monday..sunday", "room": "<slug>", "days": {"monday": [{"from","to","data"}], ...}}`. `scope` is for global and room schedules only. It creates the helper with a predictable entity ID (from the name), then labels it: global gets `autovac_schedule` and the scope label; room gets `autovac_room_schedule`, `autovac_schedule` and the scope label, and goes in the room's Home Assistant area; blackout gets `autovac_blackout`, in the room's area when `room` is given, else it vetoes the whole house. Before anything is made, it refuses a name that exists, a bad kind or scope, a blackout with a scope, a room kind without `room`, a room Autovac doesn't have, a room with no matching area (set `area_id` with `mode=configure`), a bad day, block or data key, and any label that doesn't exist yet (run `setup` with `create_labels`).
- **`schedule_remove`** takes `schedule_entity_id` and deletes it. Only a schedule carrying an Autovac label can be removed.

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

### Run on Away

`run_on_away` is a system option, off by default (`mode=configure config={"run_on_away":true}`). When it is on, five minutes after the home mode turns Away the controller calls `mode=run_away` with `trigger_id=away`. Nobody is home, so it is safe to flush whatever the next election would run: rooms that are due or force-queued, enabled, not skipped, confirmed when a room needs confirming and vacant, minus any room a blackout vetoes. The rooms go through `mode=clean`, which checks the vacuum is docked or idle and writes the run record.

The call only acts when the option is on, Autovac is enabled and not paused for today, and the battery is at or above the minimum. When nothing starts it says why: `option_off`, `not_started` (disabled, paused, low battery, or every room vetoed) or `nothing_elected`. An "Autovac Starting" push goes out when run notices are on. A reload or restart during the five minutes restarts the wait. Called by hand without `trigger_id`, `run_away` runs the elected rooms like `mode=clean`, without those checks.

### Stopping a run

`mode=stop` cancels a run in progress: it empties the force-queue, records a cancel marker, and sends the vacuum home. With nothing running it answers `nothing_to_stop` and changes nothing. A stopped run is not a finished one: when the vacuum docks, the room stamping is skipped, and the `autovac_docked` event carries `cancelled: true`, `run_state` and `queued_rooms`. Security Manager reads that flag (see its [vacuum sync](security_manager.md#vacuum-sync)). `mode=run_state` is the read-only answer to "is it running": `active`, `queued` or `idle`, with the vacuum state and any waiting rooms.

### Security options

Two Security Manager settings are about the vacuum: `vac_arm_override_enabled` (while the vacuum cleans with the alarm armed Away, move the panel to Home and put Away back at the dock) and `vac_downgrade_dock_wait_min` (minutes to wait after the dock, default 5). Autovac shows them and edits them next to the vacuum, but Security Manager owns them: `mode=configure` with no config returns `security_options` (the vacuum state and whether each setting applies), and `config={"vac_arm_override_enabled":true}` hands the edit to Security Manager's `auto_arm_config`. That needs the `zenos.actuation.security_control` cert and is refused with `vacuum_required` when no vacuum is labeled `autovac`, Autovac is not installed, or Autovac is off, and while the Security Sentry (needs Spook) is not running. Turning it off is always allowed.

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

Provision refuses a vacuum with no Home Assistant area, never takes a Home Assistant device ID as a serial number, reads wear sensors only from the robot's own device, and refuses to overwrite another robot's catalog. It reads the drawer back and answers `catalog_not_saved` if the catalog didn't save.

A preset part can record the evidence for its fit in `compatibility_state`. A part reorders only when its fit is `exact_confirmed` (the default for a part with no `compatibility_state`); any other part is tracked but never reordered. A part with no wear threshold gets no wear arithmetic. `roborock_qrevo_edge_3_pro` is the first preset that records its evidence.

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

### Notices

Two settings in the system config, set with `mode=configure config={"briefing_push":true}` or `{"run_notices":true}`:

| Setting | Gates |
|---|---|
| `briefing_push` | The "Autovac Pre-Run Briefing" push |
| `run_notices` | The confirmations Autovac Starting, Run Skipped, Paused and Dry Run |

A fresh `setup` writes both as `false`. A drawer from before 5.9.0 (`briefing_push`) or 5.10.0 (`run_notices`) has no key, and a missing key reads as on, so an upgraded install keeps what it had. The alerts are always on: Autovac Unavailable, Can't Start, Calendar Hold, worn parts, the dock notice and the one-time provision notice.

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

## Lens Bus Provider

Autovac registers itself as a Lens Bus provider of `cleaning_evidence`, keyed on `area_id` anchors (the room slug must match the HA area_id exactly):

| Mode | Purpose |
|------|---------|
| `stacks_by_anchor` | Given `anchor_type=area_id` and `anchor_ids=[...]`, returns `cleaning_evidence` per matched room: `last_cleaned`, `days_since_cleaned`, `days_between`, `due`, `queued`, `elected`, `blockers`, `requires_mop`. Any anchor type other than `area_id` returns an empty evidence list. |
| `register` | Writes an entry for `autovac` into the cabinet's `lens_registry` drawer (provider metadata: tool, department, consumes/returns, `failure_policy: soft`, `risk_class: read_only`). Merges with existing registry content. |
| `unregister` | Removes the `autovac` entry from `lens_registry`. Reports `was_registered`. |
| `health` | Reports `available`/`degraded` based on vacuum entity liveness, cabinet reachability, room config presence, and inventory provisioning. Degraded state lists which capabilities (`consumables`, `check_wear`, `stacks_by_anchor`) are affected. |
| `inspect` | Static capability descriptor — consumes/returns types, anchor-matching rule, evidence field list, failure policy, currently registered rooms. For provider discovery, not live evidence. |

---

## Dispatcher Registration

`zen_dojotools_autovac` is registered in `dojotools_dispatcher.yaml` as `zen_dojotools_autovac v1`. Fields passed through from `_payload`:

`mode`, `caller_token`, `room`, `config`, `rooms`, `dry_run`, `action`, `part`, `amount`, `wear_entity`, `preset`, `model_preset`, `schedule_entity_id`, `trigger_id`, `ack_action`, `ack_context`, `pm_tag`

**Gap:** the dispatcher's field passthrough has not been updated for the new modes — `window_start`, `window_end`, `component`, `anchor_type`, and `anchor_ids` are not in the list above (confirmed same on H:\, not a Zen-side omission). `schedule`, `kfc_manifest`, `stacks_by_anchor`, `register`, `unregister`, `health`, and `inspect` are therefore only reachable via a direct `script.zen_dojotools_autovac` call, not through a `dojotool_call` event.

---

## Dependencies

| Dependency | Purpose |
|------------|---------|
| `zen_dojotools_filecabinet` | Cabinet reads and writes (autovac drawer in household cabinet) |
| `zen_dojotools_inventory` | Grocy operations — stock, chores, shopping list |
| `zen_sutra_grocy` | Direct REST calls for stock queries during ERP loop (internal, called by the inventory tool) |
| `zen_dojotools_postman` | Briefing, wear alerts, provision notifications |
| `zen_dojotools_camera` | Post-dock map analysis |
| `Zen Household Cabinet` | `autovac` drawer — rooms + system config + grocy_catalog |
| `.autovac_presets/*.yaml` | Roborock model part catalogs (loaded via `!include`) |
