# Zen DojoTools SystemTools — 4.2.0

*HA lifecycle management, log reading, event emission, and home mode*

---

## Overview

SystemTools is the ZenOS-AI system kernel — platform-level tools for keeping HA healthy and observable, plus the home mode engine that contextualizes everything else.

This package consolidates four script modules and the home mode subsystem:

| Component | MCP-Exposed | Purpose |
|---|---|---|
| `zen_dojotools_systemtools` | **Yes** | Config check, safe restart, update management |
| `zen_dojotools_ha_log_viewer` | **Yes** | Log reading in five modes + HA 2025.11+ journal detection |
| `zen_dojotools_event_emitter` | **Yes** | Structured `zen_event` emission to the EventBus |
| `zen_dojotools_ha_api` | **No** | Internal HA API wrapper (do not expose) |
| Home Mode | N/A | Scheduled time-based mode transitions + presence detection |

---

## Setup Requirement

SystemTools requires a long-lived HA token in `secrets.yaml`:

```yaml
ha_bearer: "Bearer <your-long-lived-token>"
```

Generate at: **Settings → Profile → Security → Long-Lived Access Tokens**

This token is read once at script execution — it is never passed in call signatures or exposed to the agent.

---

## zen_dojotools_systemtools

Config validation, safe HA restart, and update management. Every destructive operation requires explicit confirmation.

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `tool` | select | — | Which tool to run (see below) |
| `confirm_action` | boolean | `false` | Required `true` for destructive operations |
| `entity_id` | text | — | Required for update actions — the `update.*` entity |
| `do_backup` | boolean | `false` | Create a backup before installing an update |

### Tools

#### `ha_config_check`

Validates your HA configuration without restarting. No confirmation required.

```json
{
  "tool": "ha_config_check",
  "result": "go",
  "valid": true,
  "errors": null,
  "message": "Config check passed."
}
```

Returns `"nogo"` with an errors array if validation fails.

---

#### `ha_restart`

Restarts Home Assistant. Requires `confirm_action: true`.

**Always runs a config check first.** If the config check fails, the restart is blocked:

```json
{
  "tool": "ha_restart",
  "result": "nogo",
  "errors": ["..."],
  "message": "Restart BLOCKED — config check failed. Fix errors before restarting."
}
```

If config is clean, restart proceeds:

```json
{
  "tool": "ha_restart",
  "result": "success",
  "config_check": "passed",
  "message": "Restart initiated."
}
```

---

#### `ha_update_install`

Installs an available update. Requires `confirm_action: true` and a valid `entity_id`.

```yaml
tool: ha_update_install
entity_id: update.home_assistant_core_update
do_backup: true
confirm_action: true
```

---

#### `ha_update_skip` / `ha_update_clear_skipped`

Skip or un-skip a pending update. Both require `confirm_action: true` and `entity_id`.

---

## zen_dojotools_ha_log_viewer

Log reader with five access modes. Automatically detects HA 2025.11+ journal mode (no log file) and returns guidance rather than an error.

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `mode` | select | `tail` | `tail`, `search`, `full`, `summary`, `page` |
| `pattern` | text | — | Regex pattern (required for `search`) |
| `lines` | number | `5000` | Lines to return (`tail` and `summary`) |
| `start` | number | `1` | Starting line (`page` mode) |
| `count` | number | `500` | Window size (`page` mode) |

### Modes

| Mode | What It Returns |
|---|---|
| `tail` | Last N lines — recent activity |
| `search` | Lines matching a regex pattern |
| `full` | Entire log (auto-truncated at 240KB) |
| `summary` | First N lines from the top |
| `page` | A window of `count` lines starting at `start` |

### HA 2025.11+ Journal Mode

From HA 2025.11, `home-assistant.log` no longer exists — logs went to the system journal. If the log file is missing, the viewer detects your HA version and returns a structured `journal_mode` response instead of an error:

```json
{
  "status": "journal_mode",
  "log_file_present": false,
  "expected_behavior": true,
  "root_cause": "core_logging_changed_to_journal",
  "user_guidance": {
    "recommended_actions": [
      "Use Settings → System → Logs",
      "Install HACS file-logger to restore persistent log"
    ]
  }
}
```

### Response Format

```json
{
  "status": "success",
  "mode": "tail",
  "lines_requested": 500,
  "truncated": false,
  "line_count": 312,
  "result": "..."
}
```

---

## zen_dojotools_event_emitter

