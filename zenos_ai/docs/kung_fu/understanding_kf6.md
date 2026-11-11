# Understanding the Dojo, Kung Fu, and the Action Pipeline

### A plain-language guide to ZenOS-AI KF6

---

## The short version

Your home has systems. The water softener. The security sensors. The hot tub. Energy usage. Each one generates a stream of state changes that — on their own — mean nothing to a language model.

**KF6 is how ZenOS-AI turns that noise into signal.**

It does this with three things working together:

- **The Dojo** — where your home's intelligence is defined
- **Kung Fu Components** — the definitions themselves
- **The action pipeline** — the engine that reads those definitions, synthesizes them, and acts on what it finds

You don't need to touch the Scheduler. You don't need to write any code. You define a component, tag some entities, and the system wires itself.

---

## What is the Dojo?

The **Dojo Cabinet** is a persistent key/value store in Home Assistant — one of the ZenOS-AI cabinets provisioned during install.

Think of it as the **rulebook for your home**. Every major subsystem that Friday should know about has an entry in the Dojo. That entry — called a **drawer** — describes what the subsystem is, which entities belong to it, and when to summarize it.

The Dojo is not a log. It's not a history. It's the living specification for how your home works.

---

## What is a Kung Fu Component?

A **Kung Fu Component (KFC)** is a drawer in the Dojo Cabinet that describes one home subsystem.

Here's what a real one looks like (Water Manager):

```yaml
friendly_name:    Water Manager
label:            Water Manager
meta:
  enabled:        true
trigger_subscriptions:
  - daily_midnight
  - water_flow_stop
  - force_summary
delay_seconds:    120
kata_key:         water_manager
component_summary: >
  Monitors municipal water consumption and the water softener. Key entities:
  flow rate sensor, shutoff valve (alert_when_on = leak detected), softener
  status (alert_when_off = service needed). Flag any flow above 2 GPM with
  no active fixture as a potential leak.
```

That's it. That drawer is the entire spec for how ZenOS-AI understands your water system.

**What each field does:**

| Field | What it means |
|-------|--------------|
| `friendly_name` | Human-readable name |
| `label` | The HA label that defines which entities belong to this component |
| `meta.enabled` | Whether this component is active (`true` / `false`). A disabled component stays out of the agent's prompt; its last kata is kept. Replaces the deprecated `master_switch` field. |
| `trigger_subscriptions` | Which Scheduler events cause this component to summarize |
| `delay_seconds` | How long to wait after trigger before dispatching (lower = higher priority) |
| `pipeline_tier` | Dispatch priority: `keeper` (default), `ambient` (low-priority), `super` (reserved), `system` (infrastructure). See below. |
| `staleness_minutes` | How old the kata can get before the drain router forces a re-run. Default: 1440 min (24 h). |
| `kata_key` | Where to write the summary output |
| `component_summary` | Instructions the AI reads when summarizing this component |
| `meta.requires` | Optional cert and minimum level the household's agent needs, granted for the background (stage `s`), for this component to run. Unmet, the run is refused and the component stays out of the prompt. Leave it empty unless you need it. |

The `component_summary` is the most important field. It's your chance to tell the AI what *normal* looks like, what *concerning* looks like, and what specific entities need special interpretation.

---

## The label is the scope

The `label` field is a **Home Assistant label** — the same labels you can apply to entities in the HA entity registry.

When the Summarizer runs for a component, it doesn't get a hardcoded list of entities. It gets the label name. ZenOS-AI then traverses the HA label graph to find every entity tagged with that label — live, at runtime.

**This means:**

> Add a new leak sensor to your home? Tag it `Water Manager`. On the next Summarizer run, it's automatically included. No code change. No drawer edit.

---

## State semantics: alert_when_off and alert_when_on

Not all sensors work the same way. Some are "on is bad." Some are "off is bad."

The AI has no way to know this unless you tell it. The `component_summary` field is where you do that — using two conventions:

