# Taskmaster — KFC Guide

> **KF4 1.5.0** | ZenOS-AI 2026.4.1

Taskmaster is ZenOS-AI's task and calendar management component. It monitors your todo lists and calendar, surfaces what needs attention, and gives you a brief when it matters — morning, evening, or when something's actually urgent.

It ships disabled. Enable it when you're ready to wire in your task lists.

---

## Enabling taskmaster

`taskmaster` ships with `meta.enabled: false`. To enable:

- Ask your AI: *"Enable taskmaster"*
- Or set `meta.enabled: true` in the `taskmaster` Dojo drawer via `zen_dojotools_scribe`

Once enabled, taskmaster runs on its scheduled triggers. To wire instant dispatch on sleep/wake transitions, see [Wiring Your Own Triggers](#wiring-your-own-triggers) below.

---

## The Conductor Pattern

Taskmaster uses a **conductor** model to organize tasks across time.

The conductor is a structured todo list (`todo.friday_s_personal_task_list` by default) that acts as a schedule. Each item in the conductor references a time block and a domain list. When a trigger fires, taskmaster:

1. Reads the conductor to find the active block for the current period
2. Pulls the referenced domain lists
3. Cross-references with calendar events
4. Surfaces what's due, overdue, or coming up

You don't need to manage the conductor manually — it's maintained by the AI and updated as tasks are completed or rescheduled.

---

## Data Sources

Taskmaster reads from entities tagged with the **`taskmaster`** label. Entities are grouped by secondary labels:

| Label | What it contains |
|-------|-----------------|
| `daily` | Recurring daily tasks (chores, health) |
| `weekly` | Weekly cadence tasks |
| `tasks` | Async backlog (tickets, automation todos) |
| `calendar` | Calendar entities for event cross-reference |

Tag entities with both `taskmaster` and their domain label. The Ninja Summarizer resolves everything automatically via label expansion.

---

## Overdue Semantics and PRN Items

- **Overdue**: a task with a `due:` date of yesterday or earlier, still `notStarted`.
- **PRN items** ("as needed"): tasks with no due date that appear in the backlog. Taskmaster surfaces count + top items but doesn't foreground them over conductor-driven tasks.
- **Urgency escalation**: a calendar event starting within 30 minutes with the relevant person not moving will push urgency to 5 (dispatch threshold).

---

## When It Runs

Taskmaster subscribes to:

- `home_occupancy_change` — arrival or departure shifts task context
- `quarter_hour` — regular cadence check
- `force_summary` — targeted dispatch on demand
- `default_user_bed_changes` / `secondary_user_bed_changes` — wake and sleep transitions (if wired)

The morning and evening cycles are the most useful:

**Morning (after wake trigger):**
1. Read the conductor — what's scheduled today?
2. Pull tasks from referenced domain lists
3. Any calendar events today?
4. Any tasks overdue from yesterday?
→ Brief morning summary with time-ordered schedule.

**Evening (bed sensors → sleep):**
1. What conductor blocks ran today?
2. What wasn't completed?
3. What's on tomorrow's conductor?
→ Brief evening wrap.

---

## Urgency Scoring

| Condition | Urgency |
|-----------|---------|
| Calendar event starting within 30 min, person not moving | 5 (dispatch) |
| Active conductor block with past-due required tasks | 4 (informational) |
| Task with `due:` yesterday or earlier, still notStarted | 3 (informational) |
| Morning brief pending | 2 |
| Nothing pressing | 1 |

Urgency 3–4 updates the kata but does **not** trigger a notification. Urgency ≥ 5 dispatches via `zen_dojotools_notification_router`.

Tasks are a reminder system. An overdue task is a reminder, not a crisis. The AI won't escalate language to match urgency numbers — see the tone directives in the system prompt.

### MED HARD CAPS

> **HARD CAP — no override path.** Medication-related tasks are **CLINICAL** items. Clinical urgency is capped at **3** (informational). Not 4. Not 5. Not higher. The medication escalation path in prior versions has been removed.
>
> Medications are part of a managed care routine, not emergencies taskmaster can assess. A missed dose is not the same as a calendar event starting in 30 minutes. Urgency 3 is the ceiling; the kata communicates the task without creating false emergency framing.
>
> This cap applies to any task tagged with `medication`, `med`, `clinical`, or `pharmacy`, or where the conductor maps to a `clinical` domain list. If you are integrating a med management system, do not route it to urgency > 3 via taskmaster — use an independent alert component with appropriate clinical context if escalation is genuinely needed.

---

## Wiring Your Own Triggers

Taskmaster's most useful wake/sleep behavior requires hardware — a bed sensor or occupancy sensor. Create a dedicated trigger file in your installer's custom packages directory:

```yaml
# packages/your_family/kfc_trigger_taskmaster.yaml
# ⚠️ PERSONAL FILE — DO NOT COMMIT TO REPO

automation:
  - id: 'YOUR_UNIQUE_ID'
    alias: KFC Trigger — Taskmaster
    description: >-
      Fires taskmaster on wake and sleep transitions.
    triggers:
      - trigger: state
        entity_id:
          - binary_sensor.your_bed_sensor       # REPLACE — occupant 1
        to: 'off'                               # got out of bed
        not_from: [unknown, unavailable]
        for:
          seconds: 30
        id: default_user_bed_changes

      - trigger: state
        entity_id:
          - binary_sensor.your_bed_sensor       # REPLACE — occupant 1
        to: 'on'                               # in bed
        not_from: [unknown, unavailable]
        for:
          seconds: 60
        id: default_user_bed_changes

    actions:
      - event: zen_event
        event_data:
          event:
            kind: summary_force
            component: taskmaster

    mode: queued
    max: 3
```

If you don't have a bed sensor, taskmaster still runs on `home_occupancy_change` and `quarter_hour`. You lose the morning/evening cycle timing but keep the core behavior.

---

## Adding Task Lists

Any todo entity tagged with `taskmaster` + a domain label (`daily`, `weekly`, `tasks`) is automatically included. To add a new list:

1. In HA's entity registry, tag the todo entity with `taskmaster` and the appropriate domain label
2. On the next summarizer run, it's included

No drawer edit. No code. The label graph handles it.

---

*ZenOS-AI KF4 1.5.0 — 2026-04-14*
*Source: Nyx (live system observation), Cayt (dev)*