Emits structured `zen_event` events onto the HA EventBus. Used for observability, breadcrumbs, and trace reconstruction. Does not alter system state.

See the full specification: **[Event Emitter Readme](zen_dojotools_event_emitter_readme.md)**

### Quick Reference

```yaml
mode: emit
component: friday
severity: info
kind: summary_force
summary: Forcing immediate summary of hot_tub_manager
```

Fires a `zen_event` on the EventBus and writes a structured entry to the system log.

**Common `kind` values:**

| Kind | Usage |
|---|---|
| `summary` | Standard summary emission |
| `summary_force` | Trigger immediate KFC summary (see Scheduler docs) |
| `kata_emit` | Kata state change |
| `heartbeat` | Periodic component health signal |

---

## zen_dojotools_ha_api

Internal HA API wrapper. **Do not expose to the conversation agent.**

Used internally by `zen_dojotools_systemtools` (config check before restart). Provides whitelisted GET access and POST to a fixed set of HA API paths:

```
error_log, events, services, config,
system_health/info, hassio/info, hassio/logs
```

All API calls go to `127.0.0.1:8123` (localhost only). Auth token is read from `secrets.yaml` — never passed at call time.

---

## Home Mode

Home mode is the system's contextual heartbeat. Friday reads `sensor.zen_home_mode` to understand what phase of the day it is, and the Scheduler uses home mode changes as a trigger for summarization.

### The Eight Modes

| Mode | Typical Time | Meaning |
|---|---|---|
| `Night-Late` | 23:00–06:00 | Sleep / late night |
| `Home-Wake` | 06:00–08:00 | Early wake |
| `Home-Morning` | 08:00–10:00 | Morning routine |
| `Home` | 10:00–17:00 | Normal daytime |
| `Home-Evening` | 17:00–21:00 | Evening |
| `Night` | 21:00–23:00 | Winding down |
| `Away` | Anytime | All occupants absent — **hold state** |
| `Paused` | Anytime | Manual freeze — **hold state** |

`Away` and `Paused` are **hold states** — they block automatic schedule transitions. The system won't advance to the next scheduled mode until the hold is released.

### How Transitions Work

Four automations manage the mode lifecycle:

| Automation | Trigger | What It Does |
|---|---|---|
| `zen_hm_init` | HA start | Evaluates current time window, sets correct mode (skips Away/Paused) |
| `zen_hm_schedule` | Each schedule anchor fires | Evaluates time window, advances mode (skips Away/Paused) |
| `zen_hm_away` | `zone.home` drops below 1 | Sets mode to `Away` |
| `zen_hm_presence_return` | `zone.home` rises above 0 | Releases Away hold, evaluates window |

The time-window evaluator (`script.zen_hm_evaluate_window`) is the single source of truth for the anchor → mode mapping. All automations delegate to it.

### Schedule Anchors

All six time boundaries are configurable via `input_datetime` helpers (Settings → Helpers):

| Helper | Default | Marks Start Of |
|---|---|---|
| `zen_am_start` | 06:00 | Home-Wake |
| `zen_morning_start` | 08:00 | Home-Morning |
| `zen_daytime_start` | 10:00 | Home |
| `zen_evening_start` | 17:00 | Home-Evening |
| `zen_night_start` | 21:00 | Night |
| `zen_late_night_start` | 23:00 | Night-Late |

Change an anchor and the schedule adjusts immediately — no automation edits needed.

### Additional Helpers

| Helper | Purpose |
|---|---|
| `binary_sensor.zen_quiet_hours` | True when inside quiet hours window (midnight-wrap aware) |
| `binary_sensor.zen_work_hours` | True during configured work hours |
| `input_boolean.zen_guest_mode` | Guest presence flag (set manually or by AI) |
| `input_boolean.zen_entertaining` | Hosting/entertaining flag |

`zen_quiet_hours` correctly handles windows that cross midnight (default: 23:00–06:00).

### Paused Mode

Setting mode to `Paused` freezes the schedule. Useful when you want Friday to stop making context-driven changes (during a party, maintenance, etc.). Release it manually by setting the mode back to any other non-hold state.

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `secrets.yaml` → `ha_bearer` | HA API auth token |
| `rest_command.ha_api_get` / `ha_api_post` | HA REST API calls |
| `shell_command.zen_log_*` | Log file access |
| `zone.home` | Presence detection for Away mode |
| `input_datetime.zen_*` | Schedule anchors and time windows |
| `update.*` entities | Update install/skip targets |
