# ZenOS-AI DojoTools — SystemTools Package (v1.3.0)

This package is the ZenOS-AI “system kernel” for Home Assistant: a single, self-contained YAML that provides guarded system hygiene tools for conversation agents (LLMs) without exposing raw Home Assistant internals.

It consolidates prior system-layer DojoTools into one file and enforces a strict security model:
- localhost-only API access
- token never passed at call-time
- whitelisted reads
- confirmation gates on writes
- no `/states`, no templates, no cabinets, no room control

---

## What’s in the box

This file contains three exposed MCP scripts (and one internal primitive), plus supporting shell/rest plumbing:

### Exposed to your conversation agent (MCP)
- `script.zen_dojotools_systemtools`
- `script.zen_dojotools_ha_log_viewer`
- `script.zen_dojotools_event_emitter`

### Internal only (do NOT expose)
- `script.zen_dojotools_ha_api`  
  A low-level API broker used by `systemtools`. Keep it private.

### Supporting primitives
- `shell_command.zen_log_*` helpers for safe log reads
- `rest_command.ha_api_get` / `rest_command.ha_api_post` pointing at `127.0.0.1:8123`

---

## Install

1) Add your long-lived token to `secrets.yaml`

```yaml
ha_bearer: "Bearer <your-long-lived-token>"
````

Generate it at: Profile → Security (e.g. `http://homeassistant.local:8123/profile/security`)

2. Drop the YAML file into your packages directory
   Example location:

```
config/packages/zenos_ai/dojotools/dojotools_systemtools.yaml
```

3. Ensure packages are enabled (if not already)

```yaml
homeassistant:
  packages: !include_dir_named packages
```

4. Restart Home Assistant

5. Expose the MCP scripts to your conversation agent
   Settings → Voice Assistants → Expose:

* ✅ `zen_dojotools_systemtools`
* ✅ `zen_dojotools_ha_log_viewer`
* ✅ `zen_dojotools_event_emitter`
* ❌ DO NOT expose `zen_dojotools_ha_api`

---

## Security model

This package is intentionally boring and strict.

* API endpoint is hardcoded to `http://127.0.0.1:8123`
* Auth token is read from `secrets.yaml` only
* Whitelisted `GET` paths only; arbitrary `/api/...` reads are blocked
* Write operations require confirmation (`confirm_action: true`)
* No access to:

  * `/api/states`
  * `/api/template`
  * cabinet sensors
  * room control / household automations

If you want more capability, build it in a different package (drivers), not in this kernel.

---

## HA 2025.11+ Journal Mode Guard

Home Assistant 2025.11 removed the persistent `home-assistant.log`. This package includes a semantic guard inside `zen_dojotools_ha_log_viewer`:

* It runs `shell_command.zen_log_check`
* If the file is missing, it checks the installed HA version
* On modern HA, it returns a structured `journal_mode` response (not an error) with guidance and the Petro31 file-logger restoration link:

  * `https://github.com/Petro31/a-file-logger`

This prevents agent hallucination when the log file is “missing by design.”

---

## Tools

### 1) `zen_dojotools_systemtools` (MCP)

System lifecycle and update operations, all guarded.

Supported `tool` values:

* `ha_config_check`
  Runs HA config validation. Returns `go` / `nogo`.
* `ha_restart` (requires `confirm_action: true`)
  Always runs config check first; blocks restart if check fails.
* `ha_update_install` (requires `confirm_action: true` + `entity_id`)
  Initiates update install. Optional `do_backup`.
* `ha_update_skip` (requires `confirm_action: true` + `entity_id`)
  Skips an update entity.
* `ha_update_clear_skipped` (requires `confirm_action: true` + `entity_id`)
  Clears skipped flag.

Example: config check

```yaml
service: script.zen_dojotools_systemtools
data:
  tool: ha_config_check
```

Example: restart (explicitly confirmed)

```yaml
service: script.zen_dojotools_systemtools
data:
  tool: ha_restart
  confirm_action: true
```

Example: install an update

