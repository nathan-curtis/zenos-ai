# ZenOS-AI Room Manager (RoomReg)

**Version:** 5.6.5
**Script:** `zen_dojotools_room_manager`
**Codename:** RoomReg

> **Not the same system as Room Manager v3 / REFLEX.** RoomReg (this doc)
> answers "what is this room, physically" — topology, egress, emergency
> snapshots. Room Manager v3/REFLEX answers "what is this room doing right
> now" — live occupancy/activity state, reactive scenes. See
> [Room Manager v3 & REFLEX](room_manager_v3_reflex.md).

---

## Overview

Zen DojoTools Room Manager is the spatial intelligence hub for ZenOS-AI. It stores and serves physical room topology — portals, adjacency, transmission values, exits, safety equipment — and provides live context slices combining topology with real-time HA state.

Every room-aware ZenOS tool (ZenLux, Media Manager, Climate, Inventory) derives room context from here.

Key capabilities:

* Physical room topology storage (portals, adjacency, transmission)
* Egress and evacuation routing (exits[], emergency exits, drop heights)
* Safety equipment inventory (fire extinguishers, AED, hazmat)
* Live context slices (+light, +topo, +climate, +media, +inventory, +chores, +tasks, +calendar, +wiki, +tickets)
* Whole-house situational awareness via `home_overview`
* Crisis snapshot via `mode=emergency` — scenario-aware guidance, shelter classification, hazards, rally point, dispatch address
* Household profile store (address, zip_code, rally_point) via `mode=set`
* Label-intersection entity discovery for tasks and calendars
* Grocy inventory anchor per room

All topology is stored in the household cabinet under key `room_topology`.

---

## First-Time Setup

### Label Requirements

| Label | Applied to | Purpose |
|-------|-----------|---------|
| `room_layout` | HA Area (not entity) | Marks area as physically registered in Room Manager. Applied automatically by `mode=set` and `mode=area_create`. |
| `zen_rm_ignore` | Entity | Suppresses entity from all `home_overview` discovery passes (battery scan, alarm panel, plant snapshot). Apply to entities that exist but should not appear in whole-home rollups. |

### Setup Steps

**Step 0 — Enumerate area IDs** (never guess slugs)

```
zen_dojotools_inspect  mode=area_list
```

**Step 1 — Deploy KFC** (one-time)

```
zen_dojotools_room_manager  mode=setup  confirm_action=true
```

Stamps the Room Manager KFC drawer into the dojo cabinet. Safe to re-run.

**Preflight gate:** `mode=setup` checks that `sensor.zen_dojo_cabinet_resolved` and the Household Cabinet are accessible before touching Scribe. If either is missing, returns `{status: error, missing[], action}` with a specific HA Labels UI action — no partial writes. Uses the canon resolver entity (`states('sensor.zen_dojo_cabinet_resolved')`), not `label_entities()`.

**Step 2 — Register each room** (`room_layout` is applied automatically)

```
mode=set  area=<area_id>  walls=4  description="Main living space"
```

For rooms that don't exist in HA yet, use `area_create` — it creates the HA area, applies `room_layout`, and optionally inits topology in one call:

```
mode=area_create  area_name="Room Name"  description="Rear bedroom"
```

**Settling delay:** `area_create` waits 1.5 seconds after HA area creation before writing topology. HA sometimes needs this window to fully register the area — without it, back-to-back `area_create` calls in OOBE can fire topology writes before the area is resolvable.

**Step 3 — Link adjacent rooms**

```
mode=link  area=<area_id>  area_b=<neighbor_id>
```

Optional transmission values: `link_sound_tx=0.30  link_light_tx=0.55`

`adjacent[]` is derived automatically from portals written by `mode=link` — do not build it by hand.

---

## Modes

