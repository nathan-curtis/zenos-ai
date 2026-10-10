# Alert Manager — KFC Guide

> **KF4 1.5.0** | ZenOS-AI 2026.5.0 "Fry's Grandpa" | Alertmanager v1.2.0

Alert Manager is ZenOS-AI's built-in alert component. It monitors entities tagged with the **`Alert Manager`** label, interprets their states, and dispatches notifications when something needs attention.

It ships disabled. Enable it when you're ready to start monitoring.

---

## Enabling alert_manager

`alert_manager` ships with `meta.enabled: false`. To enable:

- Ask your AI: *"Enable alert manager"*
- Or set `meta.enabled: true` in the `alert_manager` Dojo drawer via `zen_dojotools_scribe`

New to this? See **[Your First Alert](../getting_started/first_alert.md)** for a full zero-to-notification walkthrough.

---

## What to Wire vs Skip

Not every alert needs instant dispatch. Match trigger timing to what you're actually monitoring:

| Type | Examples | When to dispatch |
|------|----------|-----------------|
| Life safety | Smoke, CO, gas, flood sensor | Immediately — wire a per-KFC trigger file |
| Security | Door/window open, motion | Immediately or on mode change — depends on context |
| Equipment | Appliance fault, pump off | Immediately for failure states |
| Chemistry / maintenance | Pool/spa chemistry, filter | Hourly sweep is fine — no trigger file needed |
| Informational | Energy, temperature drift | Scheduled sweep — no trigger file needed |

The Scheduler's built-in `hourly_trigger` handles most passive monitoring. Reserve per-KFC trigger files for things that matter in the moment.

---

## Wiring Instant Dispatch

For entities that need immediate alert on state change, create a dedicated trigger file in your installer's custom packages directory:

```yaml
# packages/your_family/kfc_trigger_alert_manager.yaml
# ⚠️ PERSONAL FILE — DO NOT COMMIT TO REPO

automation:
  - id: 'YOUR_UNIQUE_ID'
    alias: KFC Trigger — Alert Manager
    description: >-
      Fires a targeted force_summary for alert_manager on meaningful state changes.
    triggers:
      - trigger: state
        entity_id:
          - binary_sensor.your_smoke_detector   # REPLACE
          - binary_sensor.your_co_detector      # REPLACE
        to: 'on'
        not_from: [unknown, unavailable]
        for:
          seconds: 5

    actions:
      - event: zen_event
        event_data:
          event:
            kind: summary_force
            component: alert_manager

    mode: queued
    max: 3
```

This file fires `zen_event` with `kind: summary_force`, which the core Scheduler handles as a targeted dispatch for `alert_manager`. No Scheduler changes needed.

---

## Trigger Direction — alert_when_* Labels

The AI needs to know which direction is bad for each entity. Apply one of the `alert_when_*` modifier labels alongside `alert_manager`. The label name encodes the alert condition — no additional configuration needed.

| Label | Nominal | Alert when | Use for |
|-------|---------|------------|---------|
| `alert_when_on` | off | state is `on`, `problem`, or `detected` | Leak sensors, smoke detectors, fault flags |
| `alert_when_off` | on | state is `off` or `unavailable` | Pumps, valves, line power, services that must stay running |
| `alert_when_not_off` | off | any non-`off` state (`on`, `open`, `triggered`, etc.) | Multi-state sensors where any non-idle reading is a problem |
| `alert_when_under_N` | ≥ N | numeric state drops below N (extract N from slug) | Salt level, battery %, filter life, days remaining |

Each label carries a description that the Ninja Summarizer reads as authoritative context — you don't need to re-explain the condition in `component_summary`. The label IS the rule.

Without a modifier label, an entity with `alert_manager` is treated as informational context — the AI reads it but won't flag it as a deviation unless the state is obviously wrong.

### ZQ query shortcut

To find all entities with any `alert_when_*` modifier label across all domains:

```python
{"label": "alert_manager", "label_regex": "^alert_when_"}
```

This returns every alert_manager entity that carries at least one modifier label — useful for auditing coverage.

---

## Debounce by Condition Type

| Condition | Recommended debounce |
|-----------|---------------------|
| Smoke / CO | 5 seconds or less |
| Flood / leak | 5–15 seconds |
| Door / window | 30–60 seconds |
| Motion | 60+ seconds, or skip instant dispatch entirely |
| Equipment fault | 30–60 seconds |
| Chemistry / maintenance | None — hourly sweep only |

Set `for: seconds: N` in your trigger block. Prevents false dispatches from transient sensor noise.

---

## Setting Up Notifications

`alert_manager` v1.2.0 dispatches via `zen_dojotools_postman`. Set `notify_target: postman` in your alert_manager Dojo drawer to route notifications through the postman pipeline (respects sleep gate, urgency tiers, away policy, and image attachment).

The legacy `zen_dojotools_notification_router` path is still supported but deprecated. Migrate to postman.

For a full postman setup walkthrough: **[Your First Alert](../getting_started/first_alert.md)**.

---

## Priority Inject — AI Situational Awareness (v1.2.0)

Error-severity alerts now wire into the **priority inject** system automatically. When alertmanager raises an error-severity alert, it fires `zen_event kind: priority_inject_write`, which:

1. Lands in the `_zen_priority_inject` drawer in the household cabinet (up to 5 slots, sorted by urgency)
2. Appears in the **NOTIFICATIONS block** at the top of every AI prompt context frame

The AI enters every conversation already knowing what's wrong — without being told. When the alert clears, `zen_event kind: priority_inject_clear` removes it from the inject slot automatically.

### Checking Inject State

There is no priority sensor; it was removed in Alert Manager 5.4.3. To check whether the AI is carrying active alert context, read `priority_context_state` (`active` or `clear`) from `zen_dojotools_alertmanager mode=list`, or the `all_quiet`, `priority_count`, and `highest_urgency` attributes on the Home Overview helper sensor (`sensor_helpers.yaml`). Room Manager's `home_overview` carries the full rollup, including providers and the oldest active entry. See [AlertManager](../components/alertmanager.md#reading-priority-state).

### Urgency Mapping

Alertmanager maps alert severity to inject urgency:

| Alert Severity | Inject Urgency | Expires |
|---|---|---|
| `error` | `urgent` | 60 minutes |
| Life-safety (smoke, CO, flood) | `critical` | 60 minutes |

Entries are automatically GC'd by Core on every cycle. Stale entries (past expiry) are evicted; invalid entries (bad urgency value) are dropped on write.

---

## The component_summary Field

The `component_summary` in the Dojo drawer describes the overall scope of what this component monitors. Keep it high-level — you don't need to re-explain each entity's alert condition here. The `alert_when_*` labels carry that context directly in their descriptions, and the Ninja Summarizer reads them per-entity.

Focus `component_summary` on:
- What subsystems are covered
- Severity tier groupings
- Any cross-entity rules or escalation context

Good example:

```
Monitors household safety and infrastructure sensors across all subsystems.
Severity tiers: critical (smoke/CO/leak/alarm), urgent (water valve, line power),
warning (filter, salt, maintenance), information (grid state, appliance status).
Surfaces active deviations; escalates via notification_router at urgency >= 5.
```

Per-entity semantics live in the `alert_when_*` labels — not here.

---

## Adding More Entities

Tag any entity with the **`Alert Manager`** HA label. It's automatically included on the next Summarizer run. No drawer edit needed.

---

*ZenOS-AI KF4 1.5.0 — Alertmanager v1.2.0 — 2026-05-03*
*Source: Nyx (live system observation), Cayt (dev)*
