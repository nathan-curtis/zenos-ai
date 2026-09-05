# Taskmaster — KFC Guide

> **dojotools 0.3.0** | ZenOS-AI 2026.8.1 (patch on 'Chef')

Taskmaster is ZenOS-AI's cross-backend task orchestration layer: household chores, tickets, calendar, and medications, urgency-scored and surfaced through one briefing call. As of 2026.8.0 it's a full dojotools script (`zen_dojotools_taskmaster`) with real modes, self-registering via KF5 — not a label-driven Ninja Summarizer component like earlier KF4 versions. If you're looking for the old conductor-pattern / label-graph docs, that architecture no longer exists; this page describes the current tool.

---

## Progressive Enhancement

Taskmaster does **not** merge Radar, Inventory, CRM, and To Do into one data model — each backend keeps its native shape. It picks the best available backend at creation time and fans out reads across whichever backends are configured:

| Tier | Backend | When it's used |
|------|---------|-----------------|
| Base | O365 To Do (`zen_dojotools_todo`) | Always available — every household gets this with zero config. |
| + Radar | Zammad service desk (`zen_dojotools_servicedesk`) | When configured, task creation upgrades to a real tracked ticket with lifecycle/triage. |
| + Inventory | Grocy (`zen_dojotools_inventory`) | When configured, chore-shaped recurring tasks upgrade to Grocy chores/tasks tied to product/location/stock. A chore is a persistent recurring *definition*, not a fresh record per occurrence — Taskmaster looks the chore up by name first: if it already exists, it executes it (logs the occurrence, advances `next_due`); it only creates a new chore when genuinely new. |
| + CRM | Rolodex/Twenty (`zen_dojotools_rolodex`) | Explicit contact/company context always wins the routing decision, regardless of shape. |

Routing priority at creation time: explicit contact/company → chore-shaped → Radar-shaped → Grocy task → base O365 To Do.

Taskmaster self-registers with the Lens Bus (same pattern as `zen_dojotools_inventory`), so `task_evidence` surfaces anywhere Lens Bus queries already run.

---

## Modes

