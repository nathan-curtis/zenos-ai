# Building a ZenOS KFC Component

**Audience:** Developers building a new KF4 component for ZenOS-AI.
**Scope:** Why the KFC is shaped the way it is, so you can shape yours the same way.

---

## What Is a KFC?

A **Kung Fu Component (KFC)** is ZenOS-AI's unit of scoped cognition. Each KFC owns one domain — alerts, tasks, energy, water, hot tub — and is responsible for producing a single, structured **kata**: a compact, high-signal snapshot of that domain that the knowledge tree can consume without re-reading the world.

The KFC pattern exists to solve a specific problem: **LLM invocations in event-driven systems expand unboundedly if you let them.** One state change triggers a summary, that summary triggers another, and suddenly you have a runaway chain burning tokens and time with no circuit breaker.

KFCs are the circuit breaker.

Every design decision in a KFC exists to answer one question: *how do we get the most useful answer in the least time, without letting one component blow up the whole pipeline?*

---

## The Five Decisions

When you build a KFC, you make five decisions. The rest is plumbing.

---

### Decision 1: Trigger Philosophy

**What signals are worth waking the ninja?**

ZenOS dispatches your component in two ways:

- **Hourly sweep** — the scheduler wakes every KFC on a regular cadence. You get this for free. It means every component eventually converges on current state even if no trigger fires.
- **KFC Trigger automation** — your personal automation file that fires `summary_force` for your component on specific entity state changes. This is the instant path.

Not every entity change needs the instant path. The question is: *what can't wait an hour?*

**Alert Manager** answers "safety and security events." A smoke alarm firing cannot wait for the hourly sweep.

```yaml
# kfc_trigger_alert_manager.yaml (your installation, not the repo)
# Safety-critical = instant dispatch. Informational = hourly sweep covers it.

triggers:
  - trigger: state
    entity_id: binary_sensor.home_alarm_fire
    to: 'on'
    not_from: [unknown, unavailable]   # ← never wake on startup ghosts

  - trigger: state
    entity_id: switch.main_water_valve
    to: 'off'
    not_from: [unknown, unavailable]
    for:
      minutes: 2                       # ← debounce: a 2-second valve blip isn't a crisis
```

**Taskmaster** answers "lifecycle transitions." A bed sensor changing means someone just woke up or went to sleep — that's task-relevant now, not in an hour.

```yaml
# kfc_trigger_taskmaster.yaml
triggers:
  - trigger: state
    entity_id: binary_sensor.withings_nathan_in_bed
    not_from: [unknown, unavailable]   # no to: filter — both directions matter here
```

Notice the differences:
- Alert Manager uses `to: 'on'` for alarm sensors — the off state is nominal, silence it.
- Alert Manager uses `for:` on the water valve — some state changes need time to confirm they're real.
- Taskmaster uses no `to:` filter — both wake and sleep are meaningful transitions.
- Alert Manager uses `max: 5`; Taskmaster `max: 3`. Tune this to your expected burst rate.

**The `not_from: [unknown, unavailable]` pattern appears everywhere. Always include it.** HA emits state changes on restart and sensor initialization. Without this guard, your trigger fires on every HA boot, which is not what you want.

