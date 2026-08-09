# Release Notes — 2026.9.0 'Steel Magnolia'

**Status:** Beta — ETA Saturday, 2026-09-05
**Branch:** `feat/2026.9.0`
**Base:** 2026.8.0 'Chef'

---

## Summary

Hospitality starts with knowing what's actually going on. Who's in the room, whether they're staying the night or just passing through, what they're allergic to, whether the door that just opened is a person walking in or a raccoon setting off a motion sensor. You can't host well on guesses.

Steel Magnolia is the release where the house learns to tell the difference — and then, having figured it out, gets out of the way. A good host doesn't narrate the effort. The room is just warm, the lights are just right, the guest's allergy is already flagged before anyone asks. Everything that makes that possible happens under the waterline. You don't get to see the duck paddling.

Room Manager v3 is the clearest expression of that: a full rewrite of how a room decides what state it's in, replacing a legacy `checking_timer` hack with a real wasp-in-a-box model, adding `entertaining_hold`/`guest_hold` as opt-in signals instead of a blanket toggle, and gating asleep-detection to nighttime so a hamper on a bed doesn't put a room to sleep at 2pm. None of it is visible day to day. It's just correct, quietly, every time — including the one incident that started it: an early redesign attempt broke the legacy Node-RED controller live, got fully reverted, and came back isolated by design so it can never happen again.

Twenty CRM's structured stay/appointment/service-request objects and Room Manager's shared occupant-prefs lookup are the other half of the same idea — Kitchen's allergen flagging and guest-aware meal planning both read from one source of truth about who's actually staying where, instead of five different systems each guessing separately.

The rest of this release is the plumbing that makes the front-of-house stuff trustworthy: ZQ-1 now tells you when a query silently found nothing because you asked the wrong question instead of because the data doesn't exist. Recorder stats stopped lying about energy sensors. The architecture docs got checked against the actual code, chapter by chapter, and corrected wherever they'd drifted from what shipped. None of it is glamorous. All of it is why the glamorous parts can be trusted.

---

## Inherited from 2026.8.0 'Chef'

Everything shipped in Chef — Taskmaster, the SP1 identity gate, Portainer container control, Kitchen's fulfillment/costing layer, Twenty CRM, Room Manager's guest/occupant-prefs lookup, and the full Reset Test hardening pass — is present here as a baseline. See [chef.md](chef.md) for the full writeup; this document covers what's new on top of it.

## Room Manager v3 — The Living Room-State Engine

Taskmaster got the glow-up in Chef — a new file, a new expediter identity, top billing in the release notes. Room Manager sat in the next chair the whole time, running the same `checking_timer` hack it shipped with, and it noticed. This release is Room Manager's turn in Truvy's chair: same bones, same name, walks out looking like it's never had a bad hair day in its life. See [Room Manager v3 & REFLEX](../architecture/22_Room_Manager_v3_REFLEX.md) for the design doc and [the operator's manual](../getting_started/room_manager_operators_manual.md) for the plain-language version.

**It started as an incident, not a makeover.** An early pass at converting real rooms to a new control select deleted and replaced `input_select.<room>_control` — still the household's actual production controller, owned by legacy Node-RED, running in parallel with v3 the entire time. That broke Node-RED live, mid-house. Full revert first, then the redesign below, built specifically so it can't happen that way again.

- **`room_control_manager` — the actual fix.** A new control surface, `select.<room>_control_manager`, with zero shared entity, label, or code path with legacy `input_select.<room>_control`. The domain-agnostic fallback chain that caused the original break (checking `input_text` before `input_select`) is gone entirely, not patched. Persistence moved off a per-room `input_text` helper onto a single cabinet-backed dict drawer, keyed by room — the select itself does nothing on `select_option` but fire an event; the dispatch script's listener is the sole writer. Named by ownership (`_control_manager`), not version (`_v3`), specifically so the *next* revision doesn't force another live-entity migration.
  - Caught a real orphaned-registry bug along the way: reusing the same `unique_id` across two revisions of the select (input_text-backed, then cabinet-backed) silently blocked re-registration entirely. If you ever see a template select sit at `unknown` with null `friendly_name`/attributes through multiple reloads, that symptom is this bug, not a template error — fixed here with a distinct `_cab` suffix.

- **Wasp-hold, actually wasp-in-a-box now.** The old mechanism armed a 2-minute `checking_timer` on a door open and showed `checking` while it ran — `checking` no longer exists as a producible state anywhere in the system. New mechanism: a `wasp_door` opening floors the room to `occupied` immediately and clears any latched wasp flag; motion firing while *every* `wasp_door` for the room is closed sets `wasp_flag_active` and the cascade shows `hold` instead of confidently resolving to `occupied`. The flag itself is a self-referencing template attribute, not another `input_boolean` — deliberately, to avoid a new per-room helper entity — and clears on exactly two events, never a timeout: a fresh door open, or a manual `room_control_manager` override.
  - `mode=label_discover`'s `group_warnings` now catches a lock+door pair tagged for the same opening and flags it — tag the door (`wasp_door`), never the lock. Caught live on a real shared entry where the lock was correctly tagged and the door itself was never in `trigger_entities` at all.

