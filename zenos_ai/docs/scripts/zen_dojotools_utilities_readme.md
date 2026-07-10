# Zen DojoTools Utilities — v5.1.0 (zen_dojotools_help v3.0.0)

*Calculator, dice, announcements, music search, system help, wait, cabinet audit, and canonical HA domain control tools*

---

## Overview

Utilities is a collection of general-purpose tools that don't belong to a specific subsystem. It covers math, randomness, TTS, music lookup, system introspection, cabinet auditing, and canonical GET+SET wrappers for HA domains that HA's own MCP built-ins handle inconsistently.

The canonical domain tools (`select_control`, `number`, `text`, `climate`, `water_heater`, `datetime`, `zones`) were promoted from legacy intent scripts in 2026.4. They are preferred over HA built-ins for their domains: they resolve shorthand names, validate inputs against entity constraints, and return structured responses.

---

## Script Index

| Script | Purpose |
|---|---|
| `zen_dojotools_calculator` | Math operations and GUID generation |
| `zen_dojotools_dice_roller` | D&D dice, coin flip, random numbers |
| `zen_dojotools_announce` | TTS announcement router with urgency + dedup gates |
| `zen_sutra_music_search` | Music Assistant search — **internal sutra, not MCP-exposed**. Use `zen_dojotools_media_manager` mode=search or mode=stacks_by_anchor. |
| `zen_dojotools_help` | Global help front door — system overview, help-proxy to any tool, tool-directory auto-discovery |
| `zen_dojotools_wait` | Timed delay (1–120 seconds) |
| `dojotools_volume_auditor` | Cabinet volume accessibility scanner |
| `zen_dojotools_notification_router` | **Deprecated** — legacy push notification router |
| `zen_dojotools_select_control` | CANONICAL: `select` / `input_select` GET+SET |
| `zen_dojotools_number` | CANONICAL: `number` / `input_number` GET+SET+arithmetic |
| `zen_dojotools_text` | CANONICAL: `text` / `input_text` GET+SET+string ops |
| `zen_dojotools_climate` | CANONICAL: `climate` GET+SET with topology context |
| `zen_dojotools_water_heater` | CANONICAL: `water_heater` GET+SET |
| `zen_dojotools_datetime` | CANONICAL: `input_datetime` GET+SET |
| `zen_dojotools_zones` | CANONICAL: zone CRUD + haversine bearing |

---

## zen_dojotools_calculator

Math operations and GUID generation.

### Input Fields

| Field | Type | Description |
|---|---|---|
| `number_1` | number | First operand |
| `number_2` | number | Second operand (where applicable) |
| `operator` | select | Operation to perform |

### Operators

**Two-operand:** `add`, `subtract`, `multiply`, `divide`, `exponent`, `modulus`, `percent_of`, `increase_by_percent`, `decrease_by_percent`, `round`

**One-operand:** `sqrt`, `cbrt`, `sin`, `cos`, `tan`, `degrees_to_radians`, `radians_to_degrees`, `log`, `ln`, `floor`, `ceil`, `fahrenheit_to_celsius`, `celsius_to_fahrenheit`

**Zero-operand:** `guidgen` — generates a UUID (8-4-4-4-12 hex format)

### Response

```json
{ "number_1": 9, "number_2": null, "operator": "sqrt", "result": 3.0 }
```

Errors return a string in `result`: `"Error: Division by zero"`, `"Error: Square root of negative"`, etc.

---

## zen_dojotools_dice_roller

Dice rolls, coin flips, and random numbers for D&D and anything else that needs randomness.

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `mode` | select | — | `Dice`, `Random Integer`, `Random Float`, `Coin Flip` |
| `dice_notation` | text | `d6` | XdY format (e.g., `3d6`, `d20`, `2d10`) |
| `min` | number | `1` | Range minimum (random modes) |
| `max` | number | `6` | Range maximum (random modes) |

### Response

```json
{ "mode": "dice", "dice": "3d6", "individual_rolls": [4, 2, 6], "total": 12 }
{ "mode": "coin_flip", "result": "heads" }
{ "mode": "random_integer", "min": 1, "max": 100, "value": 47 }
```

---

## zen_dojotools_announce

TTS announcement router to HA areas. Enforces four gates before firing audio — all gate responses are machine-readable status codes.

### Input Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `area` | area (multi) | Yes | One or more HA areas to announce to |
| `message` | text | Yes | Message to speak (max 255 characters) |
| `urgency` | number (1–10) | **Yes** | Urgency level — determines routing and gate behavior |

### Urgency Routing

| Urgency | Behavior |
|---|---|
| Omitted | Blocked — `status: urgency_required` |
| 1–3 | Blocked — `status: blocked_urgency`. Use `zen_dojotools_postman` for push. |
| 4–8 | Normal voice announcement range |
| 9–10 | Life-safety bypass — skips the sleep gate (smoke, CO, flood, alarm only) |

### Gates (in order)

