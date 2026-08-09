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

There is no chat-callable MCP tool for v3/REFLEX. It runs entirely as one
Home Assistant automation and one script, reacting to real signals in real
time. Operators interact with it through:

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
       name: "<Room> Room Timer"
     <room>_control_burnout:
       name: "<Room> Control Burnout"
     <room>_tv_sleep_timer:
       name: "<Room> TV Sleep Timer"
   input_number:
     <room>_asleep_minutes:
       name: "<Room> Asleep Timeout"
       min: 1
       max: 720
       initial: 30
   input_select:
     <room>_room_timer_class:
       name: "<Room> Room Timer Class"
       options: [occupied, engaged, asleep]
       initial: occupied
   ```

   > **TODO — undocumented:** `room_control_manager` (the manual-override
   > select) is declared per-room, but there's no generic example of that
   > declaration anywhere in the shared repo — it isn't an `input_select`
   > helper (legacy `room_control` was; this replaced it after the
   > isolation incident, see §22 "Full Disconnect"), and it isn't defined
   > by any blueprint or template block in `room_state.yaml` or the
   > dispatch script either. The only real declarations live in the
   > house-specific per-room package files that intentionally stay out of
   > this shared build. Tag whatever entity you do create with the label
   > `room_control_manager` and the room's own label — the rest of the
   > system is a safe no-op without it — but the actual entity
   > declaration needs a real worked example here.

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
| `wasp_door` | A door contact sensor | Real wasp-in-a-box: opening floors to `occupied` and clears any latched wasp flag. Motion firing while every `wasp_door` for the room is closed sets `wasp_flag_active` (self-referencing template attribute, not a timer) and the cascade shows `hold`. Clears only on a fresh door-open or a manual `room_control_manager` override — never a timeout. `checking`/`checking_timer` no longer exist. |
| `smoke` / `carbon_monoxide` / `moisture` / `siren` | Detector entities | Arms `emergency_latch` — human/agent ack-only clear |
| `room_control_manager` | The room's control select | Enables `Paused`/`Automation`/`Cleaning`/manual-tier overrides. Zero shared entity/label/code path with legacy `room_control`. |
| `room_timer` + `room_timer_class` | Timer + paired select | Enables decay-backed occupied/engaged/asleep — one shared clock for all three. `asleep` class default is 8h (was 30m). |
| `entertaining_hold` | `input_boolean.zen_entertaining` + the room's own label | Opt-in per room. Outranks `occupied`/`vacant`, outranked by `engaged`/`asleep`/manual override/emergency. |
| `guest_hold` | `input_boolean.zen_guest_mode` + the room's own label | Same tier/ranking as `entertaining_hold`, independent source. |
| `autosleep_disable` | Any entity carrying the room's label | Kills automatic asleep-firing for this room entirely. Manual `room_control_manager` "Asleep" pick is unaffected. |
| `asleep_window_disable` | Any entity carrying the room's label | Keeps autosleep active but removes the night→wake window gate. |
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
emergency > manual override (room_control_manager) > engaged > asleep >
child-engaged > hold (wasp / entertaining / guest) > occupied (or fridge-door hold) > vacant
```

`checking` no longer exists as a producible state anywhere in the system.

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
Set `reflex_dry_run` on instead of `reflex_enable` (via the labeled switch
entities, or the household cabinet's `reflex_config` drawer). Every would-fire
decision gets logged as a `reflex_dry_run` event instead of actually calling
`scene.turn_on` — verify the payloads, then flip to `reflex_enable`.

---

## Related

- [Architecture Ch. 22 — Room Manager v3 & REFLEX](../architecture/22_Room_Manager_v3_REFLEX.md) — full design rationale, the consolidation history, and the RoomState/RoomReg boundary
- [Room Manager Operator's Manual](../getting_started/room_manager_operators_manual.md) — plain-language guide
- [Room Manager (RoomReg)](room_manager.md) — the spatial/topology tool, a different system
- [RoomState and Perception](../architecture/11_RoomState_and_Perception.md) — the theoretical layer this system implements
