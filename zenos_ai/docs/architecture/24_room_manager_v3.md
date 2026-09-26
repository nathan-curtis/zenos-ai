# 24. Room Manager v3 Reference

Chapter 15 explains what a room's live state means and why it is built the way it is. This chapter is the operational reference: every tier, label, construct, and mode, and the rules the implementation holds to.

Room Manager v3 is not the same system as RoomReg (`zen_dojotools_room_manager`'s spatial modes: topology, egress, emergency snapshots, Chapter 8). RoomReg answers what a room is and how it connects to the house. Room Manager v3 answers what a room is doing right now, and what should happen because of it. When someone says "room manager," check which one they mean.

## 24.1 Rules of the design

* **Labels for judgment, naming for determinism.** Anything that needs a human to decide (which entity is this room's motion sensor, its timer, its control override) is found by label intersection: a class label plus the room's label, where the room label matches the area's `area_id`. Anything already fixed by the room slug at deploy time (`timer.<room>_tv_sleep_timer`, `input_number.<room>_fan_delay_minutes`) is found by name. A label for something the slug already predicts is taxonomy bloat.
* **Untagged is a safe no-op, everywhere.** No feature assumes a helper exists. Every tier and construct checks for its own helper and stays inactive without it. A room with no helpers at all still gets a working `vacant`/`occupied` cascade from motion alone.
* **One automation, one script.** The whole dispatch layer lives in `automation.zenos_room_manager_dispatch` and `script.zen_reflex_controller` (24.6).
* **Existence checks test one entity at a time.** Home Assistant's Jinja sandbox has a hard output ceiling of 262,144 characters. Materializing the house's entity list into one variable to test membership is unsafe at whole-house scale and is never done here.

## 24.2 The cascade

Every deployed room has one template sensor built from `blueprints/template/zenos/room_state.yaml`, found by the `zen_room_state` label. Its `state` is the resolved tier. The highest tier with an active source wins:

```text
emergency > manual override > asleep > engaged > child-engaged
  > hold (wasp, entertaining, guest) > occupied > vacant
```

| Tier | Source |
|---|---|
| `emergency` | `emergency_latch` input_boolean, armed by smoke, CO, moisture, or siren. Wins even over `paused`. Cleared only by a human or agent acknowledgement. |
| manual override | `room_control_manager` select: `paused`, `automation`, `cleaning`, `vacant`, `occupied`, `engaged`, `asleep`, or `hold`. Releases back to `Auto`. |
| `cleaning` | A manual override set by the Cleaning Dispatcher while a vacuum works the room. |
| `asleep` | Live signal (window-gated, 21.8), the shared timer decaying in class `asleep` (8 hour default), an `asleep_hold` entity, or a manual override (never window-gated). Beats direct `engaged` in the same room, and beats any child cascade. |
| `engaged` | Live signal (media playing, a monitored desk in use), or the shared timer decaying in class `engaged`. |
| `hold` | `wasp_flag`, `entertaining_hold`, or `guest_hold`. Same visible state, distinguished by `last_trigger` (24.7). |
| `occupied` | Live signal, the shared timer decaying in class `occupied`, a non-vacant child, or an active `hold`-labeled entity (a fridge door: floors the room at occupied with no clock, and falls through the moment it closes). |
| `vacant` | Nothing is true. |

There is no `checking` tier and no `checking_timer`. A room resolves cleanly or lands in `hold`.

**Direct asleep always wins.** Handlers that rearm a room's timer treat both `hold` and `asleep` as already held, so a motion event in an asleep room (a bathroom trip) cannot rearm the timer into class `occupied`. A child room going occupied cancels the parent's timer only when the parent's own state is outside `asleep` and `hold`. Either guard alone missing would let child activity or stray motion knock a sleeping room down to `occupied`.

### The shared timer

A room opts into decay by tagging one timer `room_timer`, plus a paired select `room_timer_class` with options exactly `[occupied, engaged, asleep]`. A room never decays two tiers at once, so one timer is enough. Ownership rules for who may start or take over the clock live in the Signal Dispatcher (24.4). The cascade sensor only reads which class holds it. A room without a timer is live-signal only: simpler, not broken.

Decay durations come from the room's `occupied_minutes`, `engaged_minutes`, and `asleep_minutes` helpers, defaulting to 10 minutes, twice the occupied value capped at 120, and 8 hours.

### Child rooms

An ensuite or sub-room contributes to its parent, default on, opt out per room with `child_link_disable`. A child at `engaged` promotes the parent to `engaged`. A child in any other active, non-paused state floors the parent at `occupied` for as long as it stays active. The parent's own direct `asleep` always wins.

### Explainability

Every tier that can be true carries a `*_last_trigger` attribute: `{entity_id, friendly_name, last_changed}` for the entity that most recently caused it, or `{reason, timer_entity, timer_last_changed}` when only timer decay is holding it. A top-level `last_trigger` does the same for whichever tier won.

## 24.3 REFLEX

**Stage 1, emitter.** Inside `room_state.yaml`. On a real transition (the resolved state differs from the sensor's previous value), it fires `zen_event` with `kind: room_state_changed` through `zen_dojotools_event_emitter`. Every room built from the blueprint emits with no extra wiring.

**Stage 2, resolver.** Inside `zen_room_manager_dispatch.yaml`. It resolves a scene by intersecting `scene_<state>`, the room label, and the current home-mode label. First match wins. With no match, it retries on room and state alone. No match at all is a no-op.

**Gates.** `reflex_enable` is the master gate. `reflex_dry_run` modifies it:

| `reflex_enable` | `reflex_dry_run` | Result |
|---|---|---|
| off | either | Nothing fires, nothing is logged |
| on | on | Logs what would have fired |
| on | off | Fires |

Every fire point respects both gates: the Stage 2 scene and both nightlight fire points.

**Debounce.** Stage 2 waits for the longer of the scene's transition or 2 seconds, re-reads the room's live state, and fires only if it still matches what this event resolved. A room that keeps changing lets its own later event fire the final scene. The automation runs `mode: parallel`, so in-flight checks never block newer events. The state sensor and the Stage 1 emitter are never debounced.

**Transitions.** Every `scene.turn_on` passes an explicit `transition:`, default 2 seconds. A scene labeled `reflex_transition_<N>` overrides it (`reflex_transition_0` is instant). This applies to every scene call in the system: the Stage 2 scene, the nightlight scene, and the nightlight-expiry re-fire.

**Nightlight.** Opt-in. While a room is `asleep`, a motion edge (not the level) fires its `scene_nightlight` scene and starts a `nightlight_timer`. On expiry the room's asleep scene re-fires. The room's state never changes: nightlight is scene-layer only.

**Scene labels.** Output scenes carry `scene_vacant`, `scene_occupied`, `scene_engaged`, `scene_asleep`, `scene_paused`, `scene_automation`, `scene_cleaning`, `scene_emergency`, or `scene_nightlight`. Only tiers that have scenes need them. These are distinct from the input-signal labels (`zen_occupied`, `zen_engaged`, `zen_asleep`) that tag source entities.

## 24.4 The Signal Dispatcher

The dispatcher listens for a fixed set of purpose triggers, each scoped by `target: label_id:` so it fires only for tagged entities:

`motion.detected`, `occupancy.detected`, `lock.locked`/`unlocked`, `media_player.started_playing`/`stopped_playing`/`paused_playing`, `air_quality.smoke_detected`, `air_quality.co_detected`, `moisture.detected`, `siren.turned_on`, `door.opened`/`closed`, `button.pressed`.

On a fire it reads the firing entity's labels and keeps the one that is a real area (`area_name(l) is not none`). That is the room. There is no room allowlist. It then writes exactly the helper the signal class needs, if tagged:

| Signal | Effect |
|---|---|
| smoke, CO, moisture, siren | Arm `emergency_latch` |
| `asleep`, `bed_occupancy` | Start the timer in class `asleep`, window-gated (24.8) |
| `motion`, `occupied` | Start the timer in class `occupied`, and run the wasp corroboration check if the room has a `wasp_door` and none is open |
| `engaged` ending | Start the timer in class `engaged` |
| `wasp_door` opening | Floor the room at `occupied` through the timer, clear any wasp hold |

**Tag the door, never the lock.** A lock and a door for the same opening must never both be signal candidates. Lock state does not track anyone crossing the threshold. Tag the door `wasp_door`. Locks carry `privacy_door` or `ext_lock` for security exposure only. `mode=label_discover` flags the pairing in `group_warnings`.

## 24.5 Opt-in constructs

Each activates by existence of its helper or label, checked at fire time. No per-room `use_blueprint:` call.

| Construct | Activation | Behavior |
|---|---|---|
| Control Burnout | Timer labeled `control_burnout` + room | Safety net for the `Automation` override only. On expiry the room reverts to `Auto`. Stateless resync. `Cleaning` self-clears, `Paused` is sticky and never touched. |
| TV Sleep Timer | `timer.<room>_tv_sleep_timer`, optional `input_number.<room>_tv_sleep_minutes` | A media edge while `asleep` restarts it. On expiry, turns off the room's media targets. Governs one device, not occupancy. |
| Vent Fan | Fan or switch labeled `vent_fan` + room, optional `input_number.<room>_fan_delay_minutes` and `_fan_min_runtime_seconds` | On entering `occupied`, waits the delay and turns on if still occupied. On leaving, waits the minimum runtime and turns off if still not occupied. |
| Wasp Hold | Entry labeled `wasp_door` + room, and the room carries `wasp_enabled` | 24.7 |
| Entertaining Hold | `input_boolean.zen_entertaining` labeled `entertaining_hold` + room | 24.7 |
| Guest Hold | `input_boolean.zen_guest_mode` labeled `guest_hold` + room | 24.7 |
| Asleep Hold | Truthy entity labeled `asleep_hold` + room | 24.8 |
| Autosleep Disable | Anything with the room label also labeled `autosleep_disable` | 24.8 |
| Asleep Window Disable | Anything with the room label also labeled `asleep_window_disable` | 24.8 |
| Autosleep Schedule | Truthy entity (toggle, calendar, `schedule.*`) labeled `autosleep_schedule` + room | 24.8 |

## 24.6 Dispatch layer

**The automation.** `automation.zenos_room_manager_dispatch`, `mode: parallel`, `max: 30`. Every trigger carries a `trigger.id` and a top-level `choose:` branches on it. Parallel is required because the vent fan's multi-minute wait would otherwise block every other trigger. Every other branch is either an idempotent resync or keyed to the single entity in its event, so concurrent runs converge.

**The script.** `script.zen_reflex_controller`, mode-dispatched like a DojoTool: `signal_dispatch`, `room_timer_reconcile`, `cleaning_control_resync`, `control_burnout_resync`, `self_label_resync`.

**Self-labeling.** `mode=self_label_resync` walks `areas()`, finds each room's state sensor by existence (`sensor.<room>_state` or `sensor.<room>_room_state`), and tags it `zen_room_state` plus the room label. It also tags the room's helpers (`room_control`, `room_timer`, `room_timer_class`, `asleep_minutes`, `vent_fan`) with their class labels, from a fixed list of label and naming-convention pairs applied the same way everywhere.

## 24.7 Control and hold

### `room_control_manager`

Each room's override surface is `select.<room>_control_manager`.

* Persistence is a per-room `room_control/<area>` key in the household cabinet, separate from the topology drawer so an override write never races a topology write. Not a per-room helper.
* The select's `select_option:` only fires a `room_control_request` event. The dispatcher's listener validates the request and writes the key.
* `Hold` has restricted exits: from `Hold` a room can go only to `Auto`, `Vacant`, `Paused`, or `Asleep`.
* `self_label_resync` tags the select `room_control_manager`. The cascade's manual-override tier reads only that.
* The name is by ownership (`_control_manager`), not by version. A version in an entity name becomes a portability problem at the next revision.

An override wins over live evidence when set, and releases on its own when the room's underlying live state changes away from what it was when the override was set (Chapter 15).

### Wasp hold

`wasp_flag_active` is level-based. It is recomputed from current state on every render and has no memory:

```text
wasp_enabled AND wasp_door entities exist AND NOT manual_override
  AND NOT any wasp_door open now
  AND (direct_occupied OR occupied timer holding)
```

Presence is live and every door is closed: the room is held. A door opens: hold clears on the same render, and the opening itself floors the room at `occupied`. It closes again with presence still live: hold resumes. Nothing latches, so nothing can get stuck.

`wasp_door` alone is not enough. The room must also carry `wasp_enabled`, on its area or on any entity with the room label. A room joined by an open archway cannot tell "someone inside with the door shut" from "there is no door," so wasp would misfire there. Set it with `mode=wasp_enable area=<room>`.

### Entertaining and guest hold

Two independent `hold` sources, opted in per room by labeling the existing shared boolean with the purpose label and the room label. A room whose label is not on the boolean is unaffected. Both outrank `occupied` and `vacant`, and are outranked by `engaged`, `asleep`, manual override, and emergency.

## 24.8 Asleep

**The window.** A direct `asleep` or `bed_occupancy` signal fires automatically only between the night and wake anchors (`input_datetime.zen_night_start` and `input_datetime.zen_am_start`, not the sun-based `sensor.period_of_day`). Outside it the signal is ignored at all three enforcement points: the live gate in `room_state.yaml`, the timer-arming handler, and the periodic timer reconcile.

| Label | Effect |
|---|---|
| `autosleep_disable` | No automatic asleep for the room. The manual override is unaffected. |
| `asleep_window_disable` | Automatic asleep at any time of day. |
| `autosleep_schedule` | The tagged entity's truthiness replaces the clock check for that room. A day sleeper's window has to shift, not widen. `asleep_window_disable` wins if both are present. |

**Asleep hold.** Structurally the same as the entertaining and guest holds, but it feeds `asleep` directly. While the tagged entity is true the room reads `asleep`, with no signal and no window check. It never arms the shared timer, and clears the moment the entity goes false or a manual override is set.

## 24.9 Tool surface

The cascade engine has no chat-callable surface. It runs on real signals. An operator steers a live room through its `room_control_manager` select and through labels.

Setup, diagnostics, and wiring go through `zen_dojotools_room_manager`: `label_discover`, `trigger_audit`, `coverage_map`, `wasp_enable`, `roomstate_enable`, `reflex_enable`, `reflex_dry_run`, `reflex_wire`, `room_control_set`, `room_signal_fire`, and `role_audit`.

**`room_control_set`** writes the same override key as the select, and needs certification: `room_behavior_control` for any write other than setting `Paused`, plus `room_control_override` with a live acknowledgement to clear a room out of `Paused` (unless the grant's scope names that area). Setting `Paused` is always open: increasing safety never needs permission. `room_signal_fire` is the other half: it simulates evidence and arms the shared timer, so the room decays normally instead of holding an override indefinitely. When a room leaves `Automation` by any path, it checks the household cabinet for a `media_activity_current_<room>` marker and, if present, pauses that room's media and clears the marker. The call back into Media Manager is a separate `script.turn_on` run, not a nested call, because Media Manager is a queued script and a nested call could deadlock.

**Activities.** `zen_dojotools_media_manager`'s `activity_apply` with `lock_room` sets the room to `Automation` through `room_control_set`, so the next occupancy event does not override the activity. `activity_end` returns it to `Auto` the same way.

**`role_audit`** (`area=` required, read-only, no cert) walks the `zen_mm_*`, `zen_lm_*`, and `zen_display_target` role labels, matching room membership by registry `area_id` or by room label, and reports:

| Finding | Means |
|---|---|
| `ambiguous_role` | Two or more holders of one role in a room, no `primary` label to break the tie |
| `area_mismatch` | A holder carries the room label but its registry `area_id` disagrees |
| `stale_state` | A holder is `unavailable` or `unknown` |

Entities labeled `zen_mm_shadow` (whole-house media groups and the like) are exempt from the ambiguity and mismatch checks.

**`zen_agent_disabled`.** Home Assistant has no template function for `disabled_by`. `zen_dojotools_ectoplasm`'s `entity_disable` and `entity_enable` tag and untag `zen_agent_disabled`, so "what has the agent disabled" stays answerable from labels, and `role_audit` can recommend disabling a duplicate holder without it vanishing from view.

---

*Related: [Room Manager v3 / REFLEX component reference](../components/room_manager_v3_reflex.md) for deployment steps and the label reference. [Room Manager (RoomReg)](../components/room_manager.md) for the spatial tool. [Chapter 15](15_live_state.md) for the concepts.*

<!-- nav -->
---

[← Developer Standards](23_developer_standards.md) · [Contents](00_toc.md) · [Appendices →](25_appendices.md)
<!-- /nav -->