1. **Urgency required** — `urgency` must be passed. Returns `status: urgency_required`.
2. **Urgency gate** — urgency ≤ 3 → returns `status: blocked_urgency`. Use Postman for push.
3. **Sleep gate** — blocked when `binary_sensor.zen_quiet_hours` is `on` or home mode is `Night`/`Night-Late`. Returns `status: blocked_sleep`. Urgency ≥ 9 bypasses.
4. **Dedup gate** — same caller + message combination within the dedup window (default 300s, configurable in `zen_scheduler_config`) → returns `status: dedup`. Prevents looping components from spamming audio.

TTS entity resolution: `label_entities(slugify(area))` intersected with `tts_output` label. If no TTS entity is found for an area, that area returns `"No TTS target available for area."` without failing the others.

After a successful fire, an entry is written to `zen_announcement_log` in the Kata cabinet (capped at 50 entries).

---

## zen_sutra_music_search (Internal Sutra)

**Not MCP-exposed.** Internal Music Assistant search connector called by `zen_dojotools_media_manager`.

For music discovery use `zen_dojotools_media_manager`:
- `mode=stacks_by_anchor` — ranked evidence + playback hints + profile pref re-ranking
- `mode=search` — lightweight raw MA results without Lens envelope

Direct MA service wrapper. Fields: `query`, `artist`, `album`, `media_type`, `number`, `status` (library_only). Returns the MA grouped response `{tracks, albums, playlists, artists, radio, audiobooks, podcasts}`.

---

## zen_dojotools_help — Global Help Front Door (v3.0.0)

`zen_dojotools_help` is now more than the system-overview tool — it's the front door for discovering and proxying to any tool's own help. Four modes:

| Mode | Does |
|------|------|
| `about` (default) | System overview including architecture, design principles, safety notes, the Pantheon roster, module list, escalation contacts, and a real-time inventory of all `zen`-labeled scripts currently loaded in HA. |
| `help` | Proxy to a specific tool's own `mode=help` — pass `tool=<script_name>`. Returns that tool's canonical help schema directly. Requires `tool`; without it, returns an error naming `mode=directory` as the discovery path. |
| `directory` | Auto-discover every `zen_dojotools_*`/`zen_stack_*`/`zen_sutra_*`/`zen_codex_*` script currently loaded (`on`/`off` state) and check each one's `tool_manifest` for `help` in its supported modes — no static list to maintain, new tools appear automatically as long as they implement `mode=help`. |
| `tool_manifest` | Self-description (UMP contract). |

```yaml
# System overview
zen_dojotools_help:
  mode: about

# Proxy to another tool's help
zen_dojotools_help:
  mode: help
  tool: zen_dojotools_finance

# Discover every help-capable tool currently loaded
zen_dojotools_help:
  mode: directory
```

`action=` remains as a backward-compatible alias for `about`/`tool_manifest` only — use `mode=` for the full mode set, including `help` and `directory`.

Useful for giving the AI a fresh picture of what's available when its context is stale, or for routing a "how do I use X" question straight to X's own documentation instead of guessing.

---

## zen_dojotools_wait

Timed delay. Useful for narrative pacing or spacing out sequential tool calls.

| Field | Type | Default | Range |
|---|---|---|---|
| `duration` | number | `15` | 1–120 seconds |

---

## dojotools_volume_auditor

Cabinet volume accessibility scanner. Reports which labeled cabinets are reachable and which are in `unknown` or `unavailable` state. Read-only — makes no changes.

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `required_labels` | text | `cabinet` | Comma- or space-separated label list |
| `require_all` | boolean | `false` | Require ALL labels to match (vs. ANY) |
| `include_hidden` | boolean | `false` | Include hidden/system volumes |

### Response

```json
{
  "status": "success",
  "result": {
    "scanned": 14,
    "accessible": ["sensor.zenos_dojo_cabinet", "..."],
    "issues": {
      "sensor.zenos_index_cabinet": { "reason": "unavailable" }
    }
  }
}
```

Use this when a script can't find a cabinet it expects — it shows which volumes are actually reachable.

---

## zen_dojotools_notification_router — Retired

**No longer exists (2026.7.1).** There was never a backing script behind this name — only a dispatcher compat arm forwarding the tool name. That arm was removed in dispatcher v5.3.0 (2026-07-04); a caller using this name now gets the dispatcher's `unknown_tool` structured fault. Use `zen_dojotools_postman` with `mode: resolve_and_dispatch` directly — it owns the authority stack, `kata_input` derivation, image payloads, actionable response buttons, and full `notification_data` passthrough.

---

## Canonical HA Domain Tools

These seven scripts are the preferred interface for their respective HA domains. They resolve shorthand names (e.g., `"office_occupancy"` → `input_select.office_occupancy`), validate inputs against entity constraints before writing, and return structured responses with `ok: true/false`. Fall back to HA built-ins only if a tool is confirmed non-functional.

All support `mode: tool_manifest` for self-description. All accept `caller_token` echoed in the response.

---

### zen_dojotools_select_control

GET+SET for `select` and `input_select` entities.

| Field | Description |
|---|---|
| `name` | Entity ID or shorthand name |
| `option` | Option to set. Omit to read. |
| `get` | Any truthy value forces read-only mode. |

