# Zen DojoTools SystemTools — 2026.6.0 'Clue'

*HA lifecycle management, log reading, event emission, and home mode — MCP-exposed*

---

## Overview

SystemTools is the ZenOS-AI system kernel — platform-level tools for keeping HA healthy and observable, plus the home mode engine that contextualizes everything else.

This package consolidates four script modules and the home mode subsystem:

| Component | MCP-Exposed | Purpose |
|---|---|---|
| `zen_dojotools_systemtools` | **Yes** | Config check, safe restart, reloads, and update management |
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

#### `ha_reload_all`

Reloads all YAML domains: automations, scripts, scenes, groups, helpers, timers, counters, schedules, YAML template sensors, MQTT YAML entities, zones, rest/command_line, custom Jinja templates, themes, and the `homeassistant:` core block.

**Config check runs first** — blocked on nogo. **Deferred via scheduler** — script fires `zen_event(kind: deferred_reload_all)` and exits; the Scheduler automation handles the actual reload from automation context. This avoids the `asyncio.InvalidStateError` that occurs when a script calls `homeassistant.reload_all` directly after a `response_variable` child call. Returns immediately with `result: success` and `config_check: passed` when queued.

**LAST RESORT ONLY** — use targeted reload modes (`ha_reload_scripts`, `ha_reload_automations`, etc.) whenever possible. `ha_reload_all` reloads every domain and should only be used when targeted reloads cannot resolve the issue.

---

#### `ha_reload_scripts`

Reloads scripts only. Config check runs first. Deferred via `zen_event(kind: deferred_script_reload)`. Returns immediately.

Use when automations are actively running and you do not want to interrupt them.

**Ships as a hard dependency pair with `dojotools_scheduler.yaml`.** The Scheduler must be loaded to handle the deferred events. Both files must be reloaded together.

---

#### `ha_update_skip` / `ha_update_clear_skipped`

Skip or un-skip a pending update. Both require `confirm_action: true` and `entity_id`.

---

#### `pnotif_list`

List all active HA persistent notifications. Returns `{count, notifications[]}` where each entry has `notification_id`, `title`, `message`, `created_at`.

> **Caveat:** reads `states.persistent_notification` — HA 2025.x+ may store some notifications outside the state machine. `count=0` does not guarantee the HA panel is clear; verify in UI if suspicious.

---

#### `pnotif_raise`

Create a new persistent notification. Fields: `notif_id`, `notif_title`, `notif_message`. All optional — HA generates an ID if omitted.

---

#### `pnotif_edit`

Update an existing persistent notification by re-raising it with the same `notif_id`. Fields: `notif_id`, `notif_title`, `notif_message`.

---

#### `pnotif_dismiss`

Dismiss a persistent notification by ID. Requires `notif_id`.

---

#### `repairs_list`

List all active HA repairs (issues). Returns `{count, issues[]}` where each entry has `entity_id`, `issue_id`, `domain`, `severity`, `is_fixable`, `ignored`, `title`.

Fixable issues can be resolved via `repair_fix`. ZenOS-generated issues have `domain: zenos`.

---

#### `repair_fix`

Fix a single HA repair issue. Delegates to `script.zen_dojotools_ectoplasm` with `action_type: repair_remove`. Requires `confirm: true`, `repair_domain`, and `issue_id`. Returns the ectoplasm result.

---

#### `notice_dashboard`

Aggregate system health view. Pulls ZenOS active alerts + persistent notifications + HA repairs in one call. Returns `action_queue[]` — a pre-built list of `{priority, item, tool, call}` entries ready to fire, ordered by urgency. Use this first for triage rather than calling `alertmanager`, `pnotif_list`, and `repairs_list` separately.

---

#### `render_dojo`

Returns a rendered copy of the Dojo cabinet contents in prompt-ready format — the same view Friday sees at inference time. Read-only. Useful for debugging what the AI has access to and verifying KFC content is loading correctly.

---

#### `render_system`

Returns a rendered copy of the System cabinet (Cortex, Directives, Purpose) in prompt-ready format. Read-only. Useful for verifying that `zen_admintools_prompt_loader` wrote the expected version of each prompt primitive.

---

#### `prompt_health`

Returns a prompt health report: token counts per major prompt section (Cortex, identity, active components, zen_summary, home_overview), total estimated context size, and a `budget_ok` flag. Use to diagnose context stuffiness before it degrades agent quality.

---

#### `pipeline`

Returns live pipeline status: which kill switches are active, the last Ninja and SuperSummary run timestamps, component count, and `zen_agent_health` sensor state. Single-call triage view without needing to read individual health sensors.

---

#### Versioned maintenance scripts

Versioned repair scripts live in the Admin/Recovery plane and are not part of the normal conversation-agent surface.

Use `script.zen_admintools_run_repair` with `confirm_action: true` for human-approved maintenance, or run the target `maint/` script directly from Developer Tools when following a release note.

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

Fires a `zen_event` on the EventBus and writes a structured entry to the system log via `system_log.write`. `severity: warn` is mapped to `level: warning` for the log write (core's `system_log.write` only accepts `warning`, not `warn`) — all other severities pass through unchanged.

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

---

## Version History

| Version | Change |
|---------|--------|
| v5.2.1 | `zen_health_report` now surfaces `sensor.zen_label_health` directly (`health_sensors.labels`/`reasons.labels`) instead of only inheriting it second-hand through `agent_health`. New `label_cabinet_consistency` block (`consistent`, `label_health_state`, `label_health_missing`, `cabinet_health_state`) — `zen_label_health` and `zen_cabinet_health` derive missing-required-label state from the same source and should always agree; a disagreement is flagged as a `SENSOR CONFLICT` (stale template render, not two real problems) with `ha_reload_templates` as the suggested first move. |
| v5.1.1 | `render_dojo`, `render_system`, `prompt_health` — prompt inspection tools. `pipeline` — pipeline status view. `tool` field resolves `(mode \| default...) or (tool \| default...)` alias for backward compat. Safer `int()` defaults throughout (`\| int(600)`, `\| int(300)`). |
| v5.1.0 | `ha_reload_all` / `ha_reload_scripts` deferred via `zen_event` kinds (`deferred_reload_all`, `deferred_script_reload`). `zen_health_report` reads 7 resolvers + 5 health sensors incl. `zen_agent_health`. HALMark FG-35/36/37 surfaced in help response. |
| v4.8.0 | `pnotif_list`, `pnotif_raise`, `pnotif_edit`, `pnotif_dismiss` — persistent notification CRUD. `repairs_list`, `repair_fix` — HA repairs surface. `notice_dashboard` — aggregate triage view with pre-built `action_queue[]`. |
| v4.5.9 | `ha_reload_all` and `ha_reload_scripts` deferred via `zen_event`. Closes asyncio `InvalidStateError` WONT FIX. All four reload modes now config-check gated. Requires Scheduler support for deferred reload events. |