```yaml
service: script.zen_dojotools_systemtools
data:
  tool: ha_update_install
  entity_id: update.home_assistant_core_update
  do_backup: false
  confirm_action: true
```

---

### 2) `zen_dojotools_ha_log_viewer` (MCP)

Safe, read-only log access with truncation and journal guard.

Modes:

* `tail` (uses `lines`)
* `search` (requires `pattern`)
* `full`
* `summary` (uses `lines`)
* `page` (uses `start` + `count`)

Example: tail last 200 lines

```yaml
service: script.zen_dojotools_ha_log_viewer
data:
  mode: tail
  lines: 200
```

Example: grep for errors

```yaml
service: script.zen_dojotools_ha_log_viewer
data:
  mode: search
  pattern: "ERROR"
```

Example: page window (start line + count)

```yaml
service: script.zen_dojotools_ha_log_viewer
data:
  mode: page
  start: 500
  count: 200
```

Output:

* Returns a structured object including:

  * `status`, `mode`, `truncated`, `line_count`, `message`, `result`
* If `home-assistant.log` is missing on HA 2025.11+, returns:

  * `status: journal_mode`
  * `expected_behavior: true`
  * guidance for journal-based logs

---

### 3) `zen_dojotools_event_emitter` (MCP)

Emits small, structured ZenOS-AI events for traceability and observability.

Modes:

* `emit`
* `help` (returns canonical event shapes)

Example: emit a breadcrumb event

```yaml
service: script.zen_dojotools_event_emitter
data:
  mode: emit
  component: friday
  severity: info
  kind: heartbeat
  summary: "Friday online and monitoring system state."
  metadata:
    build: "zenos_ai"
    node: "homeassistant"
```

EventBus:

* Emits event type: `zen_event`
* Also logs a compact line via `system_log.write`

---

### 4) `zen_dojotools_ha_api` (INTERNAL)

Low-level broker used by `systemtools`. Do not expose to an agent.

Actions:

* `help`
* `check_config`
* `error_log`
* `supervisor_logs`
* `events`
* `services`
* `config`
* `health`
* `supervisor`
* `get` (whitelisted only)

Whitelisted `get` paths:

* `error_log`
* `events`
* `services`
* `config`
* `system_health/info`
* `hassio/info`
* `hassio/logs`

---

## Known behaviors and limits

* Log output is truncated:

  * shell commands cap at ~200KB
  * script logic truncates at ~240KB
* `ha_api_post` config check parsing:

  * if HA returns non-JSON, `check_response` becomes `{}` (safe fallback)
  * if a `nogo` happens and details are missing, check core logs

---

## SystemTools Charter (why this stays “small”)

This package is the ZenOS-AI system kernel. Any additions must be:

1. PLATFORM-LEVEL only
   System hygiene, safety, observability, lifecycle. Not household automation.

2. PRIMITIVE
   Small, deterministic, single-purpose. Not convenience wrappers.

3. GUARDED
   Every write op requires confirmation or a whitelist. No raw passthrough.

Allowed:

* config validation, restart gates, log/health reads, API broker, updates/backups

Not allowed:

* `/states` access, cabinet sensors, room control, arbitrary service writes

---

## UAT / Verification

UAT PASS — 2026-03-03
Auditor: Caitlin (Anthropic Sonnet 4.6) + Veronica (OAI editorial)
HALMark: 0.9.10-draft — PASS Level 3 (Steward Safe) — 23/25 PASS, 2/25 N/A
UAT Cases: 20/20 PASS

Live verified:

* `ha_config_check → {result: go}`
* `ha_restart` clean → `{config_check: passed, result: success}`
* `ha_restart` canary YAML → restart blocked
* log viewer `tail` → success
* log viewer `journal_mode` (log renamed) → expected_behavior true

---

## License / Attribution

ZenOS-AI DojoTools — SystemTools Package
Authored and validated in the ZenOS-AI / Project Friday ecosystem.

If you fork this, keep the charter intact and treat expansions as a security-sensitive change.