Validates `option` against `state_attr(entity_id, 'options')` before applying. Returns `error: invalid_option` with `supported` list if the option is not in the entity's option list.

---

### zen_dojotools_number

GET+SET+arithmetic for `number` and `input_number` entities. Enforces entity `min`/`max`.

| Field | Description |
|---|---|
| `name` | Entity ID or shorthand name |
| `value` | Direct value to set |
| `operation` | `set` (default), `add`, `sub`, `mul`, `div`, `inc`, `dec` |
| `amount` | Operand for add/sub/mul/div |
| `cast` | `float` (default) or `int` |

Returns `error: out_of_range` with `min`/`max`/`requested` if the computed value would exceed the entity's bounds.

---

### zen_dojotools_text

GET+SET+string operations for `text` and `input_text` entities. Enforces 255-character HA platform limit (overflow truncated).

| Field | Description |
|---|---|
| `name` | Entity ID or shorthand name |
| `value` | Text to store |
| `operation` | `set` (default), `clear`, `append`, `prepend`, `replace` |
| `search_text` | Substring to find (used with `operation: replace`) |
| `replace_text` | Replacement string (used with `operation: replace`) |

---

### zen_dojotools_climate

GET+SET for `climate` entities. Inspects `supported_features` before applying — fails closed on unsupported feature. Multiple setters may be sent in one call. Omit all to read state and capabilities.

GET response includes a `topology_context` block: open doors/windows, area temperature/humidity sensors, adjacent HVAC bleed portals (via Room Manager), and a natural vent advisory.

| Field | Description |
|---|---|
| `name` | Entity ID or shorthand name |
| `room` | Area for topology context in GET mode (auto-detected if omitted) |
| `temperature` | Target temperature (single setpoint) |
| `target_low` / `target_high` | Auto/heat-cool range |
| `hvac_mode` | HVAC operating mode — read first to see supported modes |
| `fan_mode` | Fan mode |
| `swing_mode` | Swing mode |
| `preset` | Preset mode (eco, sleep, away, etc.) |
| `humidity` | Target humidity (if supported) |
| `aux_heat` | ~~Auxiliary heat~~ — `not_implemented`. `climate.set_aux_heat` was removed from HA with no replacement (surfaced by Spook v6 sweep). |
| `power` | Power control — `on`/`off` (via hvac_mode) |
| `get` | Any truthy value forces read-only |

---

### zen_dojotools_water_heater

GET+SET for `water_heater` entities. Inspects `supported_features` before applying — fails closed on unsupported feature. Multiple setters may be sent in one call.

Supported features bitmask: `TARGET_TEMPERATURE=1`, `OPERATION_MODE=2`, `AWAY_MODE=4`, `ON_OFF=8`.

| Field | Description |
|---|---|
| `name` | Entity ID or shorthand name |
| `temperature` | Target temperature — clamped to entity min/max (not rejected) |
| `mode` | Operation mode (e.g., `eco`, `heat_pump`) — validated against `operation_list` |
| `away` | Away mode — `on`/`off` |
| `power` | Power — `on`/`off` (requires `ON_OFF` feature) |
| `get` | Any truthy value forces read-only |

GET response includes `current_temperature`, `state`, and the full `capabilities` block.

---

### zen_dojotools_datetime

GET+SET for `input_datetime` entities. Resolves shorthand or full entity IDs. Accepts any datetime string parseable by `as_datetime()` including ISO 8601 with timezone offset.

| Field | Description |
|---|---|
| `name` | Entity ID or shorthand name |
| `value` | Datetime string (ISO 8601 recommended). Omit to read. |
| `get` | Any truthy value forces read-only |

GET response includes `has_date` and `has_time` flags.

---

### zen_dojotools_zones

Full zone CRUD and haversine bearing for HA zones. Zones defined in `configuration.yaml` are read-only — only dynamically created zones (via UI or this tool) support write operations. The `zone.home` zone cannot be deleted.

| Mode | Required Fields | Description |
|---|---|---|
| `read` | — | List all zones |
| `read` | `entity_id` | Inspect single zone (lat, lon, radius, persons, editable) |
| `create` | `name`, `latitude`, `longitude` | Create zone (radius defaults to 100m) |
| `update` | `entity_id` | Update any combination of name, lat, lon, radius, icon, passive |
| `delete` | `entity_id`, `confirm=true` | Delete zone (must be editable). Requires `confirm=true` — returns `confirm_required` with zone name if omitted. |
| `bearing` | `entity_id` | Haversine distance + compass direction from `zone.home` to target |
| `bearing` | `entity_id`, `origin_entity_id` | Zone-to-zone bearing |
| `bearing` | `entity_id`, `origin_latitude`, `origin_longitude` | Arbitrary-coordinates-to-zone bearing |

Optional fields for `create`/`update`: `icon` (MDI slug), `passive` (boolean — hidden from frontend).

`confirm=true` is required for `delete`. Omitting it returns `confirm_required` with the zone's friendly name so the caller can surface a confirmation before proceeding. This guard was added when `zone.edit` → `zone.update` was corrected as part of the Spook v6 compatibility sweep.