- **`entertaining_hold` / `guest_hold` — two more `hold` sources, opt-in by label, not a blanket toggle.** Legacy Node-RED held occupancy conservatively per-room the whole time `input_boolean.entertaining` was on — confirmed live that an HA reload's connectivity blip put five legacy rooms into Hold simultaneously, all at once, for no room-specific reason. The v3 equivalent scopes that same signal down to exactly the rooms that opted in: tag the *existing* `input_boolean.zen_entertaining` (or `zen_guest_mode`) with a purpose label plus the target room's own label. Nothing new created. Both outrank `occupied`/`vacant`, both outranked by `engaged`/`asleep`/manual override/emergency.

- **Asleep window — night gates the signal, doesn't replace it.** A direct `asleep`/`bed_occupancy` signal firing during the day — the real case that found this: a laundry hamper set on a bed tripped the bed sensor — previously auto-slept the room in broad daylight. Fixed with a default-on gate against the household's real scheduler-anchor clock (`zen_night_start`/`zen_am_start`), deliberately *not* `sensor.period_of_day`, a different sun-elevation-based sensor with no relation to those anchors that looks like a natural fit and isn't. Gated identically at three separate enforcement points — the live-gate, the edge-triggered timer-arming handler, and the periodic room-timer reconcile pass — because the reconcile pass independently re-derives asleep truthiness and would otherwise have silently bypassed the other two. Also fixed in the same pass: asleep-class room-timer default was 30 minutes, now a real 8 hours, closing a genuine format bug (`'00:%02d:00'` hardcoded 0 hours and silently broke for any duration ≥ 60 minutes — which 480 obviously is).

- **`autosleep_schedule` — for rooms where "night" isn't at night.** A community request: shift workers, or anyone whose sleep schedule doesn't track the house's clock. Tag any truthy-resolving entity (a toggle, a calendar, an HA `schedule.*` helper) with `autosleep_schedule` plus the room's own label, and it becomes **authoritative** for that room — fully replacing the night→wake window check rather than widening it. `asleep_window_disable` still outranks it if a room somehow carries both labels. Same label-existence idiom every other opt-in construct here already uses, no new mechanism.

- **`asleep_hold` — a third, independent path into Asleep.** Structurally identical to `entertaining_hold`/`guest_hold`, but feeds the `asleep` tier directly instead of `hold`. Tag a truthy-resolving entity with `asleep_hold` plus the room's label and the room reads `asleep` outright — no bed sensor, no trigger signal, no window check needed. Zero clock, zero decay: clears the instant the tagged entity goes false, or a manual override.

**Cascade order, current:**
```
emergency > manual override (room_control_manager) > asleep > engaged >
child-engaged > hold (wasp / entertaining / guest) > occupied (or fridge-door hold) > vacant
```

**Verified live** across the full incident-and-fix cycle: a real garage door/motion sequence produced the documented `vacant → occupied → hold` sequence correctly; `entertaining_hold`/`guest_hold` confirmed via tagged rooms with `last_trigger` correctly attributing each hold to its source boolean. The asleep-window gate hasn't been exercised against a full overnight cycle yet — built and reloaded same-day, no night had passed at verification time.

**Setup note:** six labels (`entertaining_hold`, `guest_hold`, `autosleep_disable`, `asleep_window_disable`, `autosleep_schedule`, `asleep_hold`) live in HA's label registry, not in git — the feature is a safe no-op until you create them: `zen_dojotools_labels mode=create label_list=["entertaining_hold","guest_hold","autosleep_disable","asleep_window_disable","autosleep_schedule","asleep_hold"] confirm=true`.

## Steel Magnolia Phase 7 — Manifest Audit

Manifest gains audit/inference fields and missing-label detection, closing the gap between what a household's tools claim to expose and what's actually wired up.

## Query & Recorder Correctness

- **ZQ-1 filter validation** — `filter_json` keys that don't exist (a stray `domains` instead of `domain`) used to vanish silently, producing a confidently-empty result indistinguishable from a real zero-hit query. Both `dry_run` and live responses now carry a `warning` field naming the bad key.
- **State-class-aware history stats** — requesting `mean`/`min`/`max` against a `total`/`total_increasing` energy sensor (SPAN, Emporia Vue) used to silently return empty buckets; the recorder call now branches on `state_class`. New `period_total_24h` gives a direct answer for "how much did this use today," cross-checked against recorder's own `sum` statistic and flagged when the two disagree by more than 2x — a real divergence caused by `statistic_id` re-registration on entity relabel, not a usage spike.

## Documentation

- **Architecture docs accuracy pass** — every chapter in `docs/architecture/` checked against current code by four parallel review agents, not just against each other. Stale entity/script names, fabricated event-type strings, and several sections describing design targets as if they were shipped behavior (identity_hash enforcement, Abbot-level ACL enforcement, the ch06/ch14 task-economy model) corrected or reframed.
- **Chapter 21 — Developer Taxonomy and Component Standards** and **Chapter 22 — Room Manager v3 & REFLEX** added.
- New Plant Codex docs for Emporia Vue and SPAN Panel circuit-level energy monitoring.
- Install docs gain the blueprint copy step Room Manager v3 requires.

---

*ZenOS-AI 2026.9.0 'Steel Magnolia' — the duck looks calm.*
