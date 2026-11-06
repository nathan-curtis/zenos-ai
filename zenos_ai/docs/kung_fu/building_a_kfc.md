# Building a ZenOS KFC Component

**Audience:** Developers building a Kung Fu Component for ZenOS-AI.
**Spec:** KF6 (2026.11.0). It rolls KF4 (the authored drawer) and KF5 (the self-registering tool) into one contract, and adds the security contract the pipeline runs under since 2026.11.0.
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
    entity_id: binary_sensor.withings_alex_in_bed
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

Your kata becomes a leaf in the ZenOS knowledge tree. The tree has tiers — see [`understanding_kf6.md`](understanding_kf6.md) for the full tier table and shedding thresholds. The two you'll choose between for most components:

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

<a id="kf5-self-registering-tools"></a>

## KF6: Two Ways In, One Contract

The decisions above apply however a component gets into the Dojo. KF6 (2026.11.0) rolls the two earlier patterns into one spec: KF4's authored drawer and KF5's self-registering tool are now two ways in to the same contract, and both answer to the security contract below.

Both paths end in a Dojo drawer the scheduler and the Ninja read the same way. The Dojo takes writes only from internal callers (its ACL class gives `write` and `delete` to `@internal`), so a drawer gets there through Scribe or `bootstrap_kfc`, never through a direct FileCabinet write. On an install that enforces, an agent's direct write to the Dojo is refused; on one that audits, it's logged.

### Path A: an authored drawer (Scribe)

For a component no tool owns. Write it with Scribe (`new_thought`, then `formalize_scroll`, then `publish_kfc`); `mode=schema` returns the live `kfc_template` every drawer is checked against. See [`zen_dojotools_scribe_readme.md`](../scripts/zen_dojotools_scribe_readme.md).

### Path B: a self-registering tool (`kfc_manifest`)

For a component a tool owns. The tool declares its own KFC, and `zen_dojotools_manifest mode=bootstrap_kfc` writes a live drawer mount that calls the tool fresh on every Ninja run. No Scribe step. The full pattern is in [`zen_kf6_pattern_readme.md`](../scripts/zen_kf6_pattern_readme.md).

1. Add `mode=kfc_manifest` to your tool. It returns `{status: ok, kfc: {...}}` where the `kfc` block is shaped like a dojo drawer definition (`kata_key`, `component_summary`, `component_instructions`, `seed`, `trigger_subscriptions`, `drift_threshold`, `meta.enabled`). `meta` also accepts optional `notify_tag`/`notify_response_type`: set both to get an interactive Postman push (a tag plus a response type, e.g. a single "Got it" acknowledge button) on your component's `suggested_act_event` emissions instead of the default fire-and-forget notify every component gets when these are left unset.
2. List `kfc_manifest` among the modes in your `tool_manifest`, or tag the script `zen_kfc_provider`. `bootstrap_kfc` finds providers either way and tags any that lack the label. There is no list to edit.
3. If your component's `seed` calls your own tool, list that mode in your `tool_manifest` `seed_modes` (see the security contract below).
4. Run `zen_dojotools_manifest mode=bootstrap_kfc`, or wait for it: it runs on `homeassistant_start` and daily at `00:01`.

**Your tool doesn't touch the callout whitelist.** A live drawer can only call a tool the FileCabinet callout whitelist allows, and the whitelist lives in the system cabinet, which only the admin door writes. `bootstrap_kfc` registers one entry per provider, `<tool>:kfc_manifest`, through `prompt_loader mode=whitelist`. The whitelist supports three entry formats:

| Format | Meaning |
|---|---|
| `tool` | Plain tool name, all modes reachable via a live drawer. **Deprecated, backcompat only** |
| `tool:*` | All modes reachable. For tools where every mode is FC-safe |
| `tool:mode` | Restricted to exactly one mode. **Required for any tool with write-capable modes** |

The `tool:kfc_manifest` entry means the live drawer can only ever reach that one read-only, self-describing mode, never a write path.

**Live drawer shape** written by `bootstrap_kfc` (the FC mount plus the breadcrumb fields the scheduler and prompt loader read raw):

```json
{
  "mount_point": true,
  "fc_args": {"tool": "zen_dojotools_yourtool", "params": {"mode": "kfc_manifest", "component": "your_component"}},
  "friendly_name": "Your Tool",
  "label": "Your Component",
  "seed": {},
  "trigger_subscriptions": "hourly_trigger,force_summary",
  "drift_threshold": 6,
  "emission_cooldown_minutes": 1440,
  "delay_seconds": 0,
  "kata_key": "your_component",
  "pipeline_tier": "keeper",
  "meta": {"enabled": true}
}
```

