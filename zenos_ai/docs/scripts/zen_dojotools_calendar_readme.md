# Zen DojoTools Calendar — v1.3.1

*HA Calendar domain CRUD. Split from `dojotools_office.yaml` (2026-05-15).*

---

> **Response envelope (2026.10.0):** `zen_dojotools_calendar` returns the standard OS envelope — `{status, mode, tool, result, system_message, caller_token}` (see [`envelope()`](../custom_templates/zen_os1_jinja.md#envelopestatus-mode-result-tool-caller_token--canonical-response-shape)). The response fields documented on this page live under `result`; top-level `status` is the generic `success`/`error` execution status. Read `.result` when consuming a response via `response_variable`.

> **Backstage pass required (2026.11.0).** An agent without `zenos.agent.basic` gets this tool's `help`, `tool_manifest` and `kfc_manifest` only. Anything else answers `status: error`, `reason: "guarded: onboarding_required: ..."` and `next_step: {tool: zen_flynn_oobe, mode: run}`, and writes one `zen_entry_gate` warning to the system log. Calls no user started (boot, Flynn, automations, the scheduler) pass. See [the onboarding gate](../developer_standards.md#the-onboarding-gate).

## Overview

`zen_dojotools_calendar` is the canonical tool for all AI-driven calendar operations in ZenOS-AI. All calendar reads, creates, updates, and deletes must go through this tool — never call HA calendar services directly.

Supports HA-native calendars (Google, Local, ICS), Microsoft 365, and any provider the HA calendar integration supports. MS365 calendars use native modify/remove APIs. Other providers use delete + recreate for update (required because HA's `calendar.create_event` service does not support patching).

**Split from:** `dojotools_office.yaml`. The script alias and behavior are unchanged — only the file location changed.

---

## Actions

| Action | What it does |
|--------|-------------|
| `read` (default) | Read events from a calendar for a date range. Returns event list with summary, start, end, description, location, and event_id where available. |
| `inspect` | Advanced read using the system inspect tool. Shows `event_id` if the provider exposes it. Use this to locate event_id before update/delete. |
| `create` | Create a new event. Requires `calendar_name` and `summary`. Requires the `zenos.comms.pii_disclosure` certification (level 1) as of 2026-09-10 (#10390) — see the [Security Certification Manual](../getting_started/security_certification_manual.md). |
| `update` | Update an existing event by `event_id`. Requires `event_id`. Blocked if provider does not expose event_id. Also requires `zenos.comms.pii_disclosure` (level 1). |
| `delete` | Delete an event by `event_id`. Requires `event_id`. Blocked if provider does not expose event_id. Requires the `zenos.comms.calendar_control` certification (level 2) — irreversible event removal is its own risk axis, independent of `zenos.comms.pii_disclosure`; see the [Security Certification Manual](../getting_started/security_certification_manual.md). |
| `list` | List all available calendar entities (equivalent to `calendar_name: '*'`). |
| `help` | Return full action reference, field list, and provider notes. |

---

## Inputs

| Field | Required For | Description |
|-------|-------------|-------------|
| `action_type` | — | Operation. Default: `read`. |
| `calendar_name` | `read`, `create`, `update`, `delete` | Calendar entity_id, friendly name, or `*` / blank for all. |
| `start` | `read` | Start of date range. ISO string (`2026-06-01T09:00:00`) or date string (`2026-06-01`). Default: today 00:00 local. |
| `end` | `read` | End of date range. Default: tomorrow 00:00 local. |
| `summary` | `create`, `update` | Event title. |
| `description` | `create`, `update` | Event body text. |
| `location` | `create`, `update` | Event location string. |
| `event_id` | `update`, `delete` | Unique event identifier from the provider. Required — no summary matching. |
| `label_targets` | `read` | Comma-separated HA labels. Aggregates events across all calendar entities returned by `label_entities()`. |
| `caller_token` | — | Opaque audit token echoed in response. |

---

## Provider Notes

| Provider | event_id exposed | update/delete |
|----------|-----------------|---------------|
| Microsoft 365 | Yes | Native MS365 modify/remove APIs |
| Google Calendar | No | Safely blocked — use delete + create |
| HA Local Calendar | No | Safely blocked — use delete + create |
| ICS (read-only) | No | Not applicable |

If `event_id` is not returned by `read`, use `inspect` to look for it. If the provider does not expose it, update and delete are always blocked with a clear error.

---

## Examples

**Read today's events from a calendar:**

```yaml
zen_dojotools_calendar:
  action_type: read
  calendar_name: "Family"
```

**Read events from all calendars tagged with a label:**

```yaml
zen_dojotools_calendar:
  action_type: read
  label_targets: "shared_calendar"
  start: "2026-06-01"
  end: "2026-06-08"
```

**Create an event:**

```yaml
zen_dojotools_calendar:
  action_type: create
  calendar_name: "calendar.family"
  summary: "Spa maintenance"
  start: "2026-06-15T10:00:00"
  end: "2026-06-15T11:00:00"
  description: "Monthly water chemistry check"
```

**Inspect to find event_id:**

```yaml
zen_dojotools_calendar:
  action_type: inspect
  calendar_name: "calendar.family"
```

**Delete by event_id (MS365):**

```yaml
zen_dojotools_calendar:
  action_type: delete
  calendar_name: "calendar.ms365_family"
  event_id: "AAMkAGFhZWM..."
```

---

## Notes

- Pass `calendar_name: '*'` or leave blank to list all calendars.
- `update` and `delete` operate only by `event_id`. No summary or date matching.
- If multiple calendars match a `label_targets` query and the action is `create`, `update`, or `delete`, the call is blocked with an error — specify a single calendar for write operations.
- Timestamp normalization: providers return timestamps in varying formats (UTC or local offset). Normalize to local time when displaying to users.

---

## Version History

| Version | Change |
|---------|--------|
| 2026.10.0 | `delete` gated on the new `calendar_control` certification (level 2). Response wrapped in the standard OS envelope. The update path's internal delete step unwraps its own enveloped response. |
| v1.11.0 | Split from `dojotools_office.yaml`. No behavior changes — file relocation only. |
