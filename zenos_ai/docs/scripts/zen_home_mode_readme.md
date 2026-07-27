# Zen Home Mode — 4.5.6

*Eight-state home presence and time-of-day state machine*

---

## Overview

Zen Home Mode is the ambient context layer for ZenOS-AI. It tracks what the household is doing — sleeping, waking, at home, away — and makes that context available to Friday, the Scheduler, and Kung Fu components.

The state machine has **8 modes** and two auxiliary flags. Transitions happen automatically on schedule, on presence change, and on HA restart. The schedule is fully configurable via time helpers — no code changes required.

---

## Entities

### State

| Entity | Type | Description |
|---|---|---|
| `input_select.zen_home_mode` | input_select | Canonical 8-state home mode selector |
| `sensor.zen_home_mode` | sensor | Mirror of `input_select.zen_home_mode` with per-state icons |

### Flags

| Entity | Default | Description |
|---|---|---|
| `input_boolean.zen_guest_mode` | off | Guest presence — turns on when non-household guests are home |
| `input_boolean.zen_entertaining` | off | Hosting/entertaining flag — elevated social context |

### Time Windows

| Entity | Default | Description |
|---|---|---|
| `binary_sensor.zen_quiet_hours` | — | On when current time falls within quiet hours (midnight-wrap safe) |
| `binary_sensor.zen_work_hours` | — | On when current time falls within work hours (no wrap) |

---

## The 8 Home Modes

| Mode | Typical Time | Context |
|---|---|---|
| `Home-Wake` | Wake anchor → Morning anchor | Morning transition, household waking up |
| `Home-Morning` | Morning anchor → Daytime anchor | Settled morning routine |
| `Home` | Daytime anchor → Evening anchor | Normal daytime operations |
| `Home-Evening` | Evening anchor → Night anchor | Wind-down, dinner, social |
| `Night` | Night anchor → Night-Late anchor | Quiet evening |
| `Night-Late` | Night-Late anchor → Wake anchor | Very late / overnight |
| `Away` | zone.home drops to 0 | No one home — set automatically |
| `Paused` | Manual | Temporary hold — scheduled transitions skip while active |

`Away` and `Paused` are **hold states** — automatic time-based transitions do not fire while either is active. Presence return releases `Away` automatically. `Paused` requires manual release (set a different mode, or call `zen_home_mode_release`).

---

## Schedule Anchors

Six `input_datetime` helpers control when mode transitions fire. All are time-only (no date). Defaults:

| Helper | Default | Mode it triggers |
|---|---|---|
| `input_datetime.zen_am_start` | 06:00 | → `Home-Wake` |
| `input_datetime.zen_morning_start` | 08:00 | → `Home-Morning` |
| `input_datetime.zen_daytime_start` | 10:00 | → `Home` |
| `input_datetime.zen_evening_start` | 17:00 | → `Home-Evening` |
| `input_datetime.zen_night_start` | 21:00 | → `Night` |
| `input_datetime.zen_late_night_start` | 23:00 | → `Night-Late` |

Change any anchor in Settings → Helpers → the automation fires on the new time immediately with no restart needed.

### Meal-Time Helpers (Not Schedule Anchors)

Two additional `input_datetime` helpers — `input_datetime.zen_lunch_time` (default 12:00) and `input_datetime.zen_snack_time` (default 15:00) — exist alongside the six schedule anchors but are **deliberately not part of the `scheduler_anchors` 6-anchor set**: they don't drive any `input_select.zen_home_mode` transition, and `scheduler_anchors`' ascending-order guard is scoped to the 6 home-mode anchors only. Override them directly via Settings → Helpers or Developer Tools → States.

Breakfast and dinner don't have their own helpers — they're derived at read time from the existing schedule anchors: breakfast = `zen_morning_start`, dinner = `zen_evening_start` + 1 hour. `zen_dojotools_kitchen` `prep_brief` and `zen_dojotools_taskmaster` `almanac` both compute these the same way from the same anchors, so there's no separate drift-prone copy of meal defaults.

