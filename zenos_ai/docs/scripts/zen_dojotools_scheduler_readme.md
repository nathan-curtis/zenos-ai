# Zen DojoTools Scheduler — 4.2.0

**File:** `packages/zenos_ai/dojotools/dojotools_scheduler.yaml`
**Automation:** `automation.zen_dojotools_scheduler`

---

## Overview

The Zen DojoTools Scheduler is the trigger orchestrator for the ZenOS-AI system. It fires the Ninja Summarizer pipeline, the Manifest, the SuperSummary, and FileCabinet GC based on time patterns, home state changes, and explicit force events.

As of 4.1.0 the Ninja Summarizer dispatch is **Dojo-driven** — components declare which triggers they care about in their KFC drawer, and the scheduler routes to them automatically. No hardcoded component branches, no scheduler edits required when you add a new component.

---

## Trigger IDs

These trigger IDs are active in the core scheduler. Components subscribe to them via `trigger_subscriptions` in their Dojo drawer.

| Trigger ID | Fires on |
|---|---|
| `daily_midnight` | 00:00 every day |
| `daily_noon` | 12:00 every day |
| `quarter_hour` | Every 15 minutes |
| `hourly_trigger` | Every hour |
| `every_10_minutes` | Every 10 minutes |
| `every_5_minutes` | Every 5 minutes |
| `ha_start` | HA startup |
| `home_mode_updates` | `sensor.zen_home_mode` state change |
| `start_home_wake` | `zen_home_mode` → Home-Wake |
| `start_home_evening` | `zen_home_mode` → Home-Evening |
| `home_occupancy_change` | `zone.home` count changes |
| `persistent_notification` | Notification added, removed, or updated |
| `door_opens_or_closes` | `group.all_doors` state change |
| `lock_changes` | `group.all_locks` state change |
| `window_opens_or_closes` | `group.all_windows` state change |
| `garage_door` | `group.garage_doors` state change |
| `force_summary` | `zen_event: {kind: summary_force}` fired |
| `force_gc` | `zen_event: {kind: gc_force}` fired |

---

## How Components Subscribe

Components register in their KFC drawer (via `zen_dojotools_kungfu_writer`). The relevant fields:

```yaml
trigger_subscriptions:
  - quarter_hour
  - door_opens_or_closes
  - lock_changes
delay_seconds: 30
kata_key: security_manager
```

- **`trigger_subscriptions`** — list of trigger IDs this component wants to fire on
- **`delay_seconds`** — how long to wait before running the Ninja Summarizer (skipped on `force_summary`)
- **`kata_key`** — the drawer key in the Kata cabinet where this component's summary is stored

When the scheduler fires, it reads every KFC drawer and dispatches the Ninja Summarizer for each component whose `trigger_subscriptions` includes the current `trigger.id`.

---

## Forcing a Summary

To force an immediate summary for a specific component, fire:

```yaml
event: zen_event
event_data:
  event:
    kind: summary_force
    component: security_manager   # optional — omit to run all subscribers
```

This hits the `force_summary` trigger in the scheduler. The `delay_seconds` wait is skipped on forced runs.

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

## Explicit Branches (Not Dojo-driven)

Three operations are hardcoded in the choose block and are not driven by `trigger_subscriptions`:

| Branch | Triggers |
|---|---|
| Manifest | `ha_start`, `daily_midnight` |
| SuperSummary | `daily_midnight`, `daily_noon`, `ha_start`, `every_10_minutes`, `force_summary` |
| FileCabinet GC | `daily_midnight`, `quarter_hour`, `force_gc` |

These are system-level operations and are not subscribable by components.

---

## Heartbeat

After every run, the scheduler writes a `zen_scheduler` drawer to the Kata cabinet with:

```json
{
  "status": "success",
  "trigger": {"id": "...", "platform": "...", "entity_id": "..."},
  "dispatched": ["component_a", "component_b"]
}
```

`sensor.zen_summarizer_health` reads this drawer. If the timestamp is stale (>20 minutes), it returns `warn`.

---

## Troubleshooting

**Scheduler fired but component didn't run**
- Check the component's KFC drawer — is `trigger_subscriptions` set and does it include the right trigger ID?
- Is `kata_key` set correctly?
- Pull a trace of `automation.zen_dojotools_scheduler` and check the `_subscribers` variable — if it's empty, the dispatch loop found no matching subscribers.

**`zen_summarizer_health` shows warn**
- Check `last_timestamp` attribute — is it stale?
- Check `ai_task_entity` attribute — if blank or unavailable, the scheduler can't dispatch. Set `input_text.zenos_ai_task_entity` to your conversation agent entity ID.

**Summaries are degraded, empty, or malformed**
- Check your inference server logs for context length errors. Each summarizer run ingests a supplemental prompt, prior Kata drawer content, and a full entity state snapshot. If the model is silently truncating input, output quality collapses. Increase your inference server's context allocation.
- Models under ~4B parameters do not perform reliably as the summarization backend. The summarizer does not require tool-calling capability — it needs strong summarization and JSON authoring skills.

> **⚠️ Do not use a paid inference API for the AI task entity.** The Ninja Summarizer fires multiple times per hour and the SuperSummary fires a minimum of 4 times per hour. Pointing `input_text.zenos_ai_task_entity` at a metered API will generate a significant and continuous token bill. Use a locally-hosted model for background summarization. Your frontline conversation agent operates on demand only and does not carry this risk.

**Spook ghost-entity warnings on scheduler**
- Hardware-specific entity IDs should not be in `dojotools_scheduler.yaml`. Move them to a local `zen_dojotools_scheduler_custom.yaml` (see Hardware Triggers section above).
