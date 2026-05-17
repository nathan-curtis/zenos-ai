# ZenOS-AI AlertManager

**Version:** 1.2.0
**File:** `dojotools/dojotools_alertmanager.yaml`

**Entities:**
- `automation.zen_alert_manager` — fire/clear handler
- `automation.zen_priority_inject_handler` — priority slot write/clear handler
- `sensor.zen_priority_context` — live priority status sensor

---

## Overview

AlertManager is a fire-once alert deduplication and routing system. It prevents notification fatigue by suppressing repeat notifications for the same condition until it clears and recurs, routes alerts by severity to the appropriate channel, and surfaces error-level alerts into the Room Manager `home_overview` priority context.

Key capabilities:

* Fire-once dedup — same `alert_key` never fires twice while active
* Severity tiers: `info` / `warn` / `error` → mapped to urgency and routing
* Auto-escalation of `error` alerts to Room Manager priority inject slot
* Flexible dispatch: persistent HA notification, Postman, or any `notify.*` service
* No setup required — self-bootstrapping on first event

---

## How to Fire an Alert

Emit a `zen_event` HA event:

```yaml
event: zen_event
event_data:
  event:
    kind: alert_fire
    alert_key: unique_slug        # required — used as dedup key and cabinet key
    message: "Human-readable description"   # optional, defaults to alert_key
    severity: warn                # optional — info | warn | error, defaults to warn
    notify_target: persistent     # optional — persistent | postman | notify.<service>
```

### Postman-specific fields

```yaml
    notify_target: postman
    channel_hint: push            # push | tts | teams
    title: "Override Title"       # defaults to severity label (Warning / Error / Notice)
    image_entity: camera.front    # attaches snapshot
```

---

## How to Clear an Alert

```yaml
event: zen_event
event_data:
  event:
    kind: alert_clear
    alert_key: unique_slug        # must match the fired alert_key
```

Clearing removes the entry from `_zen_active_alerts`, dismisses the persistent notification if one was created, and removes the alert from the priority inject slot.

---

## Severity Levels

| Severity | Postman Urgency | Priority Inject | Default Title |
|----------|----------------|-----------------|---------------|
| `info` | 3 (low) | No | "Notice" |
| `warn` | 5 (medium) | No | "Warning" |
| `error` | 8 (critical) | Yes → `urgency: critical` | "Error" |

Only `error`-severity alerts are auto-wired to the Room Manager priority inject slot. `warn` and `info` appear in `active_alerts[]` in `home_overview` but do not set `priority_context=active`.

---

## Notify Targets

| Value | Behavior |
|-------|---------|
| `persistent` (default) | Creates HA persistent notification |
| `postman` | Routes via `zen_dojotools_postman` with authority-stack routing. Household/family `postman_profile` policy applied. |
| `notify.<service>` | Any registered HA notify service (e.g., `notify.pushover`) |

---

## Dedup Mechanism

State stored in household cabinet drawer `_zen_active_alerts`:

```json
{
  "unique_slug": {
    "fired_at": "2026-05-16T12:00:00",
    "message": "Description",
    "severity": "warn"
  }
}
```

**Fire:** If `alert_key` already present in drawer → silent no-op. If absent → send notification + write entry.

**Clear:** If `alert_key` present → remove entry + dismiss notification + remove from priority slot. If absent → silent no-op.

This means the same condition can re-alert after it clears. The dedup window is "while the alert is active" — not time-based.

---

## Priority Inject Slot

`error`-severity alerts write to household cabinet drawer `_zen_priority_inject`:

```json
{
  "alert_unique_slug": {
    "summary": "message text",
    "urgency": "critical",
    "expires": "ISO timestamp (60 min from fire)",
    "since": "ISO timestamp",
    "entities": ["up to 3 related entity IDs"]
  }
}
```

Clearing an `error` alert removes it from this drawer.

### `sensor.zen_priority_context`

Live rollup sensor reading `_zen_priority_inject`:

| Attribute | Description |
|-----------|-------------|
| state | `active` or `clear` |
| `count` | Number of non-expired priority entries |
| `providers` | List of active provider IDs |
| `highest_urgency` | `critical` | `urgent` | `""` |
| `oldest_since` | ISO timestamp of oldest active entry |

This sensor is the integration point for Room Manager `home_overview`. Its state feeds `signal.all_quiet` and `alerts.priority_context`.

---

## Room Manager Integration

`home_overview` reads AlertManager state and includes:

```json
{
  "alerts": {
    "priority_context": "active|clear",
    "priority_count": 2,
    "highest_urgency": "critical",
    "priority_providers": ["alert_key1"],
    "oldest_since": "2026-05-16T10:00:00",
    "active_alerts": [
      { "key": "alert_key", "message": "...", "severity": "warn", "fired_at": "..." }
    ],
    "active_alert_count": 3
  },
  "signal": {
    "attention": [...],      // fresh alerts < 4h old
    "stale_alerts": [...],   // alerts >= 4h old
    "all_quiet": false       // true only when no fresh alerts AND priority_context=clear
  }
}
```

---

## Cabinet Drawers

| Drawer | Format | Auto-Expire |
|--------|--------|------------|
| `_zen_active_alerts` | `{alert_key: {fired_at, message, severity}}` | Never (protected) |
| `_zen_priority_inject` | `{provider_id: {summary, urgency, expires, since, entities}}` | Never (protected) |

Both drawers are created automatically on first write. The `_` prefix marks them hidden and protected — FileCabinet GC never collects them.

---

## Setup Requirements

None. AlertManager is self-bootstrapping.

**Dependencies (must already exist):**
- `sensor.zen_default_household_cabinet_resolved` — household cabinet entity
- `script.zen_dojotools_filecabinet` — drawer read/write
- `script.zen_dojotools_postman` — required only if using `notify_target: postman`

No HA labels, helpers, or input entities required.

---

## Emitting Alerts from Components

Any KFC component can emit alerts. Pattern:

```yaml
- event: zen_event
  event_data:
    event:
      kind: alert_fire
      alert_key: my_component_condition_name
      message: "Describe what went wrong"
      severity: error
      notify_target: persistent
```

And when the condition resolves:

```yaml
- event: zen_event
  event_data:
    event:
      kind: alert_clear
      alert_key: my_component_condition_name
```

Keep `alert_key` stable and unique across all components — it is the dedup key and appears in `home_overview` output.