**Use `to:` when direction matters.** If you only care about one direction — an alarm firing, a door opening, a valve closing — filter it. A trigger without `to:` fires on *every* state transition including `unknown` and `unavailable` transitions that slip past `not_from`. If both directions are meaningful (like taskmaster's wake/sleep), omit `to:` intentionally and say so in a comment.

---

### Decision 2: Scope Definition

**What entities does your component see?**

The answer in ZenOS is: **exactly the entities you label.**

Label IS the scope. This is the most important design decision you make. A tight label means:
- The ninja only reads what it needs
- The summary is fast
- The kata is high-signal
- The downstream consumer doesn't have to filter noise

A loose label means your ninja is reading the whole house to summarize one domain. That's slow, expensive, and produces worse output.

**Alert Manager's scope:** entities labeled `alert_manager`. That's alarm sensors, leak sensors, smoke/CO, water valve, refrigerator power. Nothing else. When the ninja runs, it sees exactly the things that can cause harm if they change — and nothing that would dilute that picture.

**How to design your scope:**

Start with the question: *what is the minimal set of entities whose state completely describes my domain?* Add nothing else. If you're tempted to add an entity "for context," that entity probably belongs in a different KFC or is ambient context the downstream consumer already has.

**Scribe is the preferred tool for authoring and editing your KFC definition.** See [`zen_dojotools_scribe_readme.md`](../scripts/zen_dojotools_scribe_readme.md). Here's the kind of blurb you'd write in a Scribe `new_thought` when starting an alert-style component:

```
Component: alert_manager
Domain: safety and security monitoring
Scope: entities where state change represents a condition requiring immediate human awareness
Nominal: most entities should be OFF/OK at rest — alert when they deviate
Alert cadence: fire-once, clear on resolution, re-arm automatically
Output: active alert list with severity, fired_at, and message
```

That short description contains all five decisions in embryonic form. Scribe helps you formalize it into a drawer definition and then press it to Dojo when it's ready.

---

### Decision 3: Ninja Output Contract

**What does your kata commit to?**

The kata is not a summary. It is a **decision tree node** — a structured, typed, indexable snapshot that the knowledge tree can consume without calling your component again.

This distinction matters. A summary says "here's what's happening in prose." A decision tree node says "here is the current state of my domain in a form that enables downstream reasoning without re-reading my inputs."

Alert Manager's kata output contract:
- `active_alerts` — list of currently firing alerts with key, severity, fired_at, message
- `alert_count` — integer, enables quick threshold checks
- `highest_severity` — `info | warn | error`, enables routing decisions
- `nominal` — boolean, true when `alert_count == 0`

A downstream consumer (supersummary, Friday) can answer "is there an active emergency?" in one lookup. That's the point.

Design your output contract before you write the ninja. Ask: *what question does downstream need to answer using my kata, and what's the minimal schema that answers it?* Then commit to that schema and don't change it without versioning.

The knowledge tree is built from these leaves. Many small, tightly-typed leaves make a fast, accurate tree. A few large, loosely-typed leaves make a slow, noisy tree. **Store a lot of small, highly indexed bits.**

---

### Decision 4: Chain Control

**How do you prevent your component from blowing up the pipeline?**

This is where the summarizer gymnastics come from.

When your ninja emits a kata, it fires a `kata_emit` event. Other components can listen to that event. If they do, and if their response triggers *your* component again, you have a loop. In a naïve event-driven system, this expands unboundedly.

ZenOS has several layers of protection:

**Emission cooldown** — the ninja won't emit a new kata if one was already emitted recently. This prevents rapid-fire re-summaries from a burst of state changes all resolving at once.

**caller_id** — every FileCabinet write carries a `caller_token` identifying who initiated it. This is currently used for audit and debugging. In future versions, it becomes the foundation for chain gating: *a component initiated by X cannot re-invoke X within the same chain.* The call graph becomes inspectable and stoppable.

**Correlation ID (coming)** — future versions will thread a correlation ID through the entire summary chain. This gives you: full chain traceability, depth gating (stop after N hops), and loop detection (A→B→A is rejected). Design your component now with the assumption that this ID will exist — don't build logic that assumes isolation.

**What this means for your component:**

- Don't emit events that are likely to re-trigger your own summary. If you do, bound it with a cooldown.
- for now you may Use `caller_token:  unique_foo` if you need basic chain attribution this will improve. 
- Design your output contract (Decision 3) so consumers can make decisions *without* re-invoking you. The kata should be self-contained.
- `mode: queued` on your trigger automation is not style — it's the mechanism that prevents a burst of simultaneous trigger fires from spawning parallel ninja runs.

---

### Decision 5: What Downstream Gets

**How is your kata consumed?**

Your kata becomes a leaf in the ZenOS knowledge tree. The tree has tiers — see [`understanding_kf4.md`](understanding_kf4.md) for the full tier table and shedding thresholds. The two you'll choose between for most components:

- **`super` tier** (alert_manager, security, energy, water, zen_system, hot_tub, taskmaster, zen_fusion) — included in every supersummary. Friday sees these on every boot.
- **`ambient` tier** (weather, room_manager, laundry, dishwasher, grocy, media, autovac, trash_trakker, water_heater) — pre-digested by Trapper Keeper, aggregated into a compressed digest. Friday gets their signal through the ambient index rather than reading each kata directly. Shed on fast triggers and under queue pressure.

The default `pipeline_tier` for a new component is **`keeper`** — dispatched on all standard triggers, deferred only under heavy queue load. Most custom components start here and only move to `ambient` or `super` if usage warrants it.

Your component's tier placement determines how frequently its kata influences Friday's awareness. `super` = always present. `ambient` = present when relevant, compressed. `keeper` = the default — reliable, not foregrounded.

The compact index (Friday's boot payload `index` key) is a pre-built view across all katas. Design your kata fields to be indexable — flat keys with typed values, not nested prose. The index is how Friday answers "what's the state of X?" without tool calls.

**Designing for downstream:**

- Name your fields predictably. `nominal`, `active_count`, `last_run` are conventions — use them.
- Boolean `nominal` field is standard. `true` = everything expected, `false` = something needs attention. This is the first thing downstream checks.
- `component_summary` in your dojo drawer defines what `alert_when_on` and `alert_when_off` mean for your entities. Get this right — it's how the ninja knows what to flag.

---

## KF5: Self-Registering Tools

The decisions above still apply — trigger philosophy, scope, output contract, chain control, downstream tier. KF5 changes *how a component gets wired into the dojo*, not what it decides.

Under KF4, a component's dojo drawer is authored via Scribe (`new_thought` → `publish_kfc`) as a standalone artifact. Under KF5, a **tool self-declares** its KFC contract by implementing `mode=kfc_manifest`, and a generic bootstrap scanner (`zen_dojotools_manifest mode=bootstrap_kfc`) discovers it and writes the live drawer mount automatically — no Scribe authoring step required for tools that already exist as scripts.

**The pattern:**

1. Add `mode=kfc_manifest` to your tool. It returns `{status: ok, kfc: {...}}` where the `kfc` block is shaped like a dojo drawer definition (`kata_key`, `component_summary`, `component_instructions`, `seed`, `trigger_subscriptions`, `drift_threshold`, `meta.enabled`).
2. Inside that same `kfc_manifest` sequence, self-register your tool on the FC callout whitelist: `tool:kfc_manifest` — hardcoded to your own tool name and this mode only. No dynamic or open whitelist writes.
3. Add `script.zen_dojotools_yourtool` to the `_bkfc_known` list in `dojotools_manifest.yaml`.
4. Run `zen_dojotools_manifest mode=bootstrap_kfc` (or wait for its automatic firing on `homeassistant_start` + daily `00:01`).

**FC whitelist mode-scoping** (what makes this safe): the FileCabinet callout whitelist supports three entry formats:

| Format | Meaning |
|---|---|
| `tool` | Plain tool name, all modes reachable via a live drawer — **deprecated, backcompat only** |
| `tool:*` | All modes reachable — preferred for tools where every mode is FC-safe |
| `tool:mode` | Restricted to exactly one mode — **required for any tool with write-capable modes** |

A finance-domain tool, for example, is scoped `zen_dojotools_finance:kfc_manifest` — the live drawer can only ever reach that one read-only, self-describing mode, never a write path. `kfc_manifest` registering only its own hardcoded `tool:mode` pair (never a wildcard, never another tool's name) is what keeps self-registration from becoming an open write vector.

**Live drawer shape** written by `bootstrap_kfc` (both the FC mount and the breadcrumb fields the summarizer resolves at runtime):

```json
{
  "mount_point": true,
  "fc_args": {"tool": "zen_dojotools_yourtool", "params": {"mode": "kfc_manifest"}},
  "friendly_name": "Your Tool",
  "label": "Your Component",
  "kata_key": "your_component",
  "meta": {"enabled": true, "requires": {"cert": "", "level": 0}}
}
```

**`component_summary` is intentionally left out of the written drawer, but not out of the picture.** `bootstrap_kfc` still reads `component_summary` from your `kfc_manifest` response and returns it in its own result (`_bkfc_registered` entries) — so direct callers of `bootstrap_kfc` or `kfc_manifest` can see it. What it deliberately does NOT do is persist it into the cabinet drawer — this mirrors the space-saving convention Scribe already uses on the KF4 path (`trim_description`, see `dojotools_scribe.yaml`). The ninja summarizer resolves an empty `component_summary` from the HA label's description at read time instead (`dojotools_summarizers.yaml`, step 3d) — the summary lives once, on the label, not duplicated into every cabinet drawer. Set a real label description for your component; that's what the summarizer actually reads at runtime, not the drawer field.

Because the live drawer resolves `kfc_manifest` fresh on every ninja run, changing `component_instructions` and reloading scripts takes effect on the next run — no re-bootstrap needed.

KF4's Scribe-authored path and KF5's self-registration path are not mutually exclusive — both ultimately produce a dojo drawer the summarizer reads the same way. KF5 is the better fit when the KFC contract belongs to a tool that already exists as a script (Firefly, Finance, any `zen_dojotools_*`); Scribe authoring remains the path for components that aren't backed by an existing tool.

---

## Appendix: KFC Template

Use this as a starting scaffold. Replace everything in `<angle brackets>`. Scribe is the preferred tool for maintaining this definition after initial creation.

### Dojo Drawer (via Scribe `new_thought`)

```yaml
component: <component_name>
version: 1.0.0
description: <one line — what domain does this own?>

label: <ha_label_slug>          # this is your scope. make it tight.
domain_filter: []                # optional: restrict to specific HA domains within the label

trigger_philosophy: <event-based | lifecycle | sweep-only>
# event-based: fires on specific entity state changes (alarm, leak, etc.)
# lifecycle: fires on household lifecycle events (wake/sleep, arrival/departure)
# sweep-only: hourly cadence is sufficient — no KFC trigger automation needed

pipeline_tier: keeper            # keeper (default) | ambient | super — see understanding_kf4.md

output_contract:
  nominal: boolean               # true = domain is in expected state
  active_count: integer          # how many entities are in alert/active state
  # add your domain-specific fields here

alert_when_on: []               # entities where ON = something to flag
alert_when_off: []              # entities where OFF = something to flag
                                 # nominal when ON (e.g., refrigerator power)

component_summary: >
  <What does this component monitor? What does it flag? What is nominal?>
```

### KFC Trigger Automation (your installation — do not commit to repo)

```yaml
# kfc_trigger_<component_name>.yaml
# ⚠️ PERSONAL FILE — DO NOT COMMIT TO REPO
#
# Wire your specific entities here. The repo ships the tool; you wire your home.
#
# Entities wired:
#   <entity_id>    — <why this entity / what state matters>

automation:
  - id: '<unique_id>'
    alias: KFC Trigger — <Component Name>
    description: >-
      Fires a targeted force_summary for <component_name> on <what triggers it>.
    triggers:

      # -- <Trigger group description> ------------------------------------------
      - trigger: state
        entity_id:
          - <entity_id>
        to: '<state>'                  # omit only if both directions are meaningful
        not_from:                      # always include this
          - unknown
          - unavailable
        # for:                         # uncomment to debounce
        #   minutes: 2

    actions:
      - event: zen_event
        event_data:
          event:
            kind: summary_force
            component: <component_name>

    mode: queued
    max: 5                             # tune to expected burst rate
```

### Checklist Before You Publish

- [ ] Label defined in HA and entities tagged
- [ ] `alert_when_on` / `alert_when_off` populated for every entity in scope
- [ ] `component_summary` written — describes nominal state and what to flag
- [ ] `output_contract` fields are flat, typed, and include `nominal: boolean`
- [ ] Trigger automation uses `not_from: [unknown, unavailable]` on all triggers
- [ ] `to:` filter set (or intentionally omitted with a comment) on every trigger
- [ ] `for:` delays added for any state that needs debouncing
- [ ] `caller_token: <component_name>` on all FileCabinet writes in your tool (if applicable)
- [ ] `mode: queued` on trigger automation
- [ ] Scribe `publish_kfc` run — artifact_state must be `formal` before Dojo accepts it
