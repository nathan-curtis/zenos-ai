# Understanding the Dojo, Kung Fu, and the Summarizer

### A plain-language guide to ZenOS-AI KF4

---

## The short version

Your home has systems. The water softener. The security sensors. The hot tub. Energy usage. Each one generates a stream of state changes that — on their own — mean nothing to a language model.

**KF4 is how ZenOS-AI turns that noise into signal.**

It does this with three things working together:

- **The Dojo** — where your home's intelligence is defined
- **Kung Fu Components** — the definitions themselves
- **The Summarizer pipeline** — the engine that reads those definitions and produces something Friday can actually reason from

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
master_switch:    input_boolean.water_manager_master_switch
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
| `master_switch` | The toggle that enables/disables this component |
| `trigger_subscriptions` | Which Scheduler events cause this component to summarize |
| `delay_seconds` | How long to wait after trigger before dispatching (lower = higher priority) |
| `kata_key` | Where to write the summary output |
| `component_summary` | Instructions the AI reads when summarizing this component |

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
1. Reads the KFC drawer (the spec you wrote)
2. Resolves the label → live entity list via HyperIndex
3. Reads the last summary for this component (for continuity)
4. Builds a prompt: spec + live data + prior summary + trigger context
5. Sends to your language model
6. Model writes a compact, structured Kata entry
7. Kata is stored in the Kata Cabinet under kata_key
```

A **Kata** is a compact, structured summary of one component at one moment in time. It captures:

- What's the current state of this subsystem?
- Anything needing attention?
- Any recent notable events?
- Confidence level, urgency score

Katas are not raw sensor data. They're **curated signal** — what a knowledgeable observer would say about this system if you asked them right now.

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

Use `zen_dojotools_kungfu_writer` to press a new drawer into the Dojo Cabinet. Ask it for `action=help` first — it'll return the live schema with all required fields.

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

The Scheduler ships with a standard set of triggers (time patterns, home mode changes, door/lock/window state changes, etc.). Your component subscribes to any combination of those by listing their IDs in `trigger_subscriptions`.

But what if you want your component to summarize when *your* specific hardware does something — a Withings sleep sensor, a specific flow sensor, a custom device?

Create a Home Assistant automation that fires when your hardware changes, and has it fire a `zen_event`:

```yaml
action:
  - event: zen_event
    event_data:
      event:
        kind: summary_force
        component: your_kata_key
```

The core Scheduler picks that up as a targeted `force_summary` for your component. Your hardware drives the dispatch. No Scheduler YAML needed.

---

## A note on label naming

> **The label name in the drawer must exactly match the HA label.**

This is the most common setup mistake. If the drawer says `label: Water Manager` and the HA label is `water_manager`, the HyperIndex returns nothing. The monk gets no entities. The kata is empty.

Get the name right, then copy-paste it. Don't type both independently.

---

## The living spec

The `kung_fu` drawer in the Dojo Cabinet is the canonical KF4 component contract — maintained by the system, readable by Friday, and always authoritative. If you're ever unsure about the schema, read it:

```
zen_dojotools_index → key: kung_fu → Dojo Cabinet
```

---

*ZenOS-AI KF4 RC2 — 2026-03-11*
*Source: Nyx (live system observation), Cait (dev)*
