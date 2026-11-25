# Zen DojoTools Scheduler — v5.7.1

> **2026.12.0 (5.7.0):** the file now holds three automations and a launcher. The Scheduler (the Abbot) only decides who runs. [`zen_abbot_launch`](#the-launcher-zen_abbot_launch) starts the work, so a scheduler run takes seconds and no longer holds its queue slot through delays. [Housekeeping](#housekeeping) and [Safety Admit](#safety-admit) are their own automations. Each housekeeping job now runs on its own: they used to share one first-match `choose`, so display session expiry, the resolver refresh, both Rolodex stay checks, the Identity Manifest and the Compact Index never ran from here. Routine refreshes are a [`summary_request`](#requesting-a-run); [force is for maintenance only](#maintenance-forces). SuperSummary runs [every quarter hour, at start and on request](#supersummary-cadence). A component with no kata is [admitted at start](#cold-start), and start-up launches are spaced 30 s apart. Reload scripts first (that creates the launcher), then automations.

> **2026.11.1 (5.4.1):** the core trigger pack registers inline, in the run that finds it missing. It used to go through an event the queued scheduler handled only after that run, and a quarter-hour run that dispatches the summarizers takes minutes, so after a cabinet reset the registry came back minutes after the tick. It now lands at the tick.

> **2026.11.0:** The trigger registry: the system cabinet drawer `zen_trigger_registry` lists every trigger a KFC can subscribe to, and only the scheduler writes it. A pack registers with `zen_event` kind `trigger_pack_register` from automation context (a user or agent call is ignored); the core pack registers at start and after an automation reload, and again on the quarter-hour tick whenever `zenos_core` is missing from the registry or registered at an older version, so a fresh install, a cabinet reset or an upgrade fills it without a restart or reload. Six label triggers, idle until an entity carries the label: `zen_leak`, `zen_smoke`, `zen_door`, `zen_power`, `zen_appliance`, and `alarm_panel` (5.3.2: a panel labeled `alarm_panel` goes to triggered; Alert Manager subscribes to it). One device trigger, `print_done` (5.4.0): a Bambu Lab print finished or failed, from that integration's own `bambu_lab_event` (Home Assistant has no purpose trigger for printers). It is idle until a component subscribes; Print Shop does while its `job_end_summary` setting is on. Reload automations first, then scripts. The group-based `door_opens_or_closes`, `window_opens_or_closes`, `lock_changes` and `garage_door` are legacy; prefer `zen_door`. See [Triggers](../kung_fu/building_a_kfc.md#triggers).

*Automation — no `tool_manifest`, so the file header version is canonical.*

**File:** `packages/zenos_ai/dojotools/dojotools_scheduler.yaml`
**Automations:** `automation.zen_dojotools_scheduler` (queued, max 15), Zen DojoTools Housekeeping (queued, max 10), Zen Safety Admit (parallel, max 10). All three declare `max_exceeded: warning`, so a run past max is dropped with a log warning, never silently.
**Script:** `script.zen_abbot_launch` (parallel, max 40, not MCP-exposed)

---

## Overview

The Zen DojoTools Scheduler (the Abbot) is the cognitive queue: it decides which components' summarizers run, and when. It only admits. Each admitted component goes to its own [`zen_abbot_launch`](#the-launcher-zen_abbot_launch) run, which does the waiting and starts the Ninja Summarizer.

Maintenance stays off the cognitive queue. The deferred reloads, the Manifest, the Identity Manifest, the Compact Index, FileCabinet GC, display session expiry, the resolver refresh and the Rolodex stay checks run in [Housekeeping](#housekeeping). Leak, alarm and smoke/CO/gas events go to [Safety Admit](#safety-admit), which starts their subscribers at once.

The Ninja dispatch is **Dojo-driven**: components declare the triggers they care about in their KFC drawer, and the scheduler routes to them. There are no hardcoded component branches, and adding a component needs no scheduler edit.

---

## Trigger IDs

These trigger IDs are active in the core scheduler. Components subscribe to them via `trigger_subscriptions` in their Dojo drawer. The registry drawer `zen_trigger_registry` is the full list, with the label triggers and any registered packs.

| Trigger ID | Fires on |
|---|---|
| `daily_midnight` | 00:01 every day |
| `daily_noon` | 12:00 every day |
| `quarter_hour` | Every 15 minutes |
| `hourly_trigger` | Every hour |
| `every_10_minutes` | Every 10 minutes |
| `ha_start` | HA startup |
| `home_mode_updates` | `sensor.zen_home_mode` state change |
| `start_home_wake` | `zen_home_mode` → Home-Wake |
| `start_home_evening` | `zen_home_mode` → Home-Evening |
| `home_occupancy_change` | `zone.home` count changes |
| `persistent_notification` | Notification added, removed, or updated |
| `vacuum_started` / `vacuum_docked` | An `autovac`-labeled vacuum starts cleaning / returns to its dock |
| `zen_door`, `zen_power`, `zen_appliance`, `print_done` | Label and device triggers (see the 2026.11.0 note above) |
| `door_opens_or_closes` | `group.all_doors` state change (legacy; prefer `zen_door`) |
| `lock_changes` | `group.all_locks` state change |
| `window_opens_or_closes` | `group.all_windows` state change (legacy; prefer `zen_door`) |
| `garage_door` | `group.garage_doors` state change |
| `summary_request` | `zen_event: {kind: summary_request, component: <key>}`: a routine refresh of one component (see [Requesting a run](#requesting-a-run)). Not subscribed to |
| `supersummary_request` | `zen_event: {kind: supersummary_request}`: a component asks for the rollup |
| `force_summary` | `zen_event: {kind: summary_force}`: maintenance only (see [Maintenance forces](#maintenance-forces)) |
| `force_ninja` | `zen_event: {kind: ninja_force}`: maintenance; runs Ninja dispatch only (remaps to `ha_start` scope), skips delay |
| `force_supersummary` | `zen_event: {kind: supersummary_force}`: maintenance; runs SuperSummary only |

`zen_leak`, `alarm_panel` and `zen_smoke` are subscribable too, but [Safety Admit](#safety-admit) handles them, not this queue. `force_gc` and the deferred reloads belong to [Housekeeping](#housekeeping).

---

## How Components Subscribe

Components register in their KFC drawer (via `zen_dojotools_scribe`). The relevant fields:

```yaml
trigger_subscriptions:
  - quarter_hour
  - door_opens_or_closes
  - lock_changes
delay_seconds: 30
kata_key: security_manager
```

- **`trigger_subscriptions`** — list of trigger IDs this component wants to fire on
- **`delay_seconds`** — how long after the trigger to run the Ninja Summarizer. It is an offset from the trigger, not a wait after the previous subscriber, and the launcher sleeps it, not the scheduler. It is skipped on `summary_request`, `force_summary` and `force_ninja`. On `ha_start`, launches are spaced 30 s apart: each component waits the larger of its delay and its slot, so a boot doesn't overflow the Ninja's queue. `super` and `system` tier components have no delay.
- **`kata_key`** — the drawer key in the Kata cabinet where this component's summary is stored

When the scheduler fires, it reads every KFC drawer and admits each component whose `trigger_subscriptions` includes the current `trigger.id`, then applies the shed rules:

- **Tier rule:** `ambient` components are shed on `quarter_hour` and `every_10_minutes`.
- **Depth gate:** the pressure is [`sensor.zen_scheduler_queue_depth`](../sensors/readme.md). Per component, `ambient` is shed at `shed_ambient_at` (default 4) and `keeper` at `shed_keeper_at` (default 8), from `zen_scheduler_config` in the household cabinet. At a pressure of 5 or more, only `system` and `super` are admitted, except on `daily_midnight` and `daily_noon`. `system` and `super` are never shed by depth.

A shed component isn't lost: the [drain router](zen_dojotools_dispatcher_readme.md) recovers it from its kata age.

### Cold start

On `ha_start`, an enabled component drawer that isn't a cache slot (`role: cache`) and has no kata at all is admitted too, through the same gates and spacing. A fresh install, a reset or wiped kata fills in at the next start.

---

## The launcher: `zen_abbot_launch`

The scheduler hands its work to `script.zen_abbot_launch`, one parallel run per job, so its own queue slot is free in seconds. It is internal: not MCP-exposed, and not for direct calls.

| Job | What it does |
|---|---|
| `ninja` | Sleeps the component's delay, reads its prior kata (with `events[]` older than 24 hours dropped), starts the Ninja Summarizer |
| `supersummary` | Waits for the Ninja burst to settle (fewer than 2 runs, 30 s ceiling), then starts SuperSummary. A `supersummary_request` skips the wait: its kata is already written |
| `heartbeat` | Waits for FileCabinet to be idle (60 s ceiling), then writes the `zen_scheduler` drawer |

---

## Requesting a run

**`summary_request`** is the routine way to refresh one component now: a trigger pack, a tool after a change, the drain router, GC after an eviction. It admits the named component whether or not it subscribes, through the normal gates (tier rule, depth gate), and runs it without its declared delay.

```yaml
event: zen_event
event_data:
  event:
    kind: summary_request
    component: security_manager   # required: a request names one component
    schedule: <sub_kata_key>      # optional: one sub-schedule of a component with schedules
```

**`supersummary_request`** lets a component ask for the rollup. The Ninja sends it when a run's urgency reaches the component's `drift_threshold`. If a rollup ran within `super_burnout_seconds` (default 300), SuperSummary drops it, and the next quarter-hour tick picks the change up.

```yaml
event: zen_event
event_data:
  event:
    kind: supersummary_request
```

## Maintenance forces

Forces are for special-purpose maintenance, never for normal operation. Use `summary_request` for routine work.

**`summary_force`** — Dispatches the subscribers of `force_summary` and skips their delay. With `component:`, it admits only that component, only if it subscribes to `force_summary`, and bypasses the shed rules.

```yaml
event: zen_event
event_data:
  event:
    kind: summary_force
    component: security_manager   # optional — omit to run all force_summary subscribers
```

**`ninja_force`** — Runs Ninja dispatch only. Remaps the trigger scope to `ha_start` so all components that subscribe to `ha_start` fire. Delay is skipped. SuperSummary does not run.

```yaml
event: zen_event
event_data:
  event:
    kind: ninja_force
```

**`supersummary_force`** — Runs SuperSummary only. Ninja dispatch is skipped entirely.

```yaml
event: zen_event
event_data:
  event:
    kind: supersummary_force
```

`zen_pipeline_autofire_on_enable` uses `ninja_force` and `supersummary_force` when kill switches are re-enabled, and the manual emit tools fire forces too. Those are maintenance.

`identity_manifest_rebuild` (rebuild `zen_identity_manifest` now) is handled by the [Dispatcher](zen_dojotools_dispatcher_readme.md), not this file. The Identity Manifest also rebuilds in Housekeeping at `ha_start` and `daily_midnight`.

---

## Cert Catalog Refresh

The deferred script reload and `reload_all` also rebuild the cert catalog cache in the vault (`zen_dojotools_identity mode=cert_cache_refresh`, forced), so a reload that changes a tool's certs shows in `cert_list` at once.

---

## Stuck-Run Guard

If a model backend stops answering mid-call, the summarizer script waits forever and never takes another run. Normal runs take seconds (cloud) to a few minutes (local), so a `supersummary` or `ninja_summarizer` run older than 15 minutes is stopped and a warning is logged, letting the next dispatch start. The queue-depth gate sheds frequent ticks when the queue backs up; the daily anchors always dispatch, since they fire once with no retry.

---

## Prior-Run Events Stale-Out

Before a component's previous Kata is fed forward as `last_kata_summary`, any `events[]` entry older than 24 hours is dropped. Garbage collection expires whole drawers by their `expires_after` metadata and cannot see entries inside one, so without this a model could keep echoing a self-authored event forward indefinitely. It is a template filter on each event's own timestamp, not a prompt instruction.

---

## Protected Drawers

The following drawers are in the scheduler's `_protected` list — the Ninja Summarizer will not attempt to summarize them:

- `zen_summary`
- `zen_library_manifest`
- `zen_identity_manifest`

Both the variables block and the `_subscribers` Jinja template include this list. If you add new system drawers that should not be summarized, update both locations.

---

## Hardware Triggers (Custom)

The core scheduler only includes generic triggers. Hardware-specific entity triggers (alarm panel, bed sensors, water flow, electrical panels, etc.) are **not** in core — they would cause Spook ghost-entity warnings on any install without that hardware.

To add hardware triggers for your install:

1. Copy `zenos_ai/docs/examples/zen_dojotools_scheduler_custom.yaml` to `<ha_config>/packages/zenos_ai/`
2. Rename it (e.g. `zen_dojotools_scheduler_custom.yaml`)
3. Replace the placeholder entity IDs with your real hardware entities
4. Add it to your local `.gitignore` — this file contains your entity IDs and should never be committed
5. Subscribe your components to the trigger IDs as normal via `trigger_subscriptions`

The example file uses the same Dojo-driven dispatch loop as the core scheduler, so your KFC drawer subscriptions work identically.

---

## SuperSummary cadence

SuperSummary runs on `quarter_hour` (the hourly, noon and midnight ticks land on one), on `ha_start`, on a component's `supersummary_request` and on `force_supersummary`. A run that only dispatched components doesn't start it. Its own governor, `super_burnout_seconds` in `zen_ninja_config` (default 300), is the minimum gap: a request inside it is dropped, not queued.

---

## Housekeeping

**Zen DojoTools Housekeeping** runs the system jobs, which components can't subscribe to. Each job is its own `if`, started as its own run with `continue_on_error`, so one that fails or runs long holds up none of the others.

| Job | Triggers |
|---|---|
| Deferred script reload, deferred reload all (each also rebuilds the cert catalog cache) | `zen_event` `deferred_script_reload` / `deferred_reload_all` |
| Manifest, Identity Manifest, Compact Index | `ha_start`, `daily_midnight` |
| FileCabinet GC | `daily_midnight`, `quarter_hour`, `force_gc` |
| Display session expiry | `quarter_hour` |
| Resolver refresh (`zen_resolver_refresh`) | `quarter_hour`, `ha_start` |
| Rolodex stay prep and checkout checks (only when Rolodex is installed) | `quarter_hour` |

A script reload no longer waits behind the Ninja dispatch.

---

## Safety Admit

**Zen Safety Admit** (parallel) takes the life-safety label triggers: `zen_leak` (a leak sensor detects water), `alarm_panel` (a labeled panel is triggered) and `zen_smoke` (smoke, carbon monoxide or gas). It starts the Ninja for every component subscribed to that trigger at once: no queue behind routine work, no depth gate, no delay. The deterministic responders (the leak watch, the panel, Security Manager) are separate automations; this is only the agent's awareness of the event.

---

## Heartbeat

After every run, the scheduler hands the launcher a `zen_scheduler` drawer for the Kata cabinet. It includes:

```json
{
  "status": "success",
  "trigger": {"id": "...", "platform": "...", "entity_id": "..."},
  "dispatched": ["component_a", "component_b"],
  "pressure": 1,
  "shed_tier": [],
  "shed_depth": []
}
```

- `pressure`: `sensor.zen_scheduler_queue_depth` at admission.
- `shed_tier`: components the tier rule dropped this run.
- `shed_depth`: components the depth gate dropped this run.

`sensor.zen_summarizer_health` reads this drawer. If the timestamp is stale (>20 minutes), it returns `warn`.

---

## Troubleshooting

**Scheduler fired but component didn't run**
- Check the component's KFC drawer — is `trigger_subscriptions` set and does it include the right trigger ID?
- Is `kata_key` set correctly?
- Pull a trace of `automation.zen_dojotools_scheduler` and check the `_subscribers` variable — if it's empty, the dispatch loop found no matching subscribers.
- Check the heartbeat's `shed_tier` and `shed_depth`: the component may have been shed.
- The scheduler only admits. If it admitted the component, look at the `script.zen_abbot_launch` trace for that run.

**`zen_summarizer_health` shows warn**
- Check `last_timestamp` attribute — is it stale?
- Check `ai_task_entity` attribute — if blank or unavailable, the scheduler can't dispatch. Set `input_text.zenos_ai_task_entity` to your conversation agent entity ID.

**Summaries are degraded, empty, or malformed**
- Check your inference server logs for context length errors. Each summarizer run ingests a supplemental prompt, prior Kata drawer content, and a full entity state snapshot. If the model is silently truncating input, output quality collapses. Increase your inference server's context allocation.
- Models under ~4B parameters do not perform reliably as the summarization backend. The summarizer does not require tool-calling capability — it needs strong summarization and JSON authoring skills.

> **⚠️ Do not use a paid inference API for the AI task entity.** The Ninja Summarizer fires multiple times per hour and the SuperSummary fires a minimum of 4 times per hour. Pointing `input_text.zenos_ai_task_entity` at a metered API will generate a significant and continuous token bill. Use a locally-hosted model for background summarization. Your frontline conversation agent operates on demand only and does not carry this risk.
>
> **The summarizers ship disabled by default.** Enable them via `input_boolean.zen_summarizers_enabled` (master), `input_boolean.zen_ninja_summarizer_enabled`, and `input_boolean.zen_supersummarizer_enabled` only after you have confirmed your AI task entity points at a local model.

**Spook ghost-entity warnings on scheduler**
- Hardware-specific entity IDs should not be in `dojotools_scheduler.yaml`. Move them to a local `zen_dojotools_scheduler_custom.yaml` (see Hardware Triggers section above).
