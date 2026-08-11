# ZenOS-AI Room Manager v3 & REFLEX

**Version:** rev 5 (2026-08-08) — `room_control_manager` isolation, wasp-hold rewrite, entertaining/guest hold, asleep window
**Files:** `blueprints/template/zenos/room_state.yaml`, `packages/zenos_ai/room_manager_v3/zen_room_manager_dispatch.yaml`
**Codename:** REFLEX

---

## Overview

Room Manager v3 is the live, per-room occupancy/activity state engine.
REFLEX is the reactive layer riding on top of it. Together they answer
**"what is this room doing right now, and what should happen because of
it"** — a different question from `zen_dojotools_room_manager` (RoomReg,
see [room_manager.md](room_manager.md)), which answers "what is this room,
physically, and how does it connect to the house."

The engine itself — the cascade/decay/timer logic that decides what
`sensor.<room>_state` reads right now — runs entirely as one Home
Assistant automation and one script, reacting to real signals in real
time; there's no MCP call that computes a room's state, only ones that
read or configure it. Setup, diagnostics, and REFLEX scene-wiring ARE
chat-callable, through `zen_dojotools_room_manager`'s v3-adjacent modes
(`label_discover`, `trigger_audit`, `coverage_map`, `wasp_enable`,
`roomstate_enable`, `reflex_enable`, `reflex_dry_run`, `reflex_wire`) —
see [Coverage &amp; Diagnostics](#coverage--diagnostics) and
[Wiring REFLEX Scenes](#wiring-reflex-scenes) below. Operators interact
with the live engine through:

- The room's `room_control_manager` select (`Auto` / `Paused` / `Automation` / `Cleaning` / `Asleep` / `Occupied` / `Engaged` / `Hold` / `Vacant`) — cabinet-backed, fully isolated from the legacy `room_control` select that Node-RED still owns
- HA Labels (Settings → Labels) — every opt-in feature is a label, not a config field
- `sensor.<room>_state` — the read surface, one per deployed room

For the full architectural treatment (cascade design, event bus, the
consolidation history), see [Architecture Ch. 22](../architecture/22_Room_Manager_v3_REFLEX.md).
For the plain-language walkthrough, see the [Room Manager Operator's Manual](../getting_started/room_manager_operators_manual.md).

---

## The One Automation, The One Script

| Entity | Role |
|---|---|
| `automation.zenos_room_manager_dispatch` | Every trigger (motion, media, locks, timers, resync events) in one automation, `mode: parallel`, branched internally by `trigger.id` |
| `script.zen_reflex_controller` | Mode-dispatched helper script: `signal_dispatch` \| `room_timer_reconcile` \| `cleaning_control_resync` \| `control_burnout_resync` \| `self_label_resync` |

Both are entirely label/naming-convention driven — deploying a new room
never requires touching either file.

---

## Deploying a New Room

1. **Declare helpers in the room's package file** (only what you want — every one of these is optional):
   ```yaml
   timer:
     <room>_room_timer:
       # THE shared clock. One timer backs decay for occupied, engaged,
       # AND asleep — room_timer_class (below) says which tier currently
       # owns it. Skip this and all three tiers become live-signal-only,
       # no grace period (a safe, simpler default, not a broken one).
       name: "<Room> Room Timer"
     <room>_control_burnout:
       # Safety net only. If room_control_manager is set to Automation
       # and nobody releases it before this timer expires, it snaps
       # back to Auto by itself. No effect on anything else.
       name: "<Room> Control Burnout"
     <room>_tv_sleep_timer:
       # Backs the opt-in TV Sleep Timer construct (Section 6 of the
       # operator's manual) — counts down while Asleep + media playing.
       name: "<Room> TV Sleep Timer"
   input_number:
     <room>_asleep_minutes:
       # Per-room override of the shared clock's asleep-class duration
       # (system default is 8h/480min). Only matters if room_timer is
       # also declared — this is what room_timer_class=asleep decays
       # against instead of the global default.
       name: "<Room> Asleep Timeout"
       min: 1
       max: 720
       initial: 30
   input_select:
     <room>_room_timer_class:
       # Paired with room_timer above — records which of the three
       # decay-backed tiers currently owns the shared clock. The
       # dispatch script is the only writer; you never set this by hand.
       name: "<Room> Room Timer Class"
       options: [occupied, engaged, asleep]
       initial: occupied
   ```

   **`room_control_manager` (the manual-override select) needs its own
   declaration** — it's a template `select`, not an `input_select` helper
   like the legacy `room_control` it replaced (see §22 "Full Disconnect"
   for why the swap happened). Its state comes from the household
   cabinet, not from HA's own entity storage — picking an option doesn't
   set the state directly, it fires an event and the dispatch script
   does the actual write, so validation (e.g. Hold's restricted exit
   list) lives in exactly one place instead of being duplicated wherever
   something wants to change a room's control value:
   ```yaml
   template:
     - select:
         - name: "<Room> Control Manager"
           unique_id: <room>_control_manager_cab
           options:
             - Auto
             - Vacant
             - Occupied
             - Engaged
             - Asleep
             - Hold
             - Paused
             - Automation
             - Cleaning
           state: >-
             {%- import 'zenos_ai/zenos_cabinets.jinja' as CABS -%}
             {%- set hh = (label_entities('Zen Household Cabinet') | select('match', 'sensor\\.') | list | first) | default('', true) -%}
             {%- set drawer = CABS.cabinet_drawer_value_mounted(hh, 'room_control_manager', {}) | from_json({}) -%}
             {{ drawer.get('<room>', 'Auto') }}
           select_option:
             - event: zen_event
               event_data:
                 event:
                   kind: room_control_request
                   metadata:
                     room: <room>
                     requested: "{{ option }}"
                     source: manual
   ```
   Then tag the entity itself with two labels: `room_control_manager` and
   the room's own label (same as everything else in this system). The
   `<room>_control_manager_cab` unique_id suffix matters — reusing the
   legacy select's `unique_id` silently blocks re-registration in the
   entity registry (see the orphaned-registry note in the
   [Steel Magnolia release notes](../releases/steel_magnolia.md)).

   <!-- ART (optional): control_flow_map.png — retro "level map"
        restyling of the Mermaid diagram below: small sprite icons
        (hand, joystick, mail envelope, filing cabinet, TV screen)
        connected by a dotted path like an 8-bit overworld map.
        Suggested size: ~500x200px, a companion beside the Mermaid
        diagram, not a replacement — keep the real diagram as the
        authoritative one. -->

   ```mermaid
   sequenceDiagram
       participant You as Human / AI
       participant Select as select.<room>_control_manager
       participant Bus as zen_event bus
       participant Dispatch as zen_room_manager_dispatch
       participant Cabinet as Household Cabinet<br/>drawer: room_control_manager
       participant Sensor as sensor.<room>_state

       You->>Select: pick "Asleep"
       Select->>Bus: fire zen_event<br/>kind=room_control_request<br/>{room, requested: Asleep}
       Bus->>Dispatch: reflex_event trigger
       Dispatch->>Dispatch: validate against allowed<br/>exits for current value
       Dispatch->>Cabinet: upsert {<room>: Asleep}
       Cabinet-->>Select: state: template re-reads drawer
       Select-->>Sensor: states(room_control_manager select) = Asleep
       Sensor-->>Sensor: manual override tier wins,<br/>state = asleep
   ```
   The select never writes its own state — it only ever asks. Everything
   that actually changes state goes through the dispatch script, which
   is what keeps the "Hold can only exit to these five values" rule
   (§22.2) enforced in one place instead of every caller having to know it.

2. **Deploy the state sensor** via the shared template blueprint:
   ```yaml
   template:
     - use_blueprint:
         path: zenos/room_state.yaml
         input:
           room: <room>
           friendly_name: "<Room Title> Room State"
           unique_id: <room>_room_state_v3
           trigger_entities:
             - <every entity this room's cascade reads — see below>
   ```
   `trigger_entities` is the authoritative live-reactivity list — anything
   tagged for this room but missing from here will not cause the sensor to
   re-evaluate when it changes. Include every signal entity, every timer,
   every latch, the `room_control_manager` select, and every duration
   input_number you declared in step 1.

3. **Apply labels.** This is the entire "configuration" surface:

| Label | Applied to | Effect |
|---|---|---|
| `<room>` (matches area_id) | The state sensor + every helper | Ties everything to this room. Auto-applied to the sensor by `self_label_resync` on next resync; helpers need it too. |
| `zen_room_state` | The state sensor | Marks it as the room's authoritative state sensor — auto-applied by `self_label_resync`, don't apply by hand unless bootstrapping a room whose sensor name doesn't match either naming convention. |
| `motion` | Motion binary_sensor(s) | Drives `occupied`; motion *edges* also drive nightlight while asleep |
| `occupied` | Presence binary_sensor(s) | Drives `occupied` |
| `engaged` | media_player(s), monitored docks | Drives `engaged` on start; on stop, arms the shared timer's engaged→occupied decay |
| `asleep` / `bed_occupancy` | Sleep-detection sensor(s) | Drives `asleep` |
| `hold` | Any binary_sensor/cover | Floors at `occupied` while true, no clock, instant fall-through on close ("fridge door mode") |
| `wasp_door` | A door contact sensor | Real wasp-in-a-box, level-based (no latch, no timer) — see [Architecture §22.9](../architecture/22_Room_Manager_v3_REFLEX.md) for the full formula. Motion/occupancy live AND every `wasp_door` for the room reads closed → `hold`, recomputed fresh on every render; any `wasp_door` opening clears it that same render, no memory of the last edge. `checking`/`checking_timer` no longer exist. Requires the room-level `wasp_enabled` gate below — `wasp_door` alone is not sufficient. |
| `wasp_enabled` | The room's Area, OR any entity carrying the room's label | Room-level opt-in gate — a room needs this AND at least one `wasp_door`-tagged entity before `hold` can ever fire from wasp. Deliberately opt-in: a room connected by an open archway instead of a real door (no way to distinguish "someone's inside with the door shut" from "there is no door") would misfire constantly if wasp were on by default. Read/write via `zen_dojotools_room_manager mode=wasp_enable area=<room>` — omit `wasp_room_enabled=` to read, pass `true`/`false` to write. |
| `smoke` / `carbon_monoxide` / `moisture` / `siren` | Detector entities | Arms `emergency_latch` — human/agent ack-only clear |
| `room_control_manager` | The room's control select | Enables `Paused`/`Automation`/`Cleaning`/manual-tier overrides. Zero shared entity/label/code path with legacy `room_control`. |
| `room_timer` + `room_timer_class` | Timer + paired select | Enables decay-backed occupied/engaged/asleep — one shared clock for all three. `asleep` class default is 8h (was 30m). |
| `entertaining_hold` | `input_boolean.zen_entertaining` + the room's own label | Opt-in per room. Outranks `occupied`/`vacant`, outranked by `engaged`/`asleep`/manual override/emergency. |
| `guest_hold` | `input_boolean.zen_guest_mode` + the room's own label | Same tier/ranking as `entertaining_hold`, independent source. |
| `autosleep_disable` | Any entity carrying the room's label | Kills automatic asleep-firing for this room entirely. Manual `room_control_manager` "Asleep" pick is unaffected. |
| `asleep_window_disable` | Any entity carrying the room's label | Keeps autosleep active but removes the night→wake window gate. |
| `autosleep_schedule` | A truthy-resolving entity (`input_boolean`/`switch`/`binary_sensor`, or `calendar`/`schedule.*`) + the room's own label | Authoritative override of the night→wake window — **replaces** the clock check entirely for that room rather than OR'ing with it. `asleep_window_disable` still outranks it if a room somehow carries both. For non-standard schedules (shift work, etc.) — see the operator's manual §6. |
| `asleep_hold` | A truthy-resolving entity + the room's own label | Forces the room to `asleep` directly while true — zero clock, zero trigger signal or window check needed. Structurally identical to `entertaining_hold`/`guest_hold` but feeds `asleep` instead of `hold`. Clears the instant the entity goes false, or a manual `room_control_manager` override. |
| `emergency_latch` | input_boolean | Enables the `emergency` tier |
| `nap_latch` / `nap_minutes` / `asleep_minutes` | input_boolean / input_number | Shortens asleep decay while the nap latch is on |
| `nightlight_timer` | Timer | Enables the nightlight construct (motion edge while asleep) |
| `control_burnout` | Timer | Auto-reverts `room_control_manager` out of `Automation` if not released in time |
| `vent_fan` | fan/switch entity | Enables occupancy-driven auto on/off |
| `scene_<tier>` (e.g. `scene_occupied`, `scene_asleep`) | Scenes | REFLEX Stage 2 fires these on real transitions |
| `zen_rm_ignore` | Any entity | Suppresses from whole-home rollups (shared with RoomReg's `home_overview`) |

4. **Naming-convention-only helpers** (no label needed — existence check is the opt-in):

| Entity | Convention | Feature |
|---|---|---|
| `timer.<room>_tv_sleep_timer` | fixed | TV Sleep Timer |
| `input_number.<room>_tv_sleep_minutes` | fixed | TV Sleep Timer duration |
| `input_number.<room>_fan_delay_minutes` | fixed | Vent Fan on-delay |
| `input_number.<room>_fan_min_runtime_seconds` | fixed | Vent Fan min-runtime before off |

5. **Fire a resync** (or just wait for HA to restart / for `zen_resolver_refresh`
   to fire naturally): `zen_dojotools_systemtools mode=zen_resolver_refresh`.
   This runs all four resync passes, including self-labeling — a brand new
   room's helpers get their labels applied automatically as long as they
   follow the naming conventions above, no manual tagging required for the
   sensor itself.

6. **Reload and verify:**
   ```
   zen_dojotools_systemtools mode=ha_config_check
   zen_dojotools_systemtools mode=ha_reload_scripts
   zen_dojotools_systemtools mode=ha_reload_automations
   ```
   Then check `sensor.<room>_state` — it should read `vacant` (or the
   correct live tier) with all attributes populated.

---

## Reading Room State

```
GetLiveContext, or:
states.sensor.<room>_state
```

| Attribute | Meaning |
|---|---|
| `state` | The resolved tier — see cascade order below |
| `engaged_active` / `occupied_active` / `asleep_active` / `checking_active` | Always true/false, never null |
| `*_last_trigger` | `{entity_id, friendly_name, last_changed}` or a timer-decay reason — why this tier is (or isn't) true |
| `emergency_active` / `paused_active` / `automation_active` / `cleaning_active` | `null` if this room has no `room_control_manager`/`emergency_latch` at all — distinct from `false` |
| `room_control_entity` / `room_control_state` | The override select and its current value |
| `room_timer_entity` / `room_timer_state` / `room_timer_class` | The shared clock's current status |
| `zone_presence` | Sub-zone dict (e.g. Aqara FP2 desk/bench zones), `null` if unused |
| `child_rooms` / `child_engaged` / `child_occupied` | Present only when this room has children configured |
| `last_trigger` | Top-level "why is `state` what it is right now" |

### Cascade order (highest wins)

```
emergency > manual override (room_control_manager) > asleep > engaged >
child-engaged > hold (wasp / entertaining / guest) > occupied (or fridge-door hold) > vacant
```

`checking` no longer exists as a producible state anywhere in the system.

---

## Coverage &amp; Diagnostics

Two `zen_dojotools_room_manager` modes audit a room's real wiring against
what it's actually labeled to have, both `area=` optional (omit for a
whole-home rollup) and both advisory-only — neither ever auto-applies
anything, `confirm_action` has no effect on either:

**`trigger_audit`** — per-class `tagged_wired` vs `tagged_gap`. Separates
classes covered by a purpose-trigger platform (motion/door/lock/media/
window/garage_door/cover — matched automatically via `target: label_id`,
no explicit listing needed) from classes that still require the entity to
be listed in `trigger_entities` by hand for the sensor to react to it.

**`coverage_map`** — the unified room-readiness view: `label_discover`'s
untagged-candidate scan + `trigger_audit`'s gaps + a `needs_helper` check
(a feature's label exists but its helper doesn't — e.g. `room_timer_class`
tagged with no paired `timer.<room>_room_timer`) + `wasp_room_enabled`
status + a `verify_domain_mismatch` flag (an entity's `device_class`
doesn't match what its label implies — advisory only, several real
device_classes are intentionally repurposed, e.g. `sensor.garage_bearing`
carrying `garage_door` for autoclose direction logic despite not being a
`binary_sensor`). Run this first when checking whether a room is actually
ready, rather than reasoning about it from the label list alone.

---

## Wiring REFLEX Scenes

REFLEX turns a `room_state.yaml` transition into a scene. Two label axes,
applied to the scene entity itself:

| Label | Meaning |
|---|---|
| `scene_<state>` | Which tier this scene fires for — `scene_vacant`, `scene_occupied`, `scene_engaged`, `scene_asleep`, `scene_paused`, `scene_automation`, `scene_cleaning`, `scene_emergency` |
| `<room>` | The room's own label — same one every other v3 signal is tagged with |
| `<home_mode_daypart>` (optional) | `home_wake` / `home_morning` / `home` / `home_evening` / `night` / `night_late` — reused directly from `zen_home_mode`'s own labels, no separate taxonomy. Omit for an any-daypart fallback. |

**Resolution**: `label_entities(scene_<state>)` ∩ `label_entities(room)` ∩
`label_entities(<daypart>)`, first match wins; falls back to dropping the
daypart filter if nothing matches with it applied.

**Inheritance**: `engaged`/`cleaning`/`emergency` fall through to
`occupied`'s resolution when they have no scene of their own — a dedicated
scene always wins over the fallback. `vacant`/`asleep`/`paused`/`automation`
never inherit.

**Checking the tool did this labeling correctly** (rather than tagging by
hand): `zen_dojotools_room_manager mode=reflex_wire area=<room>` — shows
candidate scenes already matched by name/label/area_id, the full
`wired_matrix` (every state × every daypart, same resolution REFLEX itself
uses), and `tier_status` per state (`wired`/`inherited` = covered, `gap` =
worth wiring, `ok_by_design` = `checking`/`paused`/`automation`, which
legitimately have nothing to wire). Omit `area=` for a whole-home rollup.

**Writing an assignment**: `reflex_assignments` takes a list of JSON
objects, one per scene — `{entity_id, state, mode?}` (`mode` optional,
omit for the any-daypart fallback) — plus `confirm_action=true`:

```
reflex_assignments=['{"entity_id":"scene.office_occupied_day","state":"occupied","mode":"home"}']
confirm_action=true
```

This creates the `scene_<state>` label if it doesn't exist yet, then tags
`[scene_<state>, room, mode?]` onto the entity — additive, never removes an
existing label. Writes are exact — no fuzzy matching on the way in, only
on `reflex_wire`'s own candidate-scan read side.

---

## Troubleshooting

**"Room sensor never leaves vacant"**
Check `trigger_entities` on the room's `room_state.yaml` blueprint call —
if the signal entity isn't listed there, the sensor won't re-evaluate when
it changes. A bare label retag is not enough; the entity must be in the
trigger list.

**"Room control resets itself after I set it to Automation"**
That's `control_burnout` doing its job — a burnout timer tagged for this
room expired before it was released. Either release it sooner, or don't
tag a burnout timer for this room if you want `Automation` to be sticky
indefinitely.

**"Vent fan / TV sleep timer never fires"**
Check the naming convention exactly — `input_number.<room>_fan_delay_minutes`,
not `input_number.<room>_delay_minutes` or similar. These are existence
checks, not fuzzy matches; a typo is a silent no-op, not an error.

**"New room's sensor isn't picking up its `zen_room_state` label"**
Its friendly name must resolve to one of exactly two live entity_id
conventions: `sensor.<room>_state` or `sensor.<room>_room_state`. If the
sensor was deployed with a friendly_name that produces neither (an unusual
slug collision), self-label won't find it — tag `zen_room_state` + the
room's label by hand once, and it will be maintained (shunted, not
re-applied) from then on.

**"How do I check if REFLEX is actually acting on state changes, without risking a wrong scene firing"**
Set both `reflex_enable` AND `reflex_dry_run` on (via the labeled switch
entities, or the household cabinet's `reflex_config` drawer) —
`reflex_dry_run` is a modifier of `reflex_enable`, not a standalone
alternative to it; with `reflex_enable` off, nothing happens regardless of
`reflex_dry_run`. With both on, every would-fire decision gets logged as a
`reflex_dry_run` event instead of actually calling `scene.turn_on`/
`timer.start` — verify the payloads, then flip `reflex_dry_run` off (leave
`reflex_enable` on) to go live.

---

## Related

- [Architecture Ch. 22 — Room Manager v3 & REFLEX](../architecture/22_Room_Manager_v3_REFLEX.md) — full design rationale, the consolidation history, and the RoomState/RoomReg boundary
- [Room Manager Operator's Manual](../getting_started/room_manager_operators_manual.md) — plain-language guide
- [Room Manager (RoomReg)](room_manager.md) — the spatial/topology tool, a different system
- [RoomState and Perception](../architecture/11_RoomState_and_Perception.md) — the theoretical layer this system implements