| Mode | Description |
|------|-------------|
| `briefing` | The scheduled seed — urgency-scored household briefing. Fires on `quarter_hour` + occupancy-change triggers via KF5 self-registration, 60-minute emission cooldown, 180s delay so it doesn't crowd every other component firing on the raw `quarter_hour` tick. Conductor items (Friday's own schedule-driven task list) surface first, then everything else from `task_list`, scored by shape (medication classification/PRN/date rules for todo+crm, chore due-status for inventory, priority passthrough for radar). Also returns a `context` block of real (not LLM-guessed) situational signals — see [Context Block](#context-block) below. |
| `almanac` | On-demand "what does today look like" — meal plan (`zen_dojotools_kitchen mode=run case=mealplan_today`), appointments (`zen_dojotools_rolodex mode=appointments_list`), plus a meal/appointment collision flag. Informational only, never blocking. |
| `stale_review` | Filters `briefing`'s own scoring to urgency ≥ 2, sorted highest-first. Suppresses Radar tickets triaged `radar_rank_backlog` regardless of raw priority — those are intentionally deprioritized, not stale in the "forgotten" sense. |
| `facilities_brief` | Building-super/housekeeping-lead view. Reads live kata cabinet entries for physical-plant components directly (`energy_manager`, `water_manager`, `hot_tub_manager`, `garage_freezer_thermal_model`, `dishwasher_manager`, `laundry_manager`) — no re-query — surfaces only urgency ≥ 2. Plus open Radar tickets tagged to a physical area and the house-wide chore backlog. |
| `task_create` | Create a task. Routes to the best available backend per the priority chain above. |
| `task_list` | List tasks across whichever backends are configured. The data `briefing` and `stale_review` build on. |
| `task_complete` | Mark a task complete on whichever backend it lives on. |
| `tier_status` | Reports which backends (Radar/Inventory/CRM) are actually configured right now, via each backend's own `tool_manifest`. |
| `setup` | Self-provisioning audit + fix — checks whether `zen_kfc_provider` label is applied (bootstrap_kfc discovery) and whether Taskmaster is on the summarizer seed whitelist (live briefing seed vs. index-fallback path). Both checked live, both fixed automatically unless `dry_run=true`. Same shape as `zen_dojotools_autovac mode=setup`. |
| `help` | Standard mode/field listing. |
| `register` / `unregister` | Lens Bus provider registration lifecycle. |
| `tool_manifest` | Standard manifest response. |
| `kfc_manifest` | KF5 self-registration payload — the `component_instructions` prompt text that ships with the KFC (see [KF5 Pattern](../scripts/zen_kf5_pattern_readme.md)). |

---

## Context Block

`briefing`'s response includes a `context` block of real, ground-truth situational signals — never LLM-guessed:

| Field | Source |
|-------|--------|
| `home_mode` / `is_away` | `input_select.zen_home_mode` |
| `is_weekday` | Calendar |
| `quiet_hours_active` / `work_hours_active` | `binary_sensor.zen_quiet_hours` / `binary_sensor.zen_work_hours` |
| `guest_present` / `guest_note` | Manual flags + Rolodex `stays_list` for today |
| `upcoming_appointments` | Rolodex `appointments_list` |
| `prep_schedule` | `zen_dojotools_kitchen mode=run case=prep_brief` for today — scheduled `[PREP:N]`-tagged dishes sorted by `start_by`, plus `unscheduled_count`. Wired in 2026.8.0 so Kitchen's prep timing is proactive on Taskmaster's existing trigger, not pull-only. |
| `weather.outdoor_ok` | Label `zen_weather` → `weather.*` entity; `false` when condition is rain/snow/storm. |

Consumer guidance (baked into the KFC's own `component_instructions`): use `context.prep_schedule.scheduled` entries whose `start_by` is within the next hour (or already past) as a real "start cooking now" signal, not background noise. `unscheduled_count` is informational only. `weather.outdoor_ok == false` means note outdoor chores as weather-blocked, not urgent.

---

## Deterministic Scoring

Scoring is rule-based, not LLM-guessed. Can be disabled via household cabinet key `taskmaster_scoring_enabled` (default `true`) — when off, items return unscored.

### MED HARD CAPS

> **HARD CAP — no override path.** Medications are not life-safety events. A missed dose is a reminder, not an emergency. **Medications never rank above urgency 3, for any reason.** This is enforced structurally in the tool, restated in the KFC's own prompt as defense-in-depth, and restated here as the source of truth. Medication classification (SUPPLEMENT/STANDARD/CLINICAL/BIOLOGIC) can be overridden per-title via household cabinet key `medication_classifications`.

### Chore Scoring

Active anomaly with no due date (e.g. equipment alerts) → urgency 3. Overdue 1–3 days → 2. Overdue 4–7 days → 3. Due today → 1–2 (inform, don't alarm). Due in future → 0–1, don't surface unless asked. Stale (>30 days) → ambient only, max urgency 1, surface count only.

### PRN / Optional Tasks

Tasks flagged "if needed"/"as needed"/PRN/optional never count toward `action_required` or urgency escalation — urgency contribution is 0, regardless of how overdue they are.

---

## Adding Task Lists

Any todo entity tagged with `taskmaster` + a domain label (`daily`, `weekly`, `tasks`) is picked up by the underlying backend readers — no drawer edit, no code change required. Tag the entity in HA's entity registry and it's included on the next `task_list`/`briefing` call.

---

## Related

- [Kitchen / Mealie](../plugins/mealie.md) — `prep_brief` feeds Taskmaster's `context.prep_schedule`.
- [KF5 Self-Registration Pattern](../scripts/zen_kf5_pattern_readme.md) — how Taskmaster registers itself without a manual Scribe authoring step.

---

*ZenOS-AI dojotools 0.3.0 — 2026.8.1 (patch on 'Chef')*
