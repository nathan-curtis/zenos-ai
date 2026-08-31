# Release Notes — 2026.9.0 'Steel Magnolia'

**Status:** RC1 — ETA Saturday, 2026-09-05
**Branch:** `feat/2026.9.0`
**Base:** 2026.8.0 'Chef'

---

## Summary

Hospitality starts with knowing what's actually going on. Who's in the room, whether they're staying the night or just passing through, what they're allergic to, whether the door that just opened is a person walking in or a raccoon setting off a motion sensor. You can't host well on guesses.

Steel Magnolia is the release where the house learns to tell the difference — and then, having figured it out, gets out of the way. The room is already warm, the guest's allergy is already flagged before anyone asks, the task is already sitting in the queue before you'd have remembered to write it yourself.

A good host also knows how to say no. A locked door stays locked without anyone standing guard over it. A guest gets exactly the access a stay calls for, nothing assumed, nothing left open by accident. The house holds its boundaries the same quiet way it holds its warmth — steadily, automatically, without making a production of it.

Room Manager v3 is the clearest expression of the first half of that idea: a room that genuinely knows what state it's in by reading what's actually happening in the space right now, not by watching a clock. Twenty CRM's structured stay/appointment objects, and the arrival-prep and checkout-nudge scans built on top of them, are the second half — a guest's stay is tracked start to finish, prefs and all, and the house acts on that timeline instead of just logging it. Security Manager and the new identity-gate system are the boundary half: risky actions now require an explicit household certification, with the highest-risk ones asking live every time.

---

## Inherited from 2026.8.0 'Chef'

Everything shipped in Chef — Taskmaster, the SP1 identity gate, Portainer container control, Kitchen's fulfillment/costing layer, Twenty CRM, Room Manager's guest/occupant-prefs lookup, and the full Reset Test hardening pass — is present here as a baseline. See [chef.md](chef.md) for the full writeup; this document covers what's new on top of it.

## Room Manager v3 — The Living Room-State Engine

