# 18. The Abbot

The Monastery decides what a summary says. The Abbot decides when the system thinks at all. It is not one component: it is the name for the scheduling and dispatch function, carried out jointly by two automations.

| File | Automation | Job |
|---|---|---|
| `dojotools_scheduler.yaml` | `zen_dojotools_scheduler` | Fires on schedules and state changes, decides which components run, sheds load |
| `dojotools_dispatcher.yaml` | `zen_dojotool_dispatcher` | Routes correlated tool calls over the event bus, and hosts the support routers |

## 18.1 Triggers

The Scheduler owns a fixed set of trigger IDs:

| Kind | Triggers |
|---|---|
| Time | `every_10_minutes`, `quarter_hour`, `hourly_trigger`, `daily_noon`, `daily_midnight` |
| Home | `home_mode_updates`, `home_occupancy_change`, `start_home_wake`, `start_home_evening` |
| Openings | `door_opens_or_closes`, `window_opens_or_closes`, `garage_door`, `lock_changes` |
| Devices | `vacuum_started`, `vacuum_docked`, `persistent_notification` |
| System | `ha_start`, `force_summary`, `force_ninja`, `force_supersummary`, `force_gc`, `force_identity_manifest`, `deferred_script_reload`, `deferred_reload_all` |

The force triggers correspond to `zen_event` kinds (`summary_force`, `ninja_force`, `supersummary_force`) that anything can fire to request a run now.

## 18.2 Subscriptions, not scores

The Scheduler does not score triggers for relevance. Each KFC declares, in its own definition, the trigger IDs it cares about (`trigger_subscriptions`) and its `pipeline_tier`. When a trigger fires, the Scheduler finds the components subscribed to it and dispatches each one, in order of its `delay_seconds` (lower runs sooner).

A component whose domain needs an instant response to one specific sensor does not add a Scheduler trigger. It ships its own small trigger that fires `zen_event` with `kind: summary_force` for that component.

## 18.3 Shedding load

Summarizer runs call a model, and a burst of state changes (a door opening and closing repeatedly, a restart) could queue far more runs than the model can serve. The Scheduler sheds work based on queue depth and tier, using thresholds from the household cabinet's `zen_scheduler_config` drawer:

| Tier | Dispatched | Shed when |
|---|---|---|
| `super` | Always, with no delay | Never |
| `keeper` (the default; the summarizer calls it `direct`) | On every subscribed trigger | Queue depth reaches `shed_keeper_at` (default 8) |
| `ambient` | On slower triggers only | Any fast trigger (`quarter_hour`, `every_10_minutes`), or queue depth reaches `shed_ambient_at` (default 4) |

Shed work is not lost. The drain router watches queue depth and, once it stays below `drain_below` (default 3) for a settling period, dispatches the most stale shed-eligible component. A starvation guard dispatches any component whose Kata has passed its maximum age, regardless of queue depth. The result is that bursts slow the Monastery down without letting any component go stale indefinitely.

There is no per-component cooldown timer and no quiet-hours policy in the Scheduler. Queue-depth shedding is what prevents storms. The per-component emission cooldown (Chapter 10) governs escalation, not summarization.

## 18.4 The Dispatcher

The Dispatcher decouples callers from the scripts they call. A caller fires `zen_event` with `kind: dojotool_call`, a tool name, a correlation ID, and a payload. The Dispatcher routes it to the registered script and fires `kind: dojotool_return` with the same correlation ID. An unknown tool returns a structured error on the bus instead of faulting the caller. Every call must carry a correlation ID, or it is rejected before reaching the registry.

The Dispatcher also hosts `zen_dojotools_urgency_handler`, the catch-all for components that need a human's attention but have no automatable action (Chapter 10). It opens a task through Taskmaster, and it deduplicates: every task it opens is tagged per component, and a recurrence of an open condition adds a note to the existing task instead of opening a new one. A call that asserts action is needed while carrying nothing to act on (zero urgency, no attention text, no suggested action) opens nothing.

## 18.5 Support routers

Several small routers ride alongside the Dispatcher, each doing one job:

| Router | Does |
|---|---|
| `zen_identity_manifest_router` | Rebuilds the identity manifest when provisioning changes the roster |
| `zen_label_mutation_router` | Rebuilds the compact label index after label changes, and on a schedule |
| `zen_scheduler_drain_router` | Recovers shed components (18.3) |
| `zen_flynn_health_resummarize` | Refreshes the system component when Flynn's health changes |
| `zen_cabinet_vi_autorepair` | Repairs a cabinet header stored as a string instead of a mapping |

Daily, and on Home Assistant start, `zen_dojotools_manifest` also re-runs its bootstrap modes: KFC self-registration and Lens Bus provider registration (Chapter 9).

> **Not yet built.** The Abbot does not yet route work to different inference providers. Tagging each job so it can go to the right model with the right metadata (keeping sensitive content on a local model, sending other work elsewhere) is stated direction, with the Abbot as the single dispatch point for all inference.