> **Timer seeding (4.5.6):** The `input_datetime` helpers ship with **no `initial:` value** — HA's `initial:` resets user-configured times on every restart, which was incorrect behavior. On first boot, Flynn seeds each timer individually to the defaults shown above (only if the helper is still bare — `''`, `unknown`, or `unavailable`). After that, your changes are permanent across restarts.

---

## Quiet and Work Hours

Two binary sensors derive from configurable time ranges. Both fail-open (resolve `off`) if either anchor is unavailable.

### `binary_sensor.zen_quiet_hours`

| Helper | Default |
|---|---|
| `input_datetime.zen_quiet_hours_start` | 23:00 |
| `input_datetime.zen_quiet_hours_end` | 06:00 |

**Midnight-wrap safe.** When `end < start`, the sensor is `on` if `now >= start OR now < end`. The 23:00–06:00 default spans midnight correctly.

### `binary_sensor.zen_work_hours`

| Helper | Default |
|---|---|
| `input_datetime.zen_work_hours_start` | 09:00 |
| `input_datetime.zen_work_hours_end` | 17:00 |

No midnight wrap. If you need a wrap here, use the quiet-hours pattern instead.

---

## Automations

| ID | Trigger | What it does |
|---|---|---|
| `zen_hm_schedule` | Each schedule anchor time fires | Calls `zen_hm_evaluate_window`. Guarded: skips if Away or Paused. |
| `zen_hm_away` | `zone.home` drops below 1 | Sets mode to `Away`. Guarded: skips if already Away or Paused. |
| `zen_hm_presence_return` | `zone.home` rises above 0 while in `Away` | Calls `zen_home_mode_release` → evaluates window → correct scheduled state. |
| `zen_hm_init` | HA start/restart | Calls `zen_hm_evaluate_window`. Guarded: skips if Away or Paused. |

---

## Scripts

### `zen_hm_evaluate_window`

Single source of truth for time-window → mode mapping. Reads all six anchor helpers, compares current time, and selects the correct mode. Called by the schedule automation, the init automation, and the release script.

**Do not duplicate this logic elsewhere.**

### `zen_home_mode_release`

Exits `Away` or `Paused` → calls `zen_hm_evaluate_window`. Use this whenever you want to return to the normal schedule from a hold state.

**Guarded:** exits immediately if mode is not `Away` or `Paused`.

---

## Scheduler Integration

`sensor.zen_home_mode` is a Scheduler trigger source. Components can subscribe to mode-specific transitions:

| Trigger ID | Fires when |
|---|---|
| `home_mode_updates` | Any `sensor.zen_home_mode` state change |
| `start_home_wake` | Mode transitions to `Home-Wake` |
| `start_home_evening` | Mode transitions to `Home-Evening` |
| `home_occupancy_change` | `zone.home` count changes |

Subscribe via `trigger_subscriptions` in a component's Dojo drawer. See [Scheduler readme](zen_dojotools_scheduler_readme.md).

---

## Vacation Mode

Apply the `zen_vacation_mode` label to your calendar entity. The system resolves vacation context from that label at runtime — no hardcoded entity IDs required.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Mode not advancing on schedule | Confirm mode is not `Away` or `Paused` — both block scheduled transitions |
| Away not triggering | Check `zone.home` entity state in Developer Tools |
| Presence return not releasing Away | Check `zen_hm_presence_return` automation — it only fires when mode is exactly `Away` |
| Quiet hours wrong | Verify `zen_quiet_hours_start` and `zen_quiet_hours_end` — if `end < start` the window wraps midnight (correct behavior) |
| Work hours always off | Check both `zen_work_hours_start` and `zen_work_hours_end` are set and not `unknown` |
| Timers reset on restart | This was a pre-4.5.6 issue caused by `initial:` in the helper definition. 4.5.6 removed `initial:` — Flynn now seeds defaults on first boot only. Upgrade to 4.5.6 and run `zen_maint_4_5_6_identity_family_repair` is not needed for this; timers will stabilize on the next restart after upgrading. |
