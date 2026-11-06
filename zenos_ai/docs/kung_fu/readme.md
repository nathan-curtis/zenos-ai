# Kung Fu Components — KF6

### Structured Subsystem Definitions for the Dojo Cabinet

Kung Fu Components (KFCs) define *how each major subsystem in your home works*.
They live as drawers inside the **Dojo Cabinet** and drive the Scheduler,
the Ninja Summarizer, and Friday's reasoning.

**KF6 invariants:**

- **Drawer IS the spec** — the Dojo drawer is the single source of truth for the component
- **Label IS the scope** — the HA label defines which entities belong to the component. Modifier labels (e.g. `alert_when_on`, `alert_when_off`, `alert_when_under_N`) extend this — the label name itself encodes per-entity semantics, readable by the Ninja without any component_summary explanation.
- **HyperIndex IS the data layer** — no hardcoded entity lists; the index traverses the label graph
- **The pipeline runs as someone** (2026.11.0) — the household's onboarded agent; seeds are modes a tool declares; acts need a cert granted for the background. See [the security contract](building_a_kfc.md#kf6-the-security-contract-2026110).

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

Use `zen_dojotools_scribe` (MCP-exposed) to write drawers.
Run with `mode=schema` to get the live `kfc_template` from the Dojo cabinet.

### Drawer Schema (kfc_template)

| Field | Required | Description |
|-------|----------|-------------|
| `version` | yes | Semantic version string, e.g. `1.0.0` |
| `friendly_name` | yes | Human-readable component title |
| `meta.enabled` | yes | Boolean — `true` to enable, `false` to disable this component. A disabled component stays out of the agent's prompt. |
| `meta.requires.cert` | no | Cert the household's agent must hold, granted for stage `s` (Ninja seed), for this component to run. Unmet: the Ninja refuses with `kfc_cert_required` and the component stays out of the prompt. Its tool must allow `s`. See [the minimum level](building_a_kfc.md#kf6-the-security-contract-2026110). |
| `meta.requires.level` | no | Minimum level (1–4) of that cert. `0` = no requirement. |
| `label` | yes | HA label that scopes the component's entity set |
| `component_summary` | yes | One-sentence description of what this component manages |
| `trigger_subscriptions` | yes | List of Scheduler trigger IDs that fire this component |
| `delay_seconds` | yes | Seconds to delay after trigger (0 = immediate; used for ordering) |
| `kata_key` | yes | Dojo drawer key where the Ninja writes this component's Kata |
| `command` | no | Library command that activates this component (e.g. `~WATER~`). Omit for HyperIndex-only components. |
| `schedules` | no | List of sub-schedule entries. Each entry has `kata_key` (string) and `index_call` (mapping). When present, the Scheduler expands the component into one Ninja run per schedule entry instead of a single run. A targeted `force_summary` with `schedule: <kata_key>` fires only the matching entry. |
| `tool` | no | Specific DojoTools tool this component uses, if any |
| `pipeline_tier` | no | Dispatch priority and SuperSummary routing. See tier table below. Default: `keeper`. |
| `staleness_minutes` | no | Age threshold (minutes) at which the drain router will force a re-run regardless of queue depth. Default: `1440` (24 h). Set lower for components that must stay fresh. |
| `emission_cooldown_minutes` | no | Minimum minutes between emissions for this component. Prevents notification spam on rapid-fire triggers. |
| `drift_threshold` | no | Minimum urgency delta required to consider a kata change "significant" for emission gating. |
| `suggested_act_event` | no | Slug of the action the AI should suggest in kata output (e.g. `zen_dojotools_notification_router`). `null` = no action. The act runs only if the household's agent holds the target tool's emit cert with stage `e`; otherwise the kata records `act_blocked`. |
| `chain_to` | no | Components this one may start through the Ninja's `run` act. A target outside the list is refused; the chain stops 3 deep and never loops. Needs `zenos.pipeline.chain` with stage `e`. |
| `master_switch` | removed | Ignored and stored empty since 2026.11.0. On and off is `meta.enabled` (Scribe `enable` / `disable`). |

### Pipeline Tier Values

| Tier | Shedding behavior | SuperSummary routing |
|------|-------------------|----------------------|
| `keeper` | Shed when queue depth ≥ `shed_keeper_at` (default 8). Standard components use this. | Kata read directly by SuperSummary |
| `system` | Never shed, dispatched with zero delay, same as `super`. Core platform health is the signal most needed when the queue is already under pressure. Conventional for core infrastructure components (e.g. trapper_keeper). | Excluded from SuperSummary's component data. Trapper Keeper's output reaches SuperSummary as `ambient_context` |
| `ambient` | Shed on fast triggers (`quarter_hour`, `every_10_minutes`) AND when depth ≥ `shed_ambient_at` (default 4). More aggressively deferred under load. | Kata pre-digested by Trapper Keeper into `ambient_context`; SuperSummary receives the index, not the raw kata |
| `super` | Never shed. Reserved for `zen_dojotools_supersummary` internal dispatch. Do not use on regular components. | N/A |
| `direct` | Deprecated alias for `keeper`. Honored but not written by new tooling. | Kata read directly by SuperSummary |

Omit `pipeline_tier` for standard components — `keeper` is the default.

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
| `zen_leak` | A sensor labeled `zen_leak` detects water |
| `zen_smoke` | A sensor labeled `zen_smoke` detects smoke, CO or gas |
| `zen_door` | A door or window labeled `zen_door` opens or closes |
| `zen_power` | A power sensor labeled `zen_power` stays below 1 W for 30 s |
| `zen_appliance` | A power sensor labeled `zen_appliance` starts (above 10 W) or finishes (below 3 W for 2 minutes) |

The live list is the registry: `zen_dojotools_scribe mode=list_triggers`. A subscription nobody registers never fires. The label triggers sit idle until an entity carries the label. For your own triggers, register a pack; see [Triggers](building_a_kfc.md#triggers).

---

## Example Drawer (Water Manager)

```yaml
version: "1.0.0"
friendly_name: Water Manager
meta:
  enabled: true
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

As of KF4 1.2.0, `energy_manager` is the canonical HyperIndex-only component.
Six others (`security_manager`, `taskmaster`, `room_manager`, `media_manager`,
`water_manager`, `trash_trakker`) are validated and ready for command strip.

---

---

## Guides

- [`understanding_kf6.md`](understanding_kf6.md) — schema reference, tier table, pressure dispatch
- [`building_a_kfc.md`](building_a_kfc.md) — the KF6 spec: the five decisions, the two ways in, and the security contract
- [`zen_kf6_pattern_readme.md`](../scripts/zen_kf6_pattern_readme.md) — the self-registering path in full
- [`alert_manager.md`](alert_manager.md) — reference implementation, safety-critical pattern
- [`taskmaster.md`](taskmaster.md) — lifecycle/task pattern, conductor model

---

*Last updated: 2026-11-04 — KF6 (2026.11.0)*
