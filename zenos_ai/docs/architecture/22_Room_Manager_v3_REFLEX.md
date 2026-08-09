# **22. Room Manager v3 & REFLEX: The Living Room-State Engine**

Chapter 11 (RoomState and Perception) describes, in the abstract, a real-time
semantic map of the house — a structured object per room that tells Friday
what the raw HA graph *means*, without doing any cognition of its own. This
chapter documents the concrete implementation of that idea: **Room Manager
v3**, and the reactive layer riding on top of it, **REFLEX**.

Where Chapter 11 draws the boundary ("RoomState must never do cognition, Room
Manager must never alter RoomState"), this chapter is the part that actually
computes the state, decays it, reacts to it, and writes the labels that make
the whole thing self-assembling. It is not the same tool as `components/room_manager.md`
(RoomReg — spatial topology, egress, emergency snapshots). Room Manager v3 is
a different, complementary system: RoomReg answers *"what is this room, and
how does it connect to the house physically"*; Room Manager v3 answers
*"what is this room doing, right now, and what should happen because of it."*

---

## **22.1 Design Philosophy**

Prior generations of this system (legacy, v2) required per-room wiring: new
automation, new blueprint call, new hardcoded entity list, for every room,
every time a feature was added. Room Manager v3's governing principle is that
the room blueprint supports every possible configuration on its own — a room
gains a feature by adding a timer and a label, never by adding code.
Concretely, that means:

* **Label-intersection, not naming convention, for anything that requires human judgment** (which entity is *this* room's motion sensor, occupied timer, control override). A room opts into a feature by tagging one entity with one label — the label carries both the *class* (`motion`, `room_timer`, `vent_fan`...) and the *room* (the area's own label, matching its `area_id` 1:1 by convention).
* **Naming convention, not labels, for anything fully deterministic at deploy time** — timer/duration helpers whose entity_id is already fixed the moment you name the helper in a room's package file (`timer.<room>_tv_sleep_timer`, `input_number.<room>_fan_delay_minutes`). Adding a label for something that's already 100% predictable from the room slug is pure taxonomy bloat.
* **Untagged = safe no-op, everywhere.** No feature in this system ever assumes a helper exists. Every tier, every opt-in construct, checks for its own helper and simply doesn't activate if absent. A room with zero helpers still gets a working `vacant`/`occupied` cascade from motion alone.
* **One automation. One script.** The entire dispatch layer — signal routing, cleaning sync, REFLEX scene resolution, nightlight, control-burnout safety net, TV sleep timer, vent fan automation, and the room-state self-labeling bootstrap — lives in exactly one automation entity (`automation.zenos_room_manager_dispatch`) and one script entity (`script.zen_reflex_controller`, mode-dispatched). See §22.6 for the mechanics of this consolidation.

---

## **22.2 The Cascade: `room_state.yaml`**

Every deployed room gets exactly one template sensor, built from the shared
`blueprints/template/zenos/room_state.yaml` blueprint. This sensor **is**
that room's RoomState object from Chapter 11, concretely realized — its
`state` attribute is the room's resolved tier, and its other attributes are
the structured "what does this mean" decode of the room's raw signals.

### Cascade order (highest wins)

```
emergency > manual override (room_control_manager) > engaged > asleep >
child-engaged > hold (wasp / entertaining / guest) > occupied (or hold) > vacant
```

`checking`/`checking_timer` do not exist in this system — see the wasp-hold
section below (§22.9) for the live mechanism that replaces what a
timer-based "checking" tier would otherwise do. A room either resolves
cleanly or lands in `hold`.

| Tier | Meaning | Source |
|---|---|---|
| `emergency` | Something in this room needs immediate attention (smoke/CO/moisture/siren). Wins even while paused — life-safety visibility is never silenced. | `emergency_latch` (input_boolean, human/agent ack-only clear) |
| manual override → `paused`/`automation`/`cleaning`/`vacant`/`occupied`/`engaged`/`asleep`/`hold` | A human/agent has explicitly picked one of these via `room_control_manager` (the select). Outranks everything below it except emergency; releases back to `Auto`, or is superseded by a fresh `wasp_door` open (see §22.9). | `room_control_manager` select |
| `cleaning` | A vacuum is actively working this room. | manual override, set by the Cleaning Dispatcher |
| `engaged` | Active, direct use — someone is *doing something* here right now (media playing, a monitored desk dock in use). Outranks mere presence. | live signal, or the shared room_timer decaying in class `engaged` |
| `asleep` | Someone's asleep here. Own direct signal always beats a child room cascading up — using the ensuite mid-sleep must never wake the parent. Auto-fire is night→wake window-gated by default — see §22.9. | live signal (window-gated), room_timer decaying in class `asleep` (8h default), or manual override via `room_control_manager` (never window-gated) |
| `hold` | Unresolved presence or a deliberate conservative-hold policy — three independent sources, same visible state, distinguishable via `last_trigger`. See §22.9. | wasp_flag (self-latched), entertaining_hold, guest_hold |
| `occupied` | Presence detected, no specific activity signal, or a child room is non-vacant, or a `hold`-labeled entity is active ("fridge door mode" — floors at Occupied, no clock, instant fall-through on close). | live signal, room_timer decaying in class `occupied`, child cascade, or hold |
| `vacant` | Nothing is true. Default. | — |

### The shared room timer

Legacy and early v3 each tier (occupied/engaged/asleep) had its own decay
timer. The current design collapses this to **one** shared `room_timer` +
a paired `room_timer_class` select recording which tier currently owns the
clock — a room is never simultaneously "decaying occupied" and "decaying
asleep," so one timer covers all three.
Ownership/takeover rules for who may (re)start the shared clock live in the
Signal Dispatcher (§22.3), not in the cascade sensor itself — the sensor
only *reads* whichever class currently holds it.

A room opts into decay-backed occupied/engaged/asleep at all by tagging one
timer `room_timer` (+ paired select `room_timer_class`, options exactly
`[occupied, engaged, asleep]`). Untagged room = all three tiers are
live-signal-only, no grace period — a safe, simpler no-op, not a broken
state.

### Child rooms (cascade-up)

An ensuite/sub-room's state contributes to its parent's, opt-out per room
(`child_link_disable` input_boolean), default on. A child resolved to
`engaged` promotes the parent to `engaged`. A child in any other non-vacant,
non-checking, non-paused state floors the parent at (at least) `occupied` —
indefinitely, for as long as the child stays active, no timer needed. The
one hard exception: the parent's own *direct* `asleep` always wins outright,
regardless of what any child is doing.

### Explainability

Every tier that can be true carries a `*_last_trigger` attribute —
`{entity_id, friendly_name, last_changed}` for whichever source entity most
recently caused it, or a `{reason, timer_entity, timer_last_changed}` dict if
it's true only via timer decay with nothing currently live. A top-level
`last_trigger` mirrors the same logic for whichever tier actually won the
cascade. This exists specifically so "why does the office say Engaged right
now" has a real, inspectable answer instead of a black box.

---

## **22.3 REFLEX: The Event Bus and Reactive Layer**

REFLEX is the two-stage, event-decoupled reactive system that turns a room's
resolved state into actual effects — scenes, nightlights, cleaning
coordination. Every resolver sensor change fires a structured event; a
separate listener captures that event and, if a matching scene exists,
fires it. The two stages are decoupled by design rather than combined into
one reactive automation.

**Stage 1 (Emitter)** lives *inside* `room_state.yaml` itself, not as a
separate automation — every room deployed via the shared blueprint gets
REFLEX emission automatically, zero extra per-room wiring, ever. On a REAL
transition (`_resolved_state` differs from the sensor's own previous value),
it fires a structured `zen_event` with `kind: room_state_changed` via
`zen_dojotools_event_emitter` — not HA's raw `event.fire` (not a registered
service on this install), and not an API end-run.

**Stage 2 (Resolver)**, inside `zen_room_manager_dispatch.yaml`, listens for
that event and resolves a scene: `label_entities(scene_<state>)` intersect
`label_entities(<room>)` intersect `label_entities(<home_mode_daypart>)`,
first match wins, falling back to room+state only (drop the mode filter) if
nothing matches with mode applied. No match at all is a safe no-op, never a
guess. Gated by `reflex_enable` (or `reflex_dry_run`, which logs what
*would* have fired instead of firing it — useful for verifying event payloads
before ever touching a light).

**Nightlight** is a separate, opt-in construct riding the same `motion`
purpose-trigger: while a room's v3 state is `asleep`, a motion *edge*
(not the level — `occupied` can sit true all night; motion only pulses on
real action) fires a `scene_nightlight` scene and starts a `nightlight_timer`.
On expiry, the room's normal asleep scene re-fires. The room's actual v3
`state` never changes during this — nightlight is scene-layer only.

### Scene label taxonomy

Distinct from the v2 input-signal labels (`zen_occupied`, `zen_engaged`,
`zen_asleep`, which tag *source* entities that drive occupancy) — these tag
*output* scenes: `scene_vacant`, `scene_checking`, `scene_occupied`,
`scene_engaged`, `scene_asleep`, `scene_paused`, `scene_automation`,
`scene_cleaning`, `scene_emergency`, `scene_nightlight`. Only the tiers that
actually have scenes need tagging — an untagged tier is a safe no-op scene
resolution, not an error.

---

## **22.4 The Signal Dispatcher: Label Reverse-Lookup**

The Signal Dispatcher is REFLEX's front door for raw HA events. It listens
globally for a fixed set of "purpose triggers" — `motion.detected`,
`occupancy.detected`, `lock.locked/unlocked`, `media_player.started_playing/
stopped_playing/paused_playing`, `air_quality.smoke_detected`,
`air_quality.co_detected`, `moisture.detected`, `siren.turned_on`,
`door.opened/closed`, `button.pressed` — each scoped by `target: label_id:`,
so it only fires for entities that are actually tagged for one of these
classes, never a blanket "every motion sensor in the house."

On a fire, it does a **label reverse-lookup**: reads the firing entity's own
labels, finds whichever one is a real HA area (`area_name(l) is not none`),
and that's the room — no hardcoded room allowlist, any room whose label
matches its own `area_id` just works, current or future construction. It
then intersects that room's full label set against the class of signal that
fired (smoke/CO/moisture/siren → arm `emergency_latch`; `asleep`/
`bed_occupancy` → start the shared timer in class `asleep`, gated by the
night→wake window and the two opt-out labels, see §22.9; `motion`/
`occupied` → start it in class `occupied`, and — if the room has
`wasp_door` tagged with none currently open — the wasp-flag corroboration
check, §22.9; `engaged` ending → start it in class `engaged`; `wasp_door`
opening → floors the room to `occupied` via the same room_timer mechanism
and clears any latched wasp flag) and writes exactly the helper that class
needs, if tagged, otherwise no-ops. `checking_timer` does not exist in this
system — nothing starts or reads one.

---

## **22.5 Opt-In Constructs**

Three features are opt-in per room, discovered without any per-room
`use_blueprint:` call — a room activates each purely by having the right
helper entity present, checked by existence at fire time:

| Construct | Activation | Behavior |
|---|---|---|
| **Control Burnout** | Tag a timer `control_burnout` + the room's label | Safety net for `room_control_manager`'s `Automation` tier only. If a room sits in `Automation` and its burnout timer expires without being released, it auto-reverts to `Auto`. Stateless resync every fire — self-healing, no drift risk. `Cleaning` self-clears via the Cleaning Dispatcher; `Paused` is intentionally sticky, human-ack-only, never touched here. |
| **TV Sleep Timer** | Naming convention: `timer.<room>_tv_sleep_timer` (+ optional `input_number.<room>_tv_sleep_minutes`) | A media edge while the room is `asleep` (re)starts this timer, separate from the room's own hours-scale asleep timer — it governs one device, not room occupancy. On expiry: turns off the room's TV/media targets and hands off to the room's own `asleep_timer`. |
| **Vent Fan Auto On/Off** | Tag a fan/switch `vent_fan` + the room's label (+ optional naming-convention duration knobs `input_number.<room>_fan_delay_minutes` / `_fan_min_runtime_seconds`) | Rides the same REFLEX event bus as scene resolution — on a transition to `occupied`, waits a delay then turns the fan on if still occupied; on leaving `occupied`, waits a minimum runtime then turns it off if still not occupied. A live re-check after the delay, not a fire-and-forget — avoids acting on stale state. |
| **Wasp-Hold** | Tag the entry `wasp_door` + the room's label | Self-latching — no helper to create. See §22.9 for the full mechanism. |
| **Entertaining Hold** | Tag `input_boolean.zen_entertaining` with label `entertaining_hold` + the room's label | See §22.9. |
| **Guest Hold** | Tag `input_boolean.zen_guest_mode` with label `guest_hold` + the room's label | See §22.9. |
| **Autosleep Disable** | Tag anything carrying the room's label with `autosleep_disable` | Kills automatic asleep-firing for that room; manual `room_control_manager` "Asleep" is unaffected. |
| **Asleep Window Disable** | Tag anything carrying the room's label with `asleep_window_disable` | Removes the night→wake restriction on automatic asleep-firing for that room. See §22.9. |

---

## **22.6 Dispatch Layer: One Automation, One Script**

The dispatch layer — signal routing, cleaning sync, REFLEX scene resolution,
nightlight, the control-burnout safety net, the TV sleep timer, vent fan
automation, and the room-state self-labeling bootstrap — is consolidated
into exactly one automation entity and one script entity, rather than one
automation per feature or a per-room `use_blueprint:` call for anything
beyond the core cascade.

* **One automation** (`automation.zenos_room_manager_dispatch`, `mode:
  parallel, max: 30`) carries every trigger the dispatch layer needs, each
  tagged with a `trigger.id`, and branches internally via a top-level
  `choose:` keyed on that id. `mode: parallel` is required because Vent
  Fan's multi-minute delay would otherwise queue/block every other trigger
  type behind it in a single-instance automation — every other branch is
  either a stateless resync (idempotent; a duplicate concurrent pass
  converges to the same answer) or keyed to the one entity_id in that
  specific event's payload, so parallel execution is safe.
* **One script** (`script.zen_reflex_controller`), mode-dispatched exactly
  like the dojotools scripts (`mode=signal_dispatch` /
  `room_timer_reconcile` / `cleaning_control_resync` /
  `control_burnout_resync` / `self_label_resync`).
* **Room-state self-labeling** (`mode=self_label_resync`) is the bootstrap
  step that tags a room's state sensor with `zen_room_state` + its room
  label, and tags each of that room's helper entities (`room_control`,
  `room_timer`, `room_timer_class`, `asleep_minutes`, `vent_fan`) with
  their class label. It is driven by `areas()` rather than per-room inputs:
  the sensor's entity_id is discovered by existence-checking the two
  conventions observed across deployed rooms (`sensor.<room>_state` or
  `sensor.<room>_room_state`), and the tag set is a fixed, closed list of
  (label, naming convention) pairs applied identically everywhere.