Room Manager reads a room the way a person would — not "has X minutes elapsed," but "what's actually happening in here right now." See [Room Manager v3 & REFLEX](../architecture/22_Room_Manager_v3_REFLEX.md) for the design doc and [the operator's manual](../getting_started/room_manager_operators_manual.md) for the plain-language version.

**Cascade order:**
```
emergency > manual override > asleep > engaged >
child-engaged > hold (wasp / entertaining / guest / presence) > occupied > vacant
```

- **Wasp-in-a-box.** A `wasp_door` opening floors the room to `occupied` immediately and clears any latched hold. Motion firing while every `wasp_door` for the room is closed sets the room to `hold` instead of confidently guessing `occupied` — something moved, but nothing came in or out, so the room holds its answer until a door tells it more. No timer, no self-latch: purely physical-space logic, gated behind an opt-in `wasp_enabled` label per room (rooms with an open archway instead of a real door would otherwise misfire constantly). Clears on a fresh door open or a manual override — never a timeout. An adjustable blind period (`wasp_blind_seconds`, default 5s) stops a quick door open/close cycle right after a release from re-latching immediately. `mode=label_discover`'s `group_warnings` catches a lock+door pair tagged for the same opening — tag the door, never the lock.
- **`entertaining_hold` / `guest_hold` / `presence_hold`** — opt-in hold sources by label, scoped to exactly the rooms that want them. Tag the household's existing `input_boolean.zen_entertaining`/`zen_guest_mode` (or a continuous-presence mmWave sensor for `presence_hold`) with a purpose label plus the target room's own label. All outrank `occupied`/`vacant`, all outranked by `engaged`/`asleep`/manual override/emergency. The generic `hold` label works the same way for anything else you want to plug in.
- **Asleep window.** A direct `asleep`/`bed_occupancy` signal only resolves to `asleep` inside the household's real scheduler-anchor clock — a hamper on a bed at 2pm doesn't put a room to sleep. Asleep-class room timers default to a real 8 hours.
- **`autosleep_schedule`** — for rooms where "night" isn't at night. Tag any truthy-resolving entity (a toggle, a calendar, an HA `schedule.*` helper) with `autosleep_schedule` plus the room's own label, and it becomes authoritative for that room, fully replacing the night→wake window check.
- **`asleep_hold`** — a third, independent path into Asleep, structurally identical to `entertaining_hold`/`guest_hold` but feeding `asleep` directly. Tag a truthy-resolving entity and the room reads `asleep` outright — no bed sensor, no window check needed.
- **Hold-release restarts the room's timer.** Previously, a room fell straight through to Vacant the instant any hold released, even if someone was still there. Now the room remembers what it was doing before the hold and picks back up where it left off (asleep is excluded by design — it releases plainly).
- **A generic `trouble` attribute** for anything electromechanical that wants to flag a room, plus native detection for SPAN circuit panels: a room's `trouble_active` attribute fires both on sustained overload (drawing over 80% of a breaker's rated capacity) and on direct confirmation that a breaker has actually tripped or been shed — SPAN reports both draw and real breaker position. A `trouble_last_trigger` attribute names the specific circuit and cause. This is an attribute overlay, never a cascade state — a hot breaker in the kitchen can't flip a sleeping bedroom's state.
- **New exposed attributes**: `motion_live_active` (raw live level-check), `smoke_active`, `leak_active`, `occupancy_evidence_source` (whether the last occupancy signal came from a physical device, an automation, or a person — observability only), plus the actual offending entity_id (not just a boolean) for garage/window-covering/exterior-lock conditions.
- **Paused and Emergency cascade unconditionally through nested rooms** — a safety all-stop or life-safety event in a child room can never be silently suppressed by the cosmetic child-cascade opt-out that governs ordinary occupied/engaged propagation.
- **Cross-room label sharing** is now a documented technique: one sensor's entity can carry multiple rooms' own labels when it's genuinely the shared boundary for several logical rooms — a single entry door for a nested suite, or one smoke detector covering several adjacent rooms.
- **`domain_routing` gains a `crm` entry**, making Twenty CRM's vendor/contact tooling discoverable through the path most callers use to find the right tool for a room.
- **Zone-mode disambiguation.** `zone_set`/`zone_remove`/`zones` renamed to `room_zone_set`/`room_zone_remove`/`room_zones` — the old names collided with the real HA `zone.*` geofencing tool.
- **New diagnostic modes**: `trigger_audit` (which classes are covered by live purpose-triggers vs. still need static wiring), `coverage_map` (a unified room-readiness view — label gaps, trigger coverage, dormant features, wasp-gate status), `wasp_enable` (read/write the per-room gate), and `reflex_last_fired` (cross-checks whether the scene that actually fired for a room matches what its current state expects — the fastest way to spot a stale scene or a manual override REFLEX doesn't know about).
- **`room_occupancy` cabinet mirror** — every real room-state transition also writes that room's full attribute set to the household cabinet, a read-surface companion to the native per-room sensors.
- **BLE-informed occupancy** (informational only this phase) and a purpose-neutral vibration signal class with an anomaly-gated "the load is done" detector for laundry-style use cases.
- **`room_timer_class`** (the internal tag tracking which decay timer a room is running) moved off a dedicated per-room helper entity onto the same cabinet object `room_control` already lives on — no functional change, just one less helper entity per room.

**Setup note:** six labels (`entertaining_hold`, `guest_hold`, `autosleep_disable`, `asleep_window_disable`, `autosleep_schedule`, `asleep_hold`) live in HA's label registry, not in git — every one of these features is a safe no-op until you create the label: `zen_dojotools_labels mode=create label_list=["entertaining_hold","guest_hold","autosleep_disable","asleep_window_disable","autosleep_schedule","asleep_hold"] confirm=true`.

## AutoVac — Roborock Support

A mid-cycle vacuum hardware swap (`xiaomi_miio` → core `roborock`) is now fully supported, closing every gap the swap exposed:

- Battery, wear, and consumables tracking all work correctly on the new integration (separate battery sensor, renamed/re-scaled wear sensors, a fallback chain that never reads missing data as 0%).
- Room-targeting, manual-run tracking (a run started from the vacuum's own app now gets tracked automatically), and camera/map coverage analysis all work on Roborock's real entity shapes.
- A dock genuinely docking twice at the end of one run (wash, then charge) no longer misreports as an untracked run.
- Vacuum errors now route through AlertManager, respecting quiet-hours policy instead of paging at 2am.
- `mode=clean` gains `dry_run` support, matching `run_elected`.
- **A severe fix**: the automatic room-election gate had silently blocked every eligible room from ever being auto-cleaned since it shipped. Vacant rooms now actually get elected.

## Security Manager — Full Parity with Legacy Automation

- **New arm/disarm behavior**, all off by default: the alarm panel can now drive `zen_home_mode` (and be driven by it), a real disarm can mark an arrival area occupied, an exterior-lock unlock can disarm from Home (never from Away — a thumb-turn isn't a credential the way a keypad code is), and the house can auto-arm on Night-Late or Away mode transitions.
- **Vacation wake-shift** — a household-wide toggle that shifts wake/morning anchors back an hour while a vacation calendar is active, independent of the calendar itself so nothing changes until you turn it on.
- **Roving-vacuum-while-armed handling finished**: starting a clean while the alarm is armed Away now asks live before downgrading to Home for the run, and restores Away only once the vacuum is genuinely done — a mop-wash pit-stop mid-run no longer triggers a premature restore.
- **Live security requests never silently wait for morning.** Clearing a Paused room, a certification grant or revoke, and disarming all now bypass the quiet-hours notification gate that correctly delays routine alerts — these get a human's attention immediately, any hour.
- A cabinet-write bug that could cause a configuration change to silently fail while reporting success has been fixed across every write path in this tool.

## ZenLux — Spook 5.2 and REFLEX Integration

- **Six new adjust-only light controls** (set/increase/decrease brightness, set color, set color temperature, set effect), available wherever Spook 5.2 is installed — opt in per call, current behavior is unchanged unless you ask for the adjust-only variant.
- **REFLEX now fires every scene through ZenLux** instead of bypassing it, so ZenLux's own room-lock, policy, and dusk/dawn guards apply consistently everywhere a scene fires — REFLEX included.
- **REFLEX's rehearsal mode (`reflex_dry_run`) is now correctly independent** of the master enable switch in every direction: off means nothing happens, dry-run simulates without ever touching a light, live means it actually fires.
- **`hold` now inherits the `occupied` scene** — a wasp-latched or held room is still someone being there, so it gets the same lighting treatment as occupied, not darkness.

## Hospitality Lifecycle — Arrival, Stay & Departure

A guest stay is a real, structured object from the moment you tell the house about it to the moment the room's turned over — not a calendar entry, not a note somebody has to remember to act on.

- **Arrival prep, before anyone's at the door.** Once a stay is on the books, the house checks in on it as the arrival window approaches — allergen/music/spa prefs pulled from the guest's own record, a task raised for whoever's handling turnover.
- **Checkout, with a heads-up instead of a surprise.** A nudge lands as checkout approaches, tagged to the room, worded in plain local time.
- **One source of truth for who's actually here.** Kitchen's allergen flagging, guest-aware meal planning, and the arrival/checkout scans all read from the same shared occupant-prefs lookup.
- **Ask the house directly.** A stay's status is a live, queryable fact — no need to wait on a notification to know whether anyone's checking out today.
- **Every reserved stay now correctly ages out** once its checkout date passes without being explicitly released, instead of reporting as active indefinitely.
- Guest identity and permission delegation — an owner granting a guest their own scoped access rather than one household-wide guest flag — is out of scope this release, held for the next one.

## ZenZork "Chapter 1" — Loot, Quests, Book-Lore, and a Chapter Release Model

ZenZork ships its first real content chapter, SoftDisk-style — one bundled release, not staggered feature drips. See [the unofficial player's manual](../getting_started/zenzork_manual_unofficial.md) for the in-universe guide, and the [devkit doc](../scripts/zenzork_devkit.md) if you're extending it.

- **Real weighted loot table** — 13 items across 5 rarity tiers.
- **Book-lore sequence** — a data-driven 12-entry ordered sequence with a shared 24h cooldown, checked against the real source books.
- **Quest markers: 3 → 15**, 12 of them table-driven. `missing_clock` surfaces a real household setup gap (a room supporting the asleep tier with no `tv_sleep_timer` helper) as a quest.
- **Carl's Left Sock** — a real registered landmark via Room Manager, with a one-time achievement.
- **Game Genie cheat codes** — 24 codes, two independent gates each.
- **`mode=chapters`** — status view plus a catch-up path for new household members to claim released-but-unearned content.
- North-calibration (`mode=setup answer=calibrate=X`) now correctly persists — it had never actually worked.
- Player-name resolution and portal traversal both had real bugs, now fixed: a session's own player name is correctly remembered after the first action, and a locked or closed portal actually blocks passage instead of letting you through.

### v1.8.0 — Training, Real Achievements, Threats & Combat

- **Training onramp** — a `training_quest` auto-seeds on every genuinely new session.
- **Real persisted achievements** replacing quest-catalog flavor text.
- **A data-driven turn-based threat engine**, ticking only on real turns, scaling with difficulty and room count.
- **Lite-5e `mode=attack` combat**, non-fatal always — a lost fight ends the session, not the character.
- **Household-systems achievements**, checked against real live entity presence.
- **Rare collectible cartridges** and a late-game capstone requiring both plus the full book-lore chain.
- **`mode=talk`** — always refuses. A deliberate no-NPCs gag, not a missing feature.

## Identity Gates — Certification for Risky Actuations

Certain actuations are risky enough that "the agent is allowed to call this tool" isn't sufficient on its own. Locks, covers, Security Manager, container control, Room Manager, ZenLux, climate, ZenZork, and the spa tool now gate their riskiest actions behind a certification the household grants explicitly — with the highest-risk actions additionally requiring a fresh live yes/no every single time, no standing exception.

- **Locks** — new tool, room-targetable lock/unlock. Full Keymaster code-slot detail on inspect; PINs are never surfaced, only whether a slot is set.
- **Covers** — closing any cover is cert-only; opening a barrier-class cover (garage door, exterior door, gate) requires the cert and a fresh live ack every call; ordinary window coverings stay cert-only.
- **Security Manager** — arming and alert-policy changes require the cert; disarming requires the cert and a fresh live ack every time.
- **Climate** — new cert-only gate (no live-ack tier). Also gains general-purpose `fan.*` control — speed, preset, oscillation, direction.
- **Room Manager** — two certifications covering roughly 35 modes: structural topology edits, and behavioral/state-machine writes. Moving a room out of Paused requires a fresh live ack every time. `scene_handling` (staging a whole room's scene) is now actually enforced, not just declared.
- **ZenLux** — gains its own identity gate alongside real `switch.*` control, not just lights.
- **ZenZork and the spa tool** — both were actuating real hardware (locks, covers, jets, water temperature) with zero identity check; both now route through the same gated tools as everywhere else. ZenZork's in-game actions run in preview mode by default, since a game session has no legitimate path to a real live acknowledgment.
- **A scoped admin override** lets a household admin exempt specific targets (a specific door, a specific container) from the every-call live-ack requirement, without weakening the underlying cert requirement.
- **A real deny primitive.** `cert_scope` entries can now explicitly deny a specific target, hard-blocked ahead of the certification check — distinct from an unscoped target, which still asks normally.
- **CertAdmin** — a new admin-only tool centralizes `cert_grant`/`cert_revoke`/`cert_list`, never reachable by an agent. Lets an admin waive the live-ack requirement for a specific cert (e.g. "don't ask every time I open the garage") behind three independent confirmation layers.
- **A canonical response shape (`envelope()`)** separating "did the call succeed" from "what did the tool find" — piloted on two tools this release.

## Manifest Audit & Shared Scanning

- Five of Manifest's own introspection modes now share one collector instead of each hand-rolling its own scan loop — same behavior, one audited pattern instead of five drifting copies.
- Every tool now reports a `dependencies=[]` field on `tool_manifest()`; a new `mode=toolmap` walks it into a declared dependency graph.
- Every audit mode now correctly scans the full tool roster, including the admintools tier (previously silently excluded).
- `mount_status` now correctly reports each cabinet's real mount state instead of always reporting unmounted.
- Trapper Keeper gains drift detection for a stopgap tracking field, pending its migration to real self-registration.

## Query, Recorder & Prompt Correctness

- **ZQ-1 filter validation** — a `filter_json` key that doesn't exist now returns a `warning` field naming it, instead of silently producing a confidently-empty result.
- **State-class-aware history stats** — energy sensors (SPAN, Emporia Vue) now report correctly regardless of state class. New `period_total_24h` gives a direct, cross-checked answer for "how much did this use today."
- **`sensor.home_overview` gains a real `rooms` attribute** — Friday's prompt now actually receives per-room state instead of an empty object.

## Also Fixed

- An AI persona's identity could be silently reset on bootstrap if its essence used the newer three-layer schema; fixed, and the affected reference install's essence was restored from its automatic backup.
- A self-sustaining alert loop between two summarizer components, each narrating alarm about the other's own auto-filed tickets indefinitely, is broken.
- Taskmaster gains a `catch_up` mode for rolling stale recurring items to today, and conductor items now count toward the briefing's overdue/stale totals.
- Kitchen's new `recipes_find` mode finds recipes by tag, not just name/description — Mealie's own search index doesn't cover tags, so a tag-only match was previously unfindable. A tag-hydration bug that could surface a misleading "Recipe already exists" error is also fixed.
- Agent health no longer follows the summarizer pipeline's own health into a false `warn` when the agent itself is fully bootable.
- New Ectoplasm modes (`automation_snooze`/`automation_turn_on_for`) for Spook 5.1+ installs.
- A fresh-install default no longer silently overwrites an already-configured value in AdminTools.

## Documentation

- **Architecture docs accuracy pass** — every chapter in `docs/architecture/` checked against current code.
- **Chapter 21 — Developer Taxonomy and Component Standards** and **Chapter 22 — Room Manager v3 & REFLEX** added.
- New Plant Codex docs for Emporia Vue and SPAN Panel circuit-level energy monitoring.
- Install docs gain the blueprint copy step Room Manager v3 requires.

---

*ZenOS-AI 2026.9.0 'Steel Magnolia' — the duck looks calm, and the fence is exactly where it's supposed to be.*

---

This release is dedicated to Ms. Dolly Parton, January 19, 1946 – August 25, 2026.

*"Yes, Truvy, it does take some effort to look like this."*