- `alert_when_off` — this entity is nominal when ON, concerning when OFF.
  *(Example: a valve that should always be open, a switch that should always be running)*

- `alert_when_on` — this entity is nominal when OFF, concerning when ON.
  *(Example: a leak sensor, a motion sensor in a room that should be empty)*

You can reference these in your component_summary prose, or as HA labels on individual entities.

Without this, the AI will guess — and sometimes get it wrong. A heat pump shutoff valve in the `on` (open) state is perfectly healthy. Without the convention, it might look alarming.

---

## A Summarizer-what?

The **Ninja Summarizer** is the ZenOS-AI script that runs for each Kung Fu Component when a trigger fires.

Here's what it does, step by step:

```
1. Runs as the agent responsible for your household (it must be onboarded)
2. Reads the KFC drawer (the spec you wrote)
3. Resolves the label → live entity list via HyperIndex (or calls the seed, if the tool declares that mode)
4. Reads the last summary for this component (for continuity)
5. Builds a prompt: spec + live data + prior summary + trigger context
6. Sends to your language model
7. Model writes a compact, structured Kata entry
8. Kata is stored in the Kata Cabinet under kata_key
9. If the kata calls for action, acts only with a cert granted for the background
```

**Since 2026.11.0 the pipeline runs as someone.** If the household's agent isn't onboarded, summaries pause and you get a notification saying why. A seed must be a mode its tool declares, or the run stops before the model is called. And an action at the end of a run (usually a Postman notification) needs a cert the agent holds for the background: one phone approval of `zenos_agent_autonomous` covers it. Without it the action doesn't happen, and the kata says what to grant. See [Building a KFC: the security contract](building_a_kfc.md#kf6-the-security-contract-2026110).

A **Kata** is a compact, structured summary of one component at one moment in time. It captures:

- What's the current state of this subsystem?
- Anything needing attention?
- Any recent notable events?
- Confidence level, urgency score

Katas are not raw sensor data. They're **curated signal** — what a knowledgeable observer would say about this system if you asked them right now.

---

## Pressure-aware dispatch

The Scheduler doesn't run all components on every trigger. Under load, it defers lower-priority work and recovers it later. This is controlled by two thresholds in the `zen_scheduler_config` drawer of your household cabinet:

| Setting | Default | Effect |
|---------|---------|--------|
| `shed_ambient_at` | 4 | Queue depth at which `ambient`-tier components are deferred |
| `shed_keeper_at` | 8 | Queue depth at which `keeper`-tier components are deferred |
| `drain_below` | 3 | Depth the queue must fall below before shed work trickles back |
| `warmup_minutes` | 5 | Boot delay before the first Scheduler dispatch |

**`keeper` (the default)** — dispatched on all standard triggers; deferred when queue ≥ 8.

**`ambient`** — dispatched only on slower triggers (hourly, occupancy change, etc.). Shed on fast triggers (`quarter_hour`, `every_10_minutes`) and when queue ≥ 4. These are pre-digested by Trapper Keeper rather than read directly by SuperSummary — Friday gets their signal through the ambient index.

**Shed work doesn't disappear.** The drain router runs on a low-water timer and a starvation guard. When the queue quiets down, the most-stale shed-eligible component gets a recovery dispatch. If a component's kata exceeds its `staleness_minutes` ceiling, it fires regardless of queue depth.

A targeted `force_summary` for a specific component bypasses shedding entirely.

---

## The SuperSummary

Once all the per-component Katas are written, the **SuperSummary** runs on its own schedule and does one more pass:

```
All component Katas → synthesized zen_summary → Kata Cabinet → Friday
```

The `zen_summary` is what Friday actually reasons from. Not raw entities. Not logs. A structured, component-aware, freshly-synthesized picture of your home.

---

## How to add a new component

No Scheduler changes. No code. Five steps.

**Step 1 — Write the drawer**