| Mode | Description |
|------|-------------|
| `get` | Room intelligence brief: area metadata, spatial topology, live HA state, opt-in context slices. `area=` required. |
| `set` | Write or patch room topology. Register new room or update walls, portals, windows, safety items, `grocy_location_id`. Upserts non-destructively. `dry_run=true` previews changes. Error responses are bifurcated: household cabinet missing returns a specific "label not assigned" message; `area=` omitted returns a separate "area required" message. |
| `list` | All registered rooms with `rm_managed` / `layout_labeled` flags. No `area=` needed. |
| `link` | Bidirectional adjacency + interior portal between `area=` and `area_b=`. Optionally set `link_sound_tx` and `link_light_tx` (0.0–1.0). |
| `unlink` | Remove bidirectional adjacency and portals between two areas. |
| `boundary_link` | Non-passable shared boundary (drywall, partition, floor/ceiling). Writes both sides. Requires `boundary_link_sound_tx` and `boundary_link_light_tx`. |
| `boundary_unlink` | Remove boundary link between two areas. |
| `emergency` | Crisis snapshot for the whole home or a specific room. `scenario=fire\|burglar\|weather\|medical\|general`. Returns `guidance{action,advisory}`, `exits[]`, `shelter{primary/secondary/avoid}` (floor-tagged), `safety_equipment[]`, `hazards[]`, `likely_occupied[]`, `location{name,address,zip_code,gps}`, `rally_point`. Pass `area=` to add `from_room{room_exits,adjacent_exits,nearest_shelter}`. |
| `home_overview` | Whole-house snapshot: `signal{}`, `alerts{}`, `domain_summaries{}`, `floors[]`, `weather{}`, `plant{}`, `home_mode`, `utility_index{}`. Optional: `include_notices=true`, `include_presence=true`. No `area=` needed. |
| `area_create` | Create a new HA area, apply `room_layout` label, and optionally init topology in one call. `area_name=` required. `floor_id_new=`, `description=`, `walls=` optional. Pre-flight guard: if the area already exists, returns clean `{status: error}` with a "use area_update or mode=set" hint — no exception, chat survives. Invalid `floor_id_new` also returns clean error with valid floor list. |
| `area_update` | Update floor assignment, topology, and compiled description (`area_note` + adjacency). `area=` required. Rename, icon, and picture must be set via HA Settings → Areas UI (Spook has no `update_area` service — response includes `ha_ui_advisory` when these are requested). Invalid `floor_id_new` returns clean error with valid floor list. |
| `area_delete` | Delete HA area and remove from room_topology. `area=` required. `confirm_action=true` required. |
| `utility` | Manage utility_index in household cabinet. `utility_action=list\|get\|set\|delete`. `utility_type=electric\|gas\|water\|...` required for get/set/delete. |
| `pathfind` | BFS shortest path between two areas or persons. Fields: `start` (area_id or person entity), `destination` (area_id or person entity), `max_hops` (default 20). Returns path as ordered list of area_ids, hop count, and portal sequence. Returns `no_path` if destination is unreachable within `max_hops`. |
| `room_occupant_prefs` | Guest-or-household-member prefs for a room, plus an independent `vendor_activity` caution flag. `area=` required. See below. |
| `room_status_set` | Write housekeeping status for a room: `clean`/`dirty`/`in_service`/`occupied`. `area=` and `room_status=` required. Decoupled from any guest-stay lifecycle — HA-local operational state only. See below. |
| `room_status_get` | Read housekeeping status. `area=` optional — omit for all rooms, pass for one. Areas never set return `status: unknown`, not a default of `clean`. See below. |
| `setup` | Deploy Room Manager KFC to dojo cabinet via Scribe. `confirm_action: true` required. Returns preview if omitted. |
| `help` | Full reference: purpose, when_to_call, seed_steps, domain_routing, schema, context_slices, concepts, modes. |

---

## Context Slices

Pass as comma-separated flags to `context_slices=` on `mode=get`. Any combination is valid.

