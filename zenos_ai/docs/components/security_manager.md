# ZenOS-AI Security Manager

**Version:** 5.2.0
**File:** `dojotools/dojotools_security_manager.yaml`
**Replaces:** `zen_dojotools_alarm_panel` (deleted)

**Entities:**
- `script.zen_dojotools_security_manager` — MCP-exposed alarm panel tool (**MCP-exposed**)

---

## Overview

Security Manager is the canonical alarm panel tool for ZenOS-AI. It discovers panels via the `alarm_panel` label and walks the device tree for zone sensors — no hardcoded entity IDs. It surfaces zone state, system health, and a cross-reference to camera coverage, then hands off to the visual analysis and spatial context tools via the lens pattern.

---

## Lens Pattern

Security Manager is one of three complementary security views:

| Lens | Tool | What it gives you |
|------|------|-------------------|
| Zone/panel | `zen_dojotools_security_manager` | Panel state, zones (open/fault/bypass), system health, keypad, chime |
| Visual | `zen_dojotools_camera` | Camera snapshot analysis for any entity listed in `cameras_by_area` |
| Spatial | `zen_dojotools_room_manager mode=get context_slices=+security` | Per-room zone + camera inventory |

The `read_state` response includes a `lens` field describing the handoff to the other tools. Use all three together for a complete security picture.

---

## Actions

| Action | What it does |
|--------|-------------|
| `read_state` (default) | Full panel snapshot: state, zones, system health, cameras_by_area, lens handoff note. |
| `query` | List all `alarm_control_panel` entities tagged with `alarm_panel` label, with state. |
| `arm` | Arm the panel. Requires `arm_mode`. |
| `disarm` | Disarm the panel. |
| `get_policy` | Read the `_alert_policy` from the household cabinet security_manager drawer. |
| `set_alert_policy` | Merge `policy_patch` JSON into the alert policy. Partial update — existing keys not in patch are preserved. |

---

## Inputs

| Field | Required For | Type | Description |
|-------|-------------|------|-------------|
| `action` | — | string | Operation. Default: `read_state`. |
| `arm_mode` | `arm` | string | `away` \| `home` \| `night` \| `vacation`. |
| `panel` | optional | entity | Override the auto-discovered panel entity. Omit to use `label_entities('alarm_panel')`. |
| `policy_patch` | `set_alert_policy` | JSON string | Event type → policy overrides dict. Partial — only keys in patch are updated. |
| `caller_token` | — | string | Opaque audit token echoed in response. |

---

## read_state Response

```json
{
  "action": "read_state",
  "panel_entity": "alarm_control_panel.home_alarm",
  "state": "armed_away",
  "zones": [
    { "entity_id": "binary_sensor.front_door", "state": "off", "fault": false, "bypass": false },
    ...
  ],
  "system_health": {
    "ac_power": true,
    "battery_ok": true,
    "fire": false,
    "alarm": false,
    "bell": false
  },
  "keypad": { ... },
  "chime": true,
  "cameras_by_area": {
    "front_yard": ["camera.front_doorbell"],
    "garage": ["camera.garage_cam"]
  },
  "lens": "Security Manager = zone/panel lens. For visual: zen_dojotools_camera. For spatial: zen_dojotools_room_manager mode=get context_slices=+security."
}
```

`cameras_by_area` groups `security_camera`-labeled cameras by `area_id`. Use `zen_dojotools_camera` for snapshot analysis of any listed entity.

---

## Zone Discovery

Discovery uses `device_entities(device_id)` to walk the full device tree for each alarm panel device — no entity IDs are hardcoded. Zones are classified by `device_class` and paired by `_fault` / `_bypass` `object_id` suffix convention.

Zone states:
- `state` — `on` (open/active) or `off` (closed/secure)
- `fault` — true if a `_fault` companion sensor is `on`
- `bypass` — true if a `_bypass` companion sensor is `on`

### Binary Sensor Device Class Translation

v5.2.0 adds human-readable `_interp` strings derived from `device_class`. Raw `on`/`off` states are translated per class before being surfaced in the response:

| device_class | `on` → `_interp` | `off` → `_interp` |
|---|---|---|
| `power` | `power_on` | `power_off` |
| `battery` | `battery_low` | `battery_ok` |
| `problem` | `problem_detected` | `ok` |
| `safety` / `gas` / `smoke` | `alarm_active` | `clear` |
| `lock` | `unlocked` | `locked` |
| `door` / `window` / `opening` / `garage_door` | `open` | `closed` |
| (default) | `active` | `inactive` |

**STATE FIELD READING RULES (loaded into KFC):** `locked` → SECURED. `on` → contact OPEN. `off` → contact CLOSED. `open` → cover open. Never infer breach from entity name alone — read `device_class`.

### System Health Summary Fields

`read_state` response v5.2.0 adds two top-level summary fields:

| Field | Type | Description |
|-------|------|-------------|
| `system_ok` | bool | `true` when no zones are faulted or open and panel is not in alarm |
| `system_issues` | list of strings | Names of zones contributing to a non-ok state |

---

## Alert Policy

`set_alert_policy` takes a JSON string keyed by event type. Each event type maps to a policy dict:

| Key | Type | Purpose |
|-----|------|---------|
| `urgency` | 1–10 | Postman urgency for this event type |
| `notify` | bool | Whether to route this event to Postman |
| `channel_hint` | string | `push` \| `tts` \| `teams` |
| `title` | string | Override notification title |

**Event types:** `alarm_triggered`, `alarm_pending`, `alarm_arm_away`, `alarm_arm_home`, `alarm_arm_night`, `alarm_disarm`, `panel_fault`, `smoke_co`, `door_open_armed`, `door_open_night`, `garage_open_armed`, `lock_unlocked_armed`, `lock_unlocked_night`, `lock_changed`, `person_detected_armed`.

Example:

```yaml
zen_dojotools_security_manager:
  action: set_alert_policy
  policy_patch: '{"alarm_triggered": {"urgency": 10, "notify": true, "channel_hint": "push"}}'
```

---

## Setup

1. Apply the `alarm_panel` label to your `alarm_control_panel` entity in Home Assistant.
2. Apply the `security_camera` label to any cameras you want in `cameras_by_area`.
3. Call `read_state` to verify panel discovery and zone inventory.

**If `zen_dojotools_security_manager` shows `unavailable` after reload:** HA may be holding a stale entity entry for the old `zen_dojotools_alarm_panel`. Go to Settings → Devices & Services → Entities, find the stale entry, delete it, then reload scripts.

---

## Integration

- **Room Manager** — `mode=get context_slices=+security` returns per-room zone + camera inventory derived from Security Manager data.
- **AlertManager** — use `notify_target: postman` with `zen_event(kind: alert_fire)` to route panel events through the standard alert pipeline.
- **Camera** — `zen_dojotools_camera tool=look` on any entity listed in `cameras_by_area` for visual analysis.

---

## KFC Registration (KF5)

`zen_dojotools_security_manager` self-registers via `mode=kfc_manifest`, declaring **4 components** — see [Building a KFC — KF5](../kung_fu/building_a_kfc.md#kf5-self-registering-tools):

| Component | Role |
|---|---|
| `security_manager` | Rollup — seeds from `zen_dojotools_room_manager mode=home_overview`, synthesizes panel + perimeter + camera cache into one picture. The rollup's own summarizer instructions now point at `signal.active_rooms[].v3_occupancy` (Room Manager v3's live per-room state, null if that room has no v3 sensor deployed) as a sharper occupancy signal than the coarse lights/motion heuristic already driving `active_rooms[]` membership — prefer it when present. |
| `security_perimeter` | Sub-component — door/lock/garage state transitions, security-context filtered (panel armed or Night mode) |
| `security_panel` | Sub-component — DSC alarm state machine: `panel_state`, `arming_mode`, `zones`, `system_health` |
| `security_cameras` | Sub-component — which cameras triggered, `detected_subjects`, `activity_type` |

The three sub-components each declare `parent_component_key: security_manager` — the rollup component reads their cached katas (`panel_cache`, `perimeter_cache`, `camera_cache`) rather than re-querying each subsystem itself.