Use `zen_dojotools_scribe` to press a new drawer into the Dojo Cabinet. Run with `mode=schema` first — it returns the live `kfc_template` with all required fields. (If a tool owns the domain, the tool can declare the component itself instead: see [Building a KFC](building_a_kfc.md#kf6-two-ways-in-one-contract).)

**Step 2 — Create the label**

Use `zen_dojotools_labels` to create an HA label matching the `label` field you used in the drawer. The name must match exactly — it's case-sensitive.

**Step 3 — Tag your entities**

In HA's entity registry, apply the label to every entity that belongs to this component. The HyperIndex will find them automatically on the next run.

**Step 4 — Dry run**

Before going live, test it. Fire a `force_summary` zen_event targeting your component. Review the output:

- Did it find the right entities?
- Did it interpret states correctly?
- Does the kata output make sense?

If the model is misreading an entity state, add `alert_when_off` or `alert_when_on` notes to your `component_summary` and re-run.

**Step 5 — Go live**

Satisfied? Do nothing. The next Scheduler tick will find your component in the Dojo, see its `trigger_subscriptions`, and start dispatching. It's live.

---

## Custom hardware triggers

The Scheduler ships with a standard set of triggers (time patterns, home mode changes, door/lock/window state changes, etc.). Your component subscribes to any combination of those by listing their IDs in `trigger_subscriptions`. `zen_dojotools_scribe mode=list_triggers` lists every one that's registered.

**Try a label first.** Since 2026.11.0, five triggers fire on a label instead of an entity: `zen_leak`, `zen_smoke`, `zen_door`, `zen_power` and `zen_appliance`. Label a leak sensor `zen_leak`, subscribe your component to `zen_leak`, and it runs when the sensor gets wet. No YAML, no reload. See [Triggers](building_a_kfc.md#triggers).

**Shipping a component that owns its triggers?** Put them in the component's own package file as a trigger pack that registers itself, and make it label-driven so it works in any house. See [Triggers](building_a_kfc.md#triggers).

But what if you want your component to summarize when *your* specific hardware does something: a sleep sensor, a specific flow sensor, a custom device?

Wire it with a small trigger file per KFC. It names your entity IDs, so keep it in your installation's own packages directory (outside `packages/zenos_ai/`), never in the shared repo. A trigger that names a label can ship; a trigger that names an entity ID is personal.

```yaml
# packages/your_family/kfc_trigger_<component>.yaml
# ⚠️ PERSONAL FILE — DO NOT COMMIT TO REPO

automation:
  - id: 'YOUR_UNIQUE_ID'
    alias: KFC Trigger — [Component Name]
    description: >-
      Fires a targeted force_summary for [component] on meaningful state changes.
    triggers:
      - trigger: state
        entity_id:
          - binary_sensor.your_entity_here
        to: 'on'                      # alert_when_on: only dispatch on bad state
        not_from: [unknown, unavailable]
        for:
          seconds: 5                  # debounce — adjust per entity

    actions:
      - event: zen_event
        event_data:
          event:
            kind: summary_force
            component: your_kata_key  # matches kata_key in the Dojo drawer

    mode: queued
    max: 3
```

The core Scheduler picks up `zen_event` with `kind: summary_force` as a targeted `force_summary` for your component. Your hardware drives the dispatch. No Scheduler YAML needed.

One file per KFC that needs instant dispatch. KFCs that are fine with scheduled sweeps need no trigger file at all.

---

## A note on label naming

> **The label name in the drawer must exactly match the HA label.**

This is the most common setup mistake. If the drawer says `label: Water Manager` and the HA label is `water_manager`, the HyperIndex returns nothing. The monk gets no entities. The kata is empty.

Get the name right, then copy-paste it. Don't type both independently.

---

## The living spec

The `kfc_template` drawer in the Dojo Cabinet is the canonical KF6 component contract — maintained by the system, readable by Friday, and always authoritative. If you're ever unsure about the schema, read it:

```
zen_dojotools_scribe mode=schema
```

---

*ZenOS-AI KF6 — 2026.11.0 'This Is Spinal Tap'*
*Source: Nyx (live system observation), Cayt (dev)*