| Slice | Returns |
|-------|---------|
| `+topo` | Open binary sensors (device_class: door/window/garage_door/opening/lock). `open_count`, `total_count`, `open[]` |
| `+light` | `lights_total`, `lights_on[]`, `avg_brightness_pct` (0–100) |
| `+climate` | First climate entity in area. `entity_id`, `hvac_mode`, `setpoint`, `current_temp` |
| `+covers` | `covers[]`, `open[]`, `avg_position` (0–100) |
| `+media` | Active media player. `entity_id`, `state`, `media_title`, `volume_level`. Returns `active_count: 0` when nothing playing. |
| `+inventory` | Grocy `object_lens` place lens for the area — tagged products, chores, expanded operational objects. Full data in `domain_context.room_manager[area_id].context.inventory` only. Per-entity `room_context` carries a slim `inventory_summary: {tagged_products, chores, status}` + `room_area_id` pointer. Alias: `+grocy` |
| `+chores` | Maintenance chores linked to products stocked in the area. `is_due`, `next_execution`, `cadence`, `assignee`. `context.chores.chore_actions{execute, edit, add}` — pre-built call shapes; pass `item=<chore name>`. `add` also takes `period_days=N`. Chores with a `product_id` also include `replace_action{step_1: chores_execute, step_2: stock_open_item}` — two-step replacement sequence. |
| `+tasks` | Todo entities whose labels intersect the area's HA labels. Each list entry includes `items[]` and `task_actions{complete, edit, add}` — pass `items=[<summary>]`. See Label-Intersection below. |
| `+conductor` | Todo entities labeled `schedule` (AI conductor queue). Always unfiltered — full list regardless of area. |
| `+calendar` | Calendar entities whose labels intersect area labels. 7-day lookahead. |
| `+wiki` | Wiki pages tagged with the area's ID via Lens Bus (`zen_dojotools_lens_dispatch` anchor_type=area_id). Returns `pages[]` with title/path/tags, `count`, `anchor`. Tag room wiki pages with the area slug to surface here. |
| `+tickets` | Open Zammad service desk tickets anchored to the area's ID (via `zen_dojotools_servicedesk mode=tickets_by_anchor`). Returns `tickets[]`, `count`, `anchor`. Tag tickets with the area slug to link them to this room. |

### Label-Intersection Discovery

Any entity (todo, calendar) tagged with **any label that the area also carries** is automatically surfaced in `+tasks` / `+calendar`. No explicit room assignment needed on the entity — just share a label.

Example: area `hot_tub_deck` carries labels `[hot_tub, back_yard, spa]`. A todo labeled `hot_tub` or `spa` will appear in `+tasks`.

---

## Spatial Topology Schema

### Portals (Interior / Exterior Passages)

Upserted by `to` (area_id) or `name` if provided. `clear_portals=true` resets list.

**Required:**
- `type`: `door` | `archway` | `passage`
- `normally`: `open` | `closed`
- `exterior`: bool
- `exit`: bool
- `emerg_exit`: bool

**Optional:**
- `name`: string — upsert key when multiple portals connect the same area
- `to`: area_id (target area, or null for exterior)
- `sound_tx`: 0.0–1.0 (ref: archway=0.85, door-closed=0.30, solid-core=0.15)
- `light_tx`: 0.0–1.0 (ref: archway=0.80, door-closed=0.02, glass-door=0.55)
- `exit_priority`: `primary` | `secondary` | `tertiary` | `bail_out`
- `note`: string

### Windows

Upserted by `direction` or `name`. `clear_windows=true` resets list.

**Required:**
- `direction`: `N` | `NE` | `E` | `SE` | `S` | `SW` | `W` | `NW`
- `type`: `single` | `double` | `sliding` | `casement` | `fixed` | `skylight`
- `exit`: bool
- `emerg_exit`: bool
- `drop_ft`: number (0 = ground safe, >0 = jump height in feet)

**Optional:**
- `name`: upsert key for multiple windows in same direction
- `exit_priority`: `primary` | `secondary` | `tertiary` | `bail_out`
- `count`: int (number of windows in this direction, default 1)
- `light_tx`: 0.0–1.0
- `note`: string

`emerg_exit: true` marks a designated NFPA-style evacuation route.

### Boundary Links (Non-Passable Boundaries)

Shared walls, floor/ceiling adjacency, soundproofed partitions. Written bidirectionally by `mode=boundary_link`.

Schema: `{to: area_id, sound_tx: 0.0–1.0, light_tx: 0.0–1.0, note: string}`

Use for thermal and acoustic reasoning when rooms are adjacent but not connected by a passage.

### Safety Items

Upserted by `name`. `clear_safety=true` resets list.

**Required:**
- `name`: string (upsert key)
- `type`: `fire_extinguisher` | `first_aid` | `aed` | `eyewash` | `spill_kit` | `other`