**Design constraint:** existence checks in this script test one candidate
entity_id at a time (`states[entity_id] is not none`). HA's Jinja sandbox
has a hard output-size ceiling (262,144 characters); materializing the full
house's entity list into a single Jinja variable to test membership against
it is unsafe at whole-house scale and must never be done here.

---

## **22.7 Division of Responsibility (extending Chapter 11)**

Chapter 11 draws the line: *RoomState must never do cognition, Room Manager
must never alter RoomState.* Room Manager v3's concrete pieces sit precisely
on that line:

* **`room_state.yaml`'s cascade** is the RoomState layer, concretely
  realized — declarative, no inference, no prediction. It resolves a tier
  from label-intersection truthiness and nothing else.
* **REFLEX and the opt-in constructs** are the reactive layer that consumes
  RoomState's output and acts on it — scene resolution, nightlight, vent
  fan, TV sleep. They read `room_state`'s `state` attribute but never write
  to it.
* **RoomReg** (`zen_dojotools_room_manager`, documented separately in
  `components/room_manager.md`) is the *cognitive interpretation* layer
  Chapter 11 refers to as "Room Manager" in the abstract — spatial topology,
  egress routing, emergency snapshots, whole-house `home_overview`. It reads
  the same HA graph but answers a structurally different question ("what is
  this room, physically") than v3's cascade answers ("what is this room
  doing, right now").

---

## **22.8 Agent-Facing Summary**

For an agent orienting itself to a room-aware task, the short version:

* Every deployed room has exactly one state sensor (`sensor.<room>_state`
  or `sensor.<room>_room_state`) whose `state` attribute is one of
  `emergency / paused / automation / cleaning / engaged / asleep / hold
  / occupied / vacant`. (`checking` does not exist in this system.)
  This is the authoritative answer to "what is this
  room doing right now" — don't infer it from raw motion/media entities
  directly, read the sensor. `hold` has three independent sources (wasp
  ambiguity, entertaining, guest mode), distinguishable via `last_trigger`
  — see §22.9.
* The sensor's attributes (`*_active`, `*_last_trigger`) explain *why* the
  state is what it is — useful for any response that needs to justify a
  suggestion ("the office is Engaged because Stormbreaker is actively
  playing media").
* `zen_dojotools_room_manager` (RoomReg) is the tool to call for spatial
  questions — topology, egress, emergency, whole-house overview. It is a
  **different system** from Room Manager v3/REFLEX; don't conflate the two
  when a user says "room manager."
* REFLEX itself has no MCP tool surface — it runs autonomously. An operator
  interacts with it via the room's `room_control_manager` select and HA
  labels, not through a chat-callable mode. See
  `components/room_manager_v3_reflex.md` for the practical label reference
  and deployment steps.

---

## **22.9 Control Isolation, Wasp-Hold, and the Asleep Window**

### `room_control_manager` — the room's control surface

Room Manager v3 has its own control select, `select.<room>_control_manager`,
fully isolated from the household's legacy Node-RED occupancy system, which
runs in parallel and remains the production controller for
`input_select.<room>_control`. The two share no entity, no label, and no
code path:

* Persistence lives in the household cabinet's `room_control_manager`
  drawer (a single dict, keyed by room) — not a per-room `input_text`
  helper. Self-latching template attributes and cabinet storage are used
  throughout this system specifically to avoid spinning up new HA helper
  entities where an existing mechanism already holds state.
* The select's `select_option:` does nothing but fire a
  `room_control_request` event — `zen_room_manager_dispatch.yaml`'s
  listener is the sole writer of the cabinet drawer.
* `self_label_resync` tags `input_select.<room>_control` with label
  `room_control`, and `select.<room>_control_manager` with a separate
  label, `room_control_manager`. Neither reads, writes, or labels the
  other's entity.
* `room_state.yaml`'s manual-override tier (`_manual_override_hi`) reads
  `room_control_manager` exclusively.
* Naming is by ownership, not version (`_control_manager`, not `_v3`) —
  a version-numbered name becomes a portability problem the next time
  this gets revised.

### Wasp-hold: live wasp-in-a-box semantics, no timer

A `wasp_door` opening floors the room to `occupied` immediately (via the
same room_timer mechanism motion/occupied use) and clears any latched wasp
flag — a door opening is itself presence-adjacent evidence. If motion fires
while **every** `wasp_door` for the room is closed — no door transition to
corroborate how or when someone got in — that's unresolved presence, not
confirmed presence: the room's `wasp_flag_active` attribute latches `true`,
and the cascade shows `hold` instead of confidently collapsing to
`occupied`.

The latch is a self-referencing template attribute
(`this.attributes.get('wasp_flag_active', false)`) inside `room_state.yaml`'s
own trigger-based template, not an `input_boolean`. It clears on exactly two
events, never on a timeout: a fresh `wasp_door` open, or a human/agent
forcing the room via `room_control_manager`. An unresolved room stays
unresolved until something actually resolves it.

**Anti-pattern**: a lock and a door for the same physical opening must
never both be candidates for the same signal — tag the *door*
(`wasp_door`), never the lock (`privacy_door`/`ext_lock`, security-exposure
only; lock state doesn't track threshold-crossing). `mode=label_discover`'s
`group_warnings` detects this pairing and flags it explicitly.

### `entertaining_hold` and `guest_hold` — legacy parity, opt-in by label

Legacy Node-RED holds occupancy more conservatively room-by-room while
`input_boolean.entertaining` is on. The v3 equivalent is two independent
`hold` sources, each opted in per room by label, not a blanket toggle: tag
the existing `input_boolean.zen_entertaining` (or `zen_guest_mode`) with a
purpose label (`entertaining_hold` / `guest_hold`) plus the target room's
own label. Nothing new is created — a room whose label isn't also on that
shared entity is unaffected. Both outrank `occupied`/`vacant` but are
outranked by `engaged`/`asleep`/manual override/emergency.

### The asleep window — night through wake, opt-out by label

A direct `asleep`/`bed_occupancy` signal is gated to fire automatically only
between the `night` and `wake` scheduler anchors
(`input_datetime.zen_night_start` / `zen_am_start` — not
`sensor.period_of_day`, a different, sun-elevation-based sensor unrelated
to these anchors). Outside that window the signal is silently ignored, at
all three enforcement points: the live-gate in `room_state.yaml`, the
edge-triggered timer-arming handler, and the periodic room-timer reconcile
pass in `zen_room_manager_dispatch.yaml`.

* `autosleep_disable` (label, tag anything carrying the room's own label):
  kills automatic asleep-firing entirely for that room. The manual
  `room_control_manager` "Asleep" pick is a wholly separate mechanism and
  is never time-gated.
* `asleep_window_disable` (label): keeps autosleep active but removes the
  night-only restriction, firing any time of day.

The asleep-class room_timer's default duration is 8 hours (not 30 minutes),
matching genuine overnight sleep.

---

*Related: [Room Manager (RoomReg)](../components/room_manager.md) — the
spatial topology tool. [Room Manager v3 / REFLEX — Component Reference](../components/room_manager_v3_reflex.md)
— practical deployment and label reference. [RoomState and Perception](11_RoomState_and_Perception.md)
— the theoretical layer this chapter concretizes.*
