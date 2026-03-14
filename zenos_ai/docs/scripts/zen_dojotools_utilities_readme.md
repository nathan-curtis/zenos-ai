# Zen DojoTools Utilities — 4.2.0

*Calculator, dice, announcements, music search, notifications, system help, and misc tools*

---

## Overview

Utilities is a collection of general-purpose tools that don't belong to a specific subsystem. Eight scripts covering math, randomness, TTS, music, notifications, system introspection, and cabinet auditing.

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

TTS announcement router to HA areas.

### Input Fields

| Field | Type | Description |
|---|---|---|
| `area` | area (multi) | One or more HA areas to announce to |
| `message` | text | Message to speak (max 255 characters) |

Resolves TTS media players from `label_entities(slugify(area))` and the `tts_output` label. If no TTS entity is found for an area, returns `"No TTS target available for area."` for that area without failing the others.

### Response

```json
{
  "results": [
    { "tts": { "room": "living_room", "entity_id": "media_player.living_room_speaker", "message": "Dinner is ready.", "status": "sent" } }
  ]
}
```

---

## zen_dojotools_music_search

Music Assistant library and internet search wrapper.

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `query` | text | — | Search term |
| `artist` | text | — | Filter by artist |
| `album` | text | — | Filter by album |
| `media_type` | select (multi) | — | `artist`, `album`, `track`, `playlist`, `radio`, `audiobook`, `podcast` |
| `number` | number | `50` | Result limit (1–50) |
| `status` | boolean | `true` | `true` = library only; `false` = library + internet |

Returns the Music Assistant search result object.

---

## zen_dojotools_help

Returns a live system overview including architecture, design principles, safety notes, the Pantheon, module list, and a real-time inventory of all `zen`-labeled scripts currently loaded in HA.

```yaml
action: about
```

Useful for giving the AI a fresh picture of what's available when its context is stale.

---

## zen_dojotools_wait

Timed delay. Useful for narrative pacing or spacing out sequential tool calls.

| Field | Type | Default | Range |
|---|---|---|---|
| `duration` | number | `15` | 1–120 seconds |

---

## zen_dojotools_notification_router

Multi-target notification router with quiet hours enforcement, HTML support, and breakthrough override.

### Input Fields

| Category | Fields |
|---|---|
| Content | `title` (required), `message` (required), `subject`, `tts_text` |
| Routing | `notification_targets` (default: `Admin Devices`), `click_action`, `group`, `tag` |
| Appearance | `color`, `icon_url`, `visibility`, `sticky`, `persistent`, `timeout` |
| Android | `channel`, `importance` (`min`/`low`/`default`/`high`/`max`), `vibration_pattern`, `led_color` |
| Override | `breakthrough` (boolean) — bypasses quiet/work hours blocks |

### Targets

| Target | Quiet/Work Hours Blocked? |
|---|---|
| Admin Devices | Never blocked |
| Family Devices | Blocked unless breakthrough |
| Nathan's Phone | Blocked unless breakthrough |
| Kim's Phone | Blocked unless breakthrough |

### Breakthrough Logic

Breakthrough is allowed when `importance` is `high` or `max` AND `breakthrough: true`. Without breakthrough, notifications to non-admin targets are blocked during quiet hours and work hours.

```json
{ "error": "send blocked, use high importance or better and request breakthrough if appropriate." }
```

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