**Optional:**
- `location_note`: string (e.g., "wall-mounted NE corner", "under sink")
- `rating`: string (e.g., "ABC", "2A:10B:C", expiry date)
- `grocy_location_id`: link to Grocy location if the item is tracked in inventory

---

## Exits

`mode=get` returns `exits[]` — computed list of all egress-capable portals and windows where `exit=true` OR `emerg_exit=true`.

Sort key: `exit_priority` rank (primary=0, secondary=1, tertiary=2, bail_out=3) × 2 + (0 if `emerg_exit` else 1).

Each exit entry includes `source` (`"portal"` or `"window"`), egress fields, and for windows: `drop_ft` (0 = ground safe).

---

## mode=emergency — Crisis Snapshot

The crisis authority for the home. Synthesizes topology, safety inventory, and occupancy into a scenario-aware action brief. **Do not synthesize a crisis response from kata or index — call this first.**

```
mode=emergency  scenario=fire  area=kitchen
```

### Fields

| Field | Default | Notes |
|-------|---------|-------|
| `scenario` | `general` | `fire` \| `burglar` \| `weather` \| `medical` \| `general` |
| `area` | *(none)* | When provided, adds `from_room{}` block scoped to that room. |

### Response Shape

| Block | Content |
|-------|---------|
| `guidance{}` | `action` (EVACUATE / SHELTER / CALL 911 / ASSESS), `priority`, `advisory` string with scenario-specific instructions including rally point or dispatch address. |
| `from_room{}` | Present when `area=` supplied. `room_exits[]`, `adjacent_exits[]` (with `via_area_id/name`), `nearest_shelter[]` (adjacent rooms that are primary or secondary shelter). |
| `exits[]` | All emerg_exit-flagged portals and windows across the home, sorted by priority rank. |
| `shelter{}` | `primary[]`, `secondary[]`, `avoid[]` — floor-tagged. Primary = no exterior portals + no windows. Secondary = no exterior portals + has windows. Avoid = has exterior portals. Each entry includes `floor_id`, `floor_name`. |
| `safety_equipment[]` | Items from `safety[]` classified as equipment (fire_extinguisher, first_aid, aed, eyewash). |
| `hazards[]` | Items from `safety[]` with `rating: hazmat` or `type: hazmat`. |
| `likely_occupied[]` | Rooms with lights on or motion/occupancy/presence sensor active. |
| `location{}` | `name` (from `input_text.zenos_household_name`), `address`, `zip_code`, `gps{lat,lon}` (live from `zone.home`). |
| `rally_point` | Rally point string from household profile. |
| `topology_coverage{}` | `total_rooms` (all HA areas), `configured_rooms` (rooms in topology), `sparse_rooms[]` (rooms with no portal or window data), `complete` (bool), `advisory` (populated when sparse rooms exist — lists room names). |

### Scenario Guidance Summary

| Scenario | Action | Priority | Advisory |
|----------|--------|----------|---------|
| `fire` | EVACUATE | exits | Use exits in priority order. Avoid hazard locations. Call 911. Rally at rally_point. |
| `burglar` | SHELTER | shelter | Move to nearest primary shelter. Lock interior doors. Call 911. Do not confront. |
| `weather` | SHELTER | shelter | Lowest-floor interior room. Away from windows. Stay until official all-clear. |
| `medical` | CALL 911 | location | Address for dispatch surfaced. Stay on line with 911. Send someone to front door. |
| `general` | ASSESS | general | Review exits, shelter, and hazards. Call 911 if needed. |

---

## Household Profile

Stored in the household cabinet under key `household_profile`. Written via `mode=set` — no `area=` required.

```
mode=set  address="123 Main St, Springfield, TX 78000"
mode=set  zip_code="78000"
mode=set  rally_point="mailbox at end of driveway"
```

All three fields can be passed in a single call. The write is a non-destructive merge — unspecified fields are preserved.