`meta` is your `kfc_manifest`'s own `meta`, or `{"enabled": true}` if it declares none.

**`component_summary` is intentionally left out of the written drawer, but not out of the picture.** `bootstrap_kfc` still reads `component_summary` from your `kfc_manifest` response and returns it in its own result (`_bkfc_registered` entries), so direct callers of `bootstrap_kfc` or `kfc_manifest` can see it. What it deliberately does NOT do is persist it into the cabinet drawer, the same space-saving convention Scribe uses on the authored path (`trim_description`, see `dojotools_scribe.yaml`). The ninja summarizer resolves an empty `component_summary` from the HA label's description at read time instead (`dojotools_summarizers.yaml`, step 3d): the summary lives once, on the label. Set a real label description for your component; that's what the summarizer actually reads at runtime, not the drawer field.

Because the live drawer resolves `kfc_manifest` fresh on every ninja run, changing `component_instructions` and reloading scripts takes effect on the next run, with no re-bootstrap needed.

Path B is the better fit when the KFC contract belongs to a tool that already exists as a script (Firefly, Finance, any `zen_dojotools_*`); Path A remains the path for components that aren't backed by an existing tool.

---

## KF6: The Security Contract (2026.11.0)

Before 11.0 the pipeline ran as nobody and answered to two lists in the system cabinet. In KF6 a component runs as a named agent, reads only what a tool declares, and acts only with a grant a person approved for the background. The full story is in the [release notes](../releases/spinal_tap.md#the-pipeline-gets-a-pass).

**It runs as the agent responsible for the household:** the prime in the household's ACL, else the default agent. That agent must be onboarded, or the run refuses with `agent_not_onboarded`, summaries pause, and a notification says why. SuperSummary writes as the same agent.

**Seeds are declared by the tool.** A `seed` that calls a tool must name a mode that tool lists in its `tool_manifest` `seed_modes` (each `mode` or `mode:case`, optionally tagged with a read cert; see [Read certs on seeds](#read-certs-on-seeds)), or the run stops before the model is called with `seed_tool_not_authorized`, naming the modes that are declared. Index, Room Manager, AutoVac, Taskmaster, Inventory and Finance declare theirs. A `kata_tier` seed reads the kata cabinet directly and calls no tool.

**Acts need the emit stage.** A grant says where it may be used, one letter per pipeline stage (`i` interactive, `l` live drawer, `s` Ninja seed, `e` Ninja emit, `m` SuperSummary, `c` scheduler). Each tool declares the stages its certs allow (`st_ok`). Before the Ninja acts on `suggested_act_event`, it finds the cert on the target tool whose `st_ok` includes `e`, and the responsible agent must hold that cert with stage `e`. Today that's Postman's `zenos.comms.dispatch`, granted through the `zenos_agent_autonomous` bundle. A refused act doesn't happen and lands on the kata as `act_blocked`, with what to grant. A life-safety act skips the cert check; the kill switch and the infrastructure deny still apply.

**The 11 bridge.** Through 11, the old seed whitelist still passes a tool that declares no `seed_modes`, and the old act whitelist still passes an act whose cert the agent holds for interactive use only. Both are stamped legacy. Both lists, and the bridge, are removed in 12. Declare `seed_modes` and `st_ok` now.

**Minimum level (`meta.requires`).** A drawer's `meta.requires: {cert, level}` names a cert, and the minimum level, the responsible agent must hold, granted for stage `s` (Ninja seed), for the component to run. Empty (`cert: ""`, `level: 0`) is no requirement, and that's every shipped component.
- **The Ninja checks it** after the onboarded check and before the seed and the model, however the run started. A refusal is `kfc_cert_required`: the last kata stays, a `zen_event` (`pipeline_component_refused`) and a notification (`zen_kfc_cert_required_<slug>`) say what's missing, and the answer carries `required {cert, level, stage}` and `held {cert, level, stages}`. The 11 cert-name alias applies.
- **An unmet component stays out of the prompt,** as a disabled one does: out of `dojo_loader()`, SuperSummary's feed and Trapper Keeper's `kata_tier` seed. Its kata is kept, and it comes back once the cert is granted.
- **The cert must allow stage `s`.** A tool declares that in its `tool_manifest` `certs_required` (`s` in `st_ok`), or a grant can't ask for the stage. No shipped tool allows `s` yet, so a component that adopts a minimum level needs a tool that declares it.
- **Set it in Scribe** with `requires_cert` and `requires_level`. Scribe checks every write that sets it against the cert catalog and refuses with `kfc_requires_invalid`, writing nothing, when no tool declares the cert, its tools don't allow `s`, or the level is above its `max_level`.
- **A self-registering tool** declares it in its `kfc_manifest` `meta`; `bootstrap_kfc` rewrites the mount when it changes.

**The tool's own gate still applies.** An agent without `zenos.agent.basic` still gets a tool's `help`, `tool_manifest` and `kfc_manifest`, so a live mount resolves before onboarding; anything else answers `onboarding_required`. Calls no user started (the scheduler, automations) pass.

---

## KF6: Acts, Chains, Read Certs and Triggers (2026.11.0)

The rule behind all of this is that a tool says what it does and what it needs, in its own `tool_manifest`, so nobody hand-writes it twice. A capability ships switched off, behind a cert the tool declares, and turns on only when a person approves it on a real device. The grant history then shows exactly what the agent can do in the background.

### Declared acts

- **A tool declares its act** in `tool_manifest`, next to its emit cert (the cert whose `st_ok` has `e`): `act: {mode, args: {<name>: {enum?, required?, description?}}, description}`.
- **The model sees the declaration.** When a KFC names a tool in `suggested_act_event`, the Ninja reads that tool's `act` and tells the model which args it may return (`suggested_act_args`). A KFC doesn't need a hand-written list of available actions.
- **The act is checked against it.** Only declared keys, only enum values, every required key present. Otherwise the act is refused with `no_act_declared` or `act_args_invalid` and recorded on the kata as `act_blocked`, with `args_errors` listing each problem. Then the emit cert is checked as before.
- **The dispatcher routes declared acts.** A tool with no hand-written route is called if it declares an act: the dispatcher reads the declaration itself and calls only that mode, with only the declared args (plus `kata_input`). The payload never picks the mode. A hand-written route always wins.
- **Shipped acts:** Camera `scan` (`label: exterior|interior`, required), behind `zenos.security.camera_scan`; Postman `resolve_and_dispatch`, behind `zenos.comms.dispatch`.

### Chaining summaries

- **A summary can start another summary** through the Ninja's own act, `run` (`kung_fu_component_id` required), behind `zenos.pipeline.chain` (stage `e` only).
- **A KFC lists the targets it may start in `chain_to`.** The Dojo takes writes only from internal callers, so only the KFC's author sets it. A target outside `chain_to` is refused (`act_args_invalid`).
- **Each chained run carries `chain_path`,** set by the Ninja, never by the model. A component already on the path is refused (`chain_loop`), and the chain stops 3 deep (`chain_depth`). Each hop passes every gate on its own.
- A chained run is handed off, not waited on.

### Read certs on seeds

- **A tool can say which cert reads each seed mode:** a `seed_modes` entry may carry `@cert:level`, for example `'run:budget_summary@zenos.finance.read:1'` (level defaults to 1). The part before `@` is the mode; an exact `mode:case` entry wins over a bare `mode`.
- **Tagged today:** Finance (`zenos.finance.read`, four `run` cases), Room Manager `home_overview` (`zenos.presence.read`) and Taskmaster `briefing:medications` (`zenos.health.read`). Each cert allows stages `is`, so asking in conversation isn't gated; only the background read is.
- **The switch is `seed_cert_mode`,** like the FileCabinet lock, set from a console session with `zen_admintools_prompt_loader mode=seed_cert_mode` (`action_type` `status`, `audit` or `enforce`). Missing means audit.
  - **Audit (every upgrade):** a missing read cert never stops a run. The Ninja logs `seed_cert_audit` (`would_deny`), and nothing leaves the prompt. A working install does nothing.
  - **Enforce (a fresh install; Flynn's first `complete` writes it):** a seed through a tagged mode needs the cert at that level with stage `s`, or the run returns `seed_cert_required` before the tool and the model are called. The kata is stamped `seed_requires`, and every prompt feed drops a kata whose stamp the agent doesn't meet, so turning a cert off pulls the data out of the prompt too.
- **One approval for parity.** `cert_audit` builds `zenos_agent_parity` from what actually runs: the read certs your enabled summaries need, plus the emit cert of any act still carried by the 11 act bridge. Granting it (stages `ise` by default; each cert gets only what its tool allows) keeps everything running. Under enforce, the first refusal starts one parity ask, and Flynn's `complete` asks for whatever is missing.

### Turning a component on

- **On and off is `meta.enabled`, only** (Scribe `mode=enable` / `disable`). Master switches are gone: the field is ignored and stored empty. Nothing in code ever read one.
- **Scribe's preflight** runs on `enable`, `publish_kfc` and `republish_kfc` and lists every cert the run will need: `meta.requires` at `s`, the seed's read cert at `s`, the act's emit cert at `e`, and the chain cert at `e`. Each is checked against the agent that will run it, with `held`, `enforced`, `missing` and the `grant` call. It also reports `act_problem`, `triggers_unregistered`, `triggers_frequent` and `never_fires`.
- **Turning a component on requests what's missing,** for the agent that will run it. You approve each request on your phone. Turning it off and on again retries an approval that expired or was declined. Turning it off keeps its certs; `disable` says how to revoke one.
- **`zen_dojotools_manifest mode=component_certs`** shows one row per component, on or off: every cert it needs, whether the runner holds it, and a status: `off`, `refused`, `never_fires`, `act_broken` (the act tool is missing, or a chain act has no `chain_to`), `act_blocked`, `missing_audit` or `ready`.

### Triggers

- **The registry.** The system cabinet drawer `zen_trigger_registry` lists every trigger a KFC can subscribe to: `{packs, triggers: {<id>: {pack, fires, frequency, labels, dispatch}}}`. Only the scheduler writes it. `frequency` is `continuous`, `minutes`, `hourly`, `daily` or `event`; `minutes` and `continuous` cost a model call each time they fire.
- **Scribe `list_triggers` reads it** (`source: registry`), with each trigger's pack, plain-words `fires`, frequency and labels. A KFC subscribed only to names nobody registers shows as `never_fires`. A write is never refused for an unregistered trigger, so custom triggers still work.
- **Label triggers need no YAML.** These fire on Home Assistant purpose triggers aimed at a label, and sit idle until an entity carries it:

| ID | Fires when | Label it watches |
|---|---|---|
| `zen_leak` | a moisture sensor detects water | `zen_leak` |
| `zen_smoke` | smoke, CO or gas is detected | `zen_smoke` |
| `zen_door` | a door or window opens or closes | `zen_door` |
| `zen_power` | a power sensor stays below 1 W for 30 s (a fed circuit went dead) | `zen_power` |
| `zen_appliance` | power rises above 10 W (started), or stays below 3 W for 2 minutes (finished) | `zen_appliance` |

  A label follows the entity live, so labeling a sensor is enough; no reload. Prefer `zen_door` over the older group-based `door_opens_or_closes`, `window_opens_or_closes`, `lock_changes` and `garage_door`, which never fire on an install without `group.all_doors` and its siblings.

- **The whole loop, with no YAML:** install a leak sensor; the agent labels it `zen_leak`; through Scribe it writes a KFC subscribed to `zen_leak` with what normal and abnormal look like; the sensor fires and the Ninja runs it; the KFC acts through Postman under its emit cert. Labeling needs the Labels cert, the KFC write needs `kfc_publish` or a live yes, and the act needs its emit cert.
- **A trigger pack of your own registers itself** from an automation at Home Assistant start:

```yaml
- event: zen_event
  event_data:
    event:
      kind: trigger_pack_register
      pack: my_pack              # slug
      version: "1.0.0"
      triggers:
        my_trigger_id:
          fires: "plain words: when it fires"
          frequency: event       # continuous|minutes|hourly|daily|event
          labels: [my_label]     # optional, label-driven only
          dispatch: subscribe    # or targeted
```

  A pack's registration replaces its own earlier entries. Only automation context is accepted: a registration from a user or agent call is ignored, so the agent can't invent triggers. `dispatch: subscribe` means the scheduler runs every subscribed KFC, so your automation must also run the Dojo-driven dispatch (see [the custom scheduler example](../examples/zen_dojotools_scheduler_custom.yaml)); `targeted` means your pack calls one component itself and is listed for visibility only.

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

pipeline_tier: keeper            # keeper (default) | ambient | super — see understanding_kf6.md

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
- [ ] No `master_switch`: on and off is `meta.enabled`
- [ ] Path A: Scribe `publish_kfc` run — artifact_state must be `formal` before Dojo accepts it
- [ ] Path B: `kfc_manifest` declared (or the script tagged `zen_kfc_provider`), and `bootstrap_kfc` lists your `kata_key` in `registered[]`; no whitelist write in your tool
- [ ] Drawer written by Scribe or `bootstrap_kfc`, never by a direct FileCabinet write (the Dojo takes writes only from `@internal`)
- [ ] A `seed` that calls a tool names a mode in that tool's `tool_manifest` `seed_modes`
- [ ] A `suggested_act_event` target has a cert whose `st_ok` includes `e`, and the household's agent holds it with stage `e` (`zenos_agent_autonomous`)
- [ ] `meta.requires` left empty unless the component needs a minimum level; if set, its cert allows stage `s` and the agent holds it with `s`
- [ ] A tool your KFC acts through declares its `act`; a chaining KFC lists its targets in `chain_to`
- [ ] `trigger_subscriptions` names registered triggers (Scribe `list_triggers`); a labeled device uses a label trigger, not a hand-written automation
- [ ] Turned on with Scribe `enable`, and `component_certs` shows it `ready`
