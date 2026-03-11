# Kung Fu Components — KF4 RC2

### Structured Subsystem Definitions for the Dojo Cabinet

Kung Fu Components (KFCs) define *how each major subsystem in your home works*.
They live as drawers inside the **Dojo Cabinet** and drive the Scheduler,
the Ninja Summarizer, and Friday's reasoning.

**KF4 RC2 invariants:**

- **Drawer IS the spec** — the Dojo drawer is the single source of truth for the component
- **Label IS the scope** — the HA label defines which entities belong to the component
- **HyperIndex IS the data layer** — no hardcoded entity lists; the index traverses the label graph

---

## What a Kung Fu Component Is

A KFC is a structured drawer in the Dojo Cabinet describing:

- what the subsystem does and how Friday should reason about it
- which HA label scopes it (and therefore which entities are in scope)
- which Scheduler triggers should fire its Ninja Summarizer
- how long to delay dispatch after trigger (priority ordering)
- where to write its Kata output
- which master switch controls it
- which library command activates it (optional — HyperIndex-only components omit)

Each drawer is **both documentation and logic** — declarative, readable, and
prompt-friendly. The Scheduler reads it. The Ninja reads it. Friday reads it.

---

## How KFC Fits Into the System

```
Trigger fires
    ↓
Scheduler reads Dojo drawers → finds subscribers (trigger_subscriptions ∋ trigger.id)
    ↓
For each subscriber (sorted by delay_seconds):
    → Ninja Summarizer reads label graph (HyperIndex)
    → Monk interprets component spec + live entity data
    → Writes Kata to Kata Cabinet
    ↓
SuperSummary aggregates → Friday reasons
```

Adding a component requires **no Scheduler changes** — write a drawer, create a label,
tag entities. The loop auto-discovers it on the next tick.

---

## Creating Components

Use `zen_dojotools_kungfu_writer` (AI-accessible tool) to write drawers.
Run with `action=help` to get the live schema from the Dojo cabinet.

### Drawer Schema (kfc_template)

| Field | Required | Description |
|-------|----------|-------------|
| `version` | yes | Semantic version string, e.g. `1.0.0` |
| `friendly_name` | yes | Human-readable component title |
| `master_switch` | yes | `input_boolean.*` that enables/disables the component |
| `label` | yes | HA label that scopes the component's entity set |
| `component_summary` | yes | One-sentence description of what this component manages |
| `trigger_subscriptions` | yes | List of Scheduler trigger IDs that fire this component |
| `delay_seconds` | yes | Seconds to delay after trigger (0 = immediate; used for ordering) |
| `kata_key` | yes | Dojo drawer key where the Ninja writes this component's Kata |
| `command` | no | Library command that activates this component (e.g. `~WATER~`). Omit for HyperIndex-only components. |
| `tool` | no | Specific DojoTools tool this component uses, if any |

### Standard Trigger IDs

| ID | When |
|----|------|
| `ha_start` | HA restart |
| `daily_midnight` | 00:00 daily |
| `daily_noon` | 12:00 daily |
| `hourly_trigger` | Every hour |
| `quarter_hour` | Every 15 minutes |
| `every_10_minutes` | Every 10 minutes |
| `home_mode_updates` | `zen_home_mode` changes |
| `home_occupancy_change` | `zone.home` count changes |
| `alarm_panel` | Alarm panel state change |
| `force_summary` | Targeted force via zen_event |

For custom triggers: create a user automation that fires a `zen_event` with your
custom event kind, then add a 5-line `zen_event` trigger to the Scheduler stem.
The Dojo loop picks up any subscriber registered to that trigger ID automatically.

---

## Example Drawer (Water Manager)

```yaml
version: "1.0.0"
friendly_name: Water Manager
master_switch: input_boolean.water_manager_master_switch
label: Water Manager
command: "~WATER~"
component_summary: Manages household water flow, leak detection, and salt level.
trigger_subscriptions:
  - daily_midnight
  - daily_noon
  - start_home_wake
  - water_flow_stop
  - force_summary
delay_seconds: 60
kata_key: water_manager
```

---

## When to Add a Component

Add a KFC when:

- a subsystem has domain-specific reasoning rules
- a domain needs consistent interpretation by Friday
- a behavior needs to be taught (what "normal" looks like, what failure means)
- summarization needs richer context
- a safety boundary needs defining

---

## Command Strip (KF4 Migration Note)

Components that were created before KF4 may have a `command` field pointing to a
Library macro (e.g. `~SECURITY~`). Once the HyperIndex label graph is fully tagged
and the Ninja Summarizer has been validated on real data, the `command` field can be
stripped — the component operates entirely through label traversal.

As of KF4 RC2, `energy_manager` is the canonical HyperIndex-only component.
Six others (`security_manager`, `taskmaster`, `room_manager`, `media_manager`,
`water_manager`, `trash_trakker`) are validated and ready for command strip.

---

*Last updated: 2026-03-11 — KF4 RC2*