| Field | Notes |
|-------|-------|
| `address` | Full street address for emergency dispatch. Store what 911 needs. |
| `zip_code` | Postal code. |
| `rally_point` | Human-readable rally point description. Read-back in every `mode=emergency` response. |
| `default_room` | Default area_id written to `household_profile.default_room`. Used as the implied starting area for pathfinding and other tools when no explicit `area=` is given. Set with `mode=set default_room=<area_id>`. |

GPS coordinates are **not** stored — read live from `zone.home` at query time.

---

## mode=room_occupant_prefs — Who's Relevant Here Right Now

"Who's the relevant person for this room right now, guest or household member, and what are their prefs." Used by domain tools (e.g. Kitchen's mealplan_suggest) so a family dinner gets the same allergen flagging a guest stay does.

```
mode=room_occupant_prefs  area=kitchen
```

**Priority chain:**

1. **Guest stay** — checks `zen_dojotools_rolodex mode=active_stay_prefs` for event-shaped guest presence (Twenty CRM `guestPrefs`). If active, this wins.
2. **Household member fallback** — if no active guest, resolves this room's static `occupants[]` assignment against `zen_dojotools_profile_editor`. No name→cabinet index exists for profile_editor's fixed target slots (`user`, `family`, `expansion_1`–`expansion_5`), so occupant name resolution is a bounded scan (max 7 reads per occupant) matching `first_name`/`preferred_name`. Allergens are unioned across all matched occupants (a missed one is a safety issue); `media_prefs` comes from the first occupant matched — a stated simplification, not a hidden guess.
3. **None** — no active guest and no occupant match.

**`vendor_activity`** is independent of the above chain — a vendor being present doesn't change whose meal/music prefs apply, it's an orthogonal caution signal. Computed via `zen_dojotools_rolodex mode=active_appointment` for a vendor/contact mid-appointment in this room right now, and attached to every branch's response (`guest_stay` / `household_member` / `none`). Use it to avoid firing a scene while a vendor's mid-repair.

### Response Shape

| Field | Notes |
|-------|-------|
| `active` | bool — true if a guest or matched household member was found |
| `source` | `guest_stay` \| `household_member` \| `none` |
| `guest_name` | Guest name (guest_stay) or joined occupant names (household_member) |
| `area_id` | Echoed area |
| `guest_prefs` | `{allergens[], media}` — from Twenty (guest_stay) or profile_editor (household_member) |
| `vendor_activity` | `{active, vendor_name, appointment_summary}` — present on every branch |

---

## mode=room_status_set / room_status_get — Housekeeping Status

Room-level housekeeping state — `clean`/`dirty`/`in_service`/`occupied`. **Deliberately decoupled from any guest-stay/reservation lifecycle** — this is HA-local operational state, not a Twenty object, and not part of the Steel Magnolia PMS reservation model. Storage mirrors `room_topology`'s own shape: a dict keyed by `area_id` on the household cabinet, merge-only writes (matches this file's existing non-destructive-merge convention).

```
mode=room_status_set  area=guest_suite_1  room_status=dirty  room_status_source=turnover
mode=room_status_get  area=guest_suite_1
mode=room_status_get                        # all rooms
```

### Fields

| Field | Required | Description |
|-------|----------|--------------|
| `area` | Yes (both modes*) | HA area. Must resolve to a real area. *`room_status_get` may omit `area` to return every room's status. |
| `room_status` | `room_status_set` only | `clean` \| `dirty` \| `in_service` \| `occupied`. |
| `room_status_source` | Optional | What triggered the change: `turnover` \| `manual` \| `autovac`. Default `manual`. |
| `room_status_note` | Optional | Free-text note (e.g. which chore triggered a `dirty` mark). |

### Response Shape

An area that has never had status set reads back as `{status: 'unknown', updated_at: none, source: none, note: ''}` — **explicit absence beats a silent default**, so an unset room never reports falsely as `clean`. `home_overview`'s response also surfaces this per-room under `housekeeping_status`.

---

## Tax / Real-Estate Fields (v5.4.1)

Per-room fields for depreciation and business-use calculations, set via `mode=set`:

| Field | Values | Notes |
|-------|--------|-------|
| `tax_bucket` | `conditioned` \| `conditioned_storage` \| `conditioned_circulation` \| `conditioned_utility` \| `under_roof_porch` \| `under_roof_garage` \| `under_roof_attic` | IRS/appraisal space classification. Feeds the denominator for `zen_codex_finance_depreciation`'s `asset_allocation_*` business-use-percent suggestions — see [Firefly III — Codex Tier](../plugins/firefly_iii.md#codex-tier). |
| `home_office_claimed` | boolean | Whether this room is claimed as a home-office deduction. |
| `parent_area_id` | area_id | Override for area hierarchy where the physical/logical parent isn't derivable from HA's own area-parent relationship. |

All three are optional and additive — omitting them preserves existing values on `mode=set`, same non-destructive merge behavior as the household profile fields above.

---

## KFC Registration (KF5)

`zen_dojotools_room_manager` self-registers its dojo drawer via `mode=kfc_manifest` (`room_manager` component, no seed — pure rollup) — see [Building a KFC — KF5](../kung_fu/building_a_kfc.md#kf5-self-registering-tools).

---

## home_overview — Whole-House Snapshot

Returns everything needed for AI home-state reasoning in a single call.

### Top-Level Fields

| Field | Content |
|-------|---------|
| `home_mode` | Current ZenOS home mode string (`sensor.zen_home_mode` or `input_select.zen_home_mode`). |
| `generated` | ISO timestamp of the snapshot. |
| `registered_room_count` | Total rooms in topology. |
| `floor_count` | Distinct floor count. |

### `signal{}` — Pre-synthesized

| Field | Content |
|-------|---------|
| `attention[]` | Fresh alerts < 4 hours old with `age_h`. Items needing action. |
| `stale_alerts[]` | Alerts >= 4 hours old. Informational. |
| `active_rooms[]` | Rooms with lights on or media playing. `{area_id, display_name, lights_on, media}` |
| `open_sensors[]` | Rooms with open doors/windows/locks. `{area_id, display_name, open_count, open[]}` |
| `all_quiet` | `true` if no fresh alerts and `priority_context='clear'` |

### `alerts{}`

`priority_context`, `priority_count`, `highest_urgency`, `active_alerts[]`, `oldest_since`.

### `domain_summaries{}`

Whole-home rollups: `lights{on_total, total}`, `covers{avg_position}`, `media{active_count}`, `climate{temp_min, temp_max}`.

### `maintenance{}`

`battery_count` (sensors below 20%), `alert_count` (active non-idle alerts).

Entities tagged `zen_rm_ignore` are excluded from battery scan and alarm panel discovery.

### `floors[]`

Per-floor grouping of rooms with room_count and rooms[].

### `weather{}`

Current conditions from the entity labeled `zen_weather` (or first available `weather.*`).

| Field | Content |
|-------|---------|
| `available` | `false` if no weather entity found or unavailable |
| `entity_id` | Source entity |
| `condition` | Current condition string |
| `temperature` | Current temperature |
| `humidity` | Current humidity |
| `wind_speed` | Current wind speed |
| `next_hour` | First forecast slot from `forecast` attribute |
| `detail` | `"weather_forecasts MCP — hourly | daily"` — depth call pointer |

### `plant{}`

Top-line physical plant snapshot. For depth, call `zen_dojotools_plant` directly.

| Field | Content |
|-------|---------|
| `live_kw` | Whole-home live draw in kW (null if no entity resolved) |
| `water_gal` | Current water reading in gallons (null if no entity resolved) |
| `detail` | `"zen_dojotools_plant — electric | water | hvac | gas | circuits | validate"` |

Discovery waterfall mirrors Plant Manager: `zen_plant_site_power` → `label:utility_main` → `label:main_panel` *site_power* for power; `zen_plant_water` → `label:water_usage` → `label:droplet` for water. Entities tagged `zen_rm_ignore` are excluded.

### `utility_index{}`

Utility registry from the household cabinet — same data as `zen_dojotools_plant` `utilities` field. Contains provider, contact, and service entry info for electric, gas, water, etc.

### `notices{}` — `include_notices=true`

Active system alerts, HA state, and a pre-built action queue. Default `false` — opt in explicitly.

| Field | Content |
|-------|---------|
| `zen_alerts[]` | Active ZenOS alerts from the kata cabinet |
| `persistent_notifications[]` | Active HA persistent notifications |
| `ha_repairs[]` | Open HA repair issues |
| `postman_dispatches[]` | Postman messages sent in the last hour |
| `action_queue[]` | Pre-built action entries: `{priority, item, tool, call}` — fire directly without additional resolution |

### `presence{}` — `include_presence=true`

Presence tracker block. Default `false` — opt in explicitly.

| Parameter | Values | Effect |
|-----------|--------|--------|
| `include_presence` | `true` / `false` | Include presence block in response |
| `presence_mode` | `filtered` (default) | Return only `hps`-labeled presence trackers — lean, low noise |
| `presence_mode` | `discover` | Return all BPS (Bluetooth/Presence/Sensor) candidates with `hint` (phone/wearable/mobile_tag/appliance/tablet/unknown) and `suggest` (whether to add `hps` label) — use to guide initial labeling |

Apply `hps` label to phone and wearable device trackers to enroll them in filtered mode.

### `confident_presence{}` — always on, no flag needed

High-confidence current-room lookup. Unlike `presence{}`
above (a raw BPS device-discovery scan, opt-in because it's comparatively
expensive), this only reads however many entities carry the
`zen_presence_room` label — cheap enough to include on every
`home_overview` call by default.

Person-keyed dict, one entry per person who resolves to a real HA
`person.*` entity **and** is currently `home`:

```json
"confident_presence": {
  "person_a": {"room": "garage", "confidence": "medium", "source": "bayesian", "entity": "sensor.person_a_confident_room"},
  "person_b": {"room": "unknown", "confidence": "low", "source": "last_confident_fallback", "entity": "sensor.person_b_confident_room"}
}
```

| Field | Meaning |
|-------|---------|
| `room` | Room slug (matches `room_topology` keys) or `unknown` |
| `confidence` | `high` / `medium` / `low` |
| `source` | `bayesian` (a live Bayesian sensor cleared threshold) or `last_confident_fallback` (no sensor currently confident — falling back to the last known room) |
| `entity` | The underlying `zen_presence_room`-labeled sensor |

Untracked people (no `person.*` entity) and tracked-but-away people are
simply **absent** from the dict — not shown with a null/unknown entry.

Implementation (a Bayesian presence grid + Markov/adjacency fusion layer,
in a household-custom package outside `packages/zenos_ai/`) is deliberately opaque to this field
— any install satisfying the `zen_presence_room` label contract populates
it, no code change needed here. Same data is also available reverse
(who's in room X) via the Lens Bus `presence` provider's `area_id` anchor
— see `library/lenses.md`.

---

## zen_rm_ignore Label

Tag any entity with `zen_rm_ignore` to suppress it from all `home_overview` discovery passes:

- Battery scan (`battery_count` rollup)
- Alarm panel discovery (`_ho_security`)
- Plant snapshot (`_ho_plant` — power + water waterfalls)

Useful for entities that exist and are functional but should not appear in whole-home rollups — test sensors, duplicate meters, auxiliary devices that would skew aggregates.

The ignore list is read once at `home_overview` start and applied uniformly via `reject('in', _rm_ignored)` filters.

---

## mode=utility — Utility Index

Read and write utility provider info for the home. Data is stored in the household cabinet under `utility_index` and injected into every `zen_dojotools_plant` response.

```
mode=utility  utility_action=set  utility_type=electric
  utility_name="CPS Energy"
  emerg_phone="1-800-555-0100"
  emerg_cutoff_loc="main panel, breaker 1"
  service_entry_loc="south wall of garage"
```

| Parameter | Notes |
|-----------|-------|
| `utility_action` | `list` (default) \| `get` \| `set` \| `delete` |
| `utility_type` | `electric` \| `gas` \| `water` \| any string key |
| `utility_name` | Provider name |
| `emerg_phone` | Emergency line for outages |
| `emerg_email` | Emergency contact email |
| `emerg_cutoff_loc` | Physical location of shutoff (main breaker, valve, etc.) |
| `service_entry_loc` | Where the utility enters the home |
| `utility_notes` | Free-form notes |

`utility_action=list` returns the full `utility_index{}`. `get` returns a single entry by type.

---

## Domain Routing

`mode=get` returns `domain_routing{}` telling downstream tools where to route intents:

| Domain | Tool | Notes |
|--------|------|-------|
| `lighting` | `zen_dojotools_lights` | Pass `area_id`. Reads adjacency for bleed-aware scenes. |
| `media` | `zen_dojotools_media_manager` | Pass `area_id` or area name. |
| `climate` | `zen_dojotools_climate` | Pass `area_id`. `spatial.area_sqm` provides thermal load context. |
| `spa` | `zen_dojotools_spamaster` | Present only if `spa_climate` entity detected in area. All spa intents. |
| `egress` | *(inline)* | Filter `exits[]` from get response. `drop_ft=0` = ground safe. `emerg_exit=true` = evacuation route. |
| `occupant_profile` | `zen_dojotools_profile_editor` | `occupants[]` names → `mode=read` to load persona. |
| `grocy` | `zen_dojotools_inventory` | `spatial.grocy_location_id` is the anchor. |
| `chores` | *(context slice)* | `context_slices=+chores` — chore_actions{execute, edit, add} included at context level |
| `tasks` | *(context slice)* | `context_slices=+tasks` — task_actions{complete, edit, add} included per list |
| `calendar` | *(context slice)* | `context_slices=+calendar` |

---

## Index / `+rm` Pipeline

When `zen_dojotools_index` is called with `output_fields=+rm`, Room Manager context is injected per entity:

- `room_context`: topology + live slices for the entity's area (topo, light, climate, covers, media, grocy anchor)
- `domain_context.room_manager[area_id]`: same context keyed by area_id for cross-area reasoning

This enables the Index → Inspect → Room Manager pipeline: a single Index call surfaces spatially-aware entity context for any area in the home.

---

## Transmission Reference Values

| Portal type | `sound_tx` | `light_tx` |
|-------------|-----------|-----------|
| Open archway | 0.85 | 0.80 |
| Door (open) | 0.70 | 0.70 |
| Door (closed, hollow-core) | 0.40 | 0.02 |
| Door (closed, solid-core) | 0.15 | 0.02 |
| Glass door (closed) | 0.30 | 0.55 |

| Boundary type | `sound_tx` | `light_tx` |
|--------------|-----------|-----------|
| Standard drywall | 0.20 | 0.00 |
| Soundproofed wall | 0.05 | 0.00 |
| Floor/ceiling (concrete) | 0.10 | 0.00 |
| Glass partition | 0.40 | 0.70 |

These are reference values — set what matches your actual build.

---

## Identity Gate (2026-08-16)

Room Manager joins the gated tools. All ~35 modes were surveyed; read stays fully open (matches the universal convention every other gated tool already follows — the cert boundary only ever sits at write/actuation, never read). Two certifications cover the entire write surface, both cert-only (no live-ack tier) and `cert_scope`-deny hard-blocked, same shared `cert_scope_check` mechanism as every other gated tool:

- **`room_topology_edit`** — structural edits: `set`, `setup`, `area_create`, `area_update`, `link`, `unlink`, `boundary_link`, `boundary_unlink`, `room_zone_set`, `room_zone_remove`, `landmark_set`, `landmark_remove` (level 1). `area_delete` requires level 2 — the existing per-cert graduated `max_level` mechanism, not a third cert class.
- **`room_behavior_control`** — `room_status_set`, `roomstate_enable`, `reflex_enable`, `reflex_dry_run`, `wasp_enable`, `reflex_wire` (area= write), and `room_control_set` writes **except** setting Paused itself (stays open — same "increasing safety never needs permission" asymmetry the pre-existing unpause gate already established) and the unpause path itself (untouched, its own separate gate — see the operator's manual, Section 5).

`room_control_override` (the pre-existing unpause gate) is additive, not replaced by these two. `scene_stage` (a thin delegate to ZenLux) gates there instead — see `components/zenlux.md`. `room_occupant_prefs` is pure read despite the name and was deliberately excluded from both new certs.

Grant either via `zen_dojotools_persona_editor mode=cert_grant cert_component=room_topology_edit` (or `room_behavior_control`). See the [Security & Certification System operator manual](../getting_started/security_certification_manual.md) for the full model.
