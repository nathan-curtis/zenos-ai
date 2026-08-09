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

Full rewrite of the per-room state cascade. See [Room Manager v3 & REFLEX](../architecture/22_Room_Manager_v3_REFLEX.md) for the design doc and [the operator's manual](../getting_started/room_manager_operators_manual.md) for the plain-language version.

- **`room_control_manager`** — a new control surface, fully isolated from the legacy `input_select.<room>_control` that Node-RED still owns. No shared entity, label, or code path between them, permanently.
- **Wasp-hold, rewritten.** The old 2-minute `checking_timer` is gone. A room now floors to `occupied` the instant a `wasp_door` opens, and motion with every `wasp_door` closed sets a real wasp-flag instead of running a clock — cleared only by a fresh door-open or a manual override, never a timeout.
- **`entertaining_hold` / `guest_hold`** — opt-in per room by label, not a blanket toggle. Tag an existing `input_boolean` with a purpose label plus the room's own label; nothing new to create.
- **Asleep window** — a direct sleep signal during the day (a hamper on a bed, say) no longer puts a room to sleep. Gated against the household's real scheduler-anchor clock, enforced at three separate points so nothing can slip past it.

**Setup note:** four labels (`entertaining_hold`, `guest_hold`, `autosleep_disable`, `asleep_window_disable`) live in HA's label registry, not in git — the feature is a safe no-op until you create them.

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
