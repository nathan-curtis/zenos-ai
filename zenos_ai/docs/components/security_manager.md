# ZenOS-AI Security Manager

**Version:** 5.26.2

> **2026.12.0 (5.10.0 to 5.26.2): the foundation, vacuum sync, lighting, exterior covers and the appliance sentry.** Four new reads, no cert: `readiness`, `override_list`, `activity` and `lens`. A record for temporary overrides and an activity ledger, with internal actions that only automations can call. A second automation, the Zen Security Sentry, ends overrides on time. The first behavior built on it is vacuum sync, off by default: with `vac_arm_override_enabled` on, the panel goes from Away to Home while the vacuum cleans and back to Away at the dock. An optional notice when a room reads occupied while armed Away, security lighting at dusk and dawn, and arrival lights after dark, each off by default. While an override is in force the agent knows it (one Priority Inject line). The appliance sentry tells the household when a labeled appliance has been on with nobody home (off by default). Exterior covers (garage doors and gates) left open, at Night, or in damaging wind can be offered a close, closed, or reported, by setting, each off by default; in a secure stance (armed, or Away or Night) a cover left open closes when nobody answers. `auto_arm_config` with no patch now reads the settings. See [Readiness, overrides and activity](#readiness-overrides-and-activity), [Occupied while away](#occupied-while-away) and [Exterior covers](#exterior-covers-5230).

> **2026.11.0 (5.9.1, 5.9.2):** `auto_arm_check` is refused from a user or agent session; only the auto-arm automations call it. A `zone.home` that is unknown or unavailable is never an empty house: the count reads -1, nothing arms and the arrival path doesn't run, and the Night-Late and Away watchers ignore a change from unknown or unavailable (a template reload).

> **2026.11.0 (5.8.0, 5.9.0):** the `security_manager` KFC drops `master_switch` and its instruction; on and off is `meta.enabled`. `security_perimeter`, `security_panel` and `security_cameras` are cache slots (`role: cache`, no triggers): a trigger pack of your own writes them and the `security_manager` rollup reads them as `perimeter_cache`, `panel_cache` and `camera_cache`. On a stock install nothing fills them, and the rollup runs on what it has. Its summary seeds Room Manager's `home_overview`, so under `seed_cert_mode` enforce it needs `zenos.presence.read` with stage `s`.
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
| `arm` | Arm the panel. Requires `arm_mode`. **Identity-gated (2026-08-15)** — requires the `zenos.actuation.security_control` certification, fail-closed until granted. |
| `disarm` | Disarm the panel. **Identity-gated, higher tier than arm** — requires `zenos.actuation.security_control` **plus a fresh live household-admin ack every call**, no standing-cert shortcut. Disarming is the highest-risk direction; arming down is cert-only. |
| `get_policy` | Read the `_alert_policy` from the household cabinet security_manager drawer. |
| `set_alert_policy` | Merge `policy_patch` JSON into the alert policy. Partial update — existing keys not in patch are preserved. Same `zenos.actuation.security_control` cert gate as `arm`. |
| `readiness` | Can the house arm now, and if not, what's in the way. Read, no cert. See [Readiness](#readiness). |
| `summary` | (5.26.0) One scored security answer, computed in code: `urgency` 0 to 10 (the pipeline scale, so an alarm going off reaches the agent's active list), `action_required` (5 or more), `reasons[]` highest first, `attention[]` with names, the panel, `people_home`, `home_mode`, `secure_stance`, `readiness`, Room Manager v3 `rooms[]` (every room not vacant), `covers_left_open[]`, `appliances_on_nobody_home[]`, `overrides[]` and `sentry_state`. The seed of the `security_manager` KFC. Read: asking needs no cert; the background summary needs `zenos.security.read`. |
| `override_list` | The temporary overrides in force: a change the system made to the panel on its own, what it left, what it moved to, why, and minutes left. Read, no cert. |
| `activity` | What security has done on its own, and what it declined to do, newest first. `activity_hours` (default 24, at most 168), `activity_limit` (default 50, at most 150). Read, no cert. |
| `lens` | The one security answer other tools use: the panel and its state, ready and blocker count, any temporary override as a plain sentence (or one that outlived its time), the last ledger line, `sentry_state` and one plain `line`. Room Manager's `home_overview` and `summary` carry it. Read, no cert. |
| `auto_arm_config` | Read or change the auto-arm settings (`auto_arm_patch` JSON). With no patch it reads the settings and the `vacuum` block (see [Vacuum sync](#vacuum-sync)). Changing them needs `zenos.actuation.security_control`. |
| `override_set`, `override_end`, `override_sweep`, `ledger_add`, `auto_arm_check` | Internal: not for agents or people. Each refuses any call that has a user in it; automations call them. |
| `help` | Full action reference — all actions, fields, and cert tiers in one call. Not gated. |

**Self-published cert:** `zenos.actuation.security_control` is declared in this tool's own `tool_manifest.certs_required` — that's what makes it a valid `cert_grant` target under the live-calculated catalog (`zen_dojotools_manifest mode=cert_audit`; see `zen_dojotools_profile_readme.md`'s certification section). Grant via `zen_dojotools_persona_editor mode=cert_req_grant cert_component=zenos.actuation.security_control`.

**Scoped disarm override:** unlike locks/covers, there's no per-entity target for a single alarm panel to scope against — an admin exempts disarm from the every-call ack via `cert_grant cert_component=zenos.actuation.security_control cert_scope=["disarm"]`, a literal token rather than an entity_id. Resolution goes through the same shared `cert_scope_check` macro as every other gated tool (2026-08-16 centralization) — see the [Security & Certification System operator manual](../getting_started/security_certification_manual.md).

**v5.3.0 (2026-08-15):** full SESE (single-entry/single-exit) and canonical `envelope()` migration alongside the identity-gate work above — response shape internals changed accordingly (existing fields now live under `result`), same pattern as `zen_health_report`'s own envelope adoption.

---

## Readiness, overrides and activity

### Readiness

`readiness` answers "can the house arm now?" from three things: the panel's own zones (and its own "not ready" flag), entities labeled `zen_door` (doors, windows, garage doors), and entities labeled `ext_lock` (exterior locks).

- **Blockers:** a panel that is pending, triggered or unreadable; the panel saying it isn't ready; an open or faulted zone that isn't bypassed; an open or unreadable labeled door or window; a jammed or unreadable exterior lock.
- **Fixable:** an unlocked exterior lock. It doesn't block arming.
- **Notes:** bypassed zones.
- **`blind: true`:** nothing visible was checked (no panel zones and nothing labeled `zen_door`), and the summary says so instead of a bare "ready".

### Temporary overrides

A temporary override is a change the system makes to the panel on its own for a while, kept as a record (one per kind) in the household `security_manager` drawer, so it survives a reload or restart: what it left, what it moved to, why, when it began and ends, and a ceiling from its start (default 180 minutes) its end can never pass. `override_list` reads them. Only automations make and end them, and each step (begin, extend, clamp, end, expiry) writes a line to the activity ledger.

While an override is in force the agent knows it: `override_set` writes one Priority Inject line to its active context (provider `security_override_<kind>`, urgency `urgent`, expiring with the override), for example "Security is armed home instead of armed away for vacuum sync, since 10:02. It goes back by about 10:45." An extension rewrites it; `override_end` and the Sentry's sweep clear it (5.22.0).

**Spook is required.** The Zen Security Sentry ends overrides on time, and it runs only when Spook's trigger platforms work. Without it, `override_set` and turning on `vac_arm_override_enabled` answer `spook_required`; turning that setting off always works. `lens` says when an override has outlived its time.

### The activity ledger

What security did on its own, and what it declined to do, is a short list in the household cabinet: at most 150 entries, at most 7 days old. `activity` reads it.

### The Zen Security Sentry

A second automation, kept apart from the Auto-Arm Watcher so a missing or changed Spook can't switch the watcher off. It runs every minute while an override exists, ends any that have expired (one ledger line each), and fires a `zen_event` of kind `security_override_expired` so the behavior that owns the override can put the panel back. For an expired override it never touches the panel itself; for vacuum sync it also reconciles a dock that was never reported (see [Vacuum sync](#vacuum-sync)). The health report names both watchers and raises a CRITICAL when one is unavailable. Security Manager finds the Sentry by its automation id, so renaming its entity doesn't make it read as off (5.23.1).

### Vacuum sync

`vac_arm_override_enabled` (off by default, asked once when turned on) lets the vacuum run while the house is armed Away. When the vacuum starts cleaning with the panel armed Away, Security Manager moves the panel to Home (override `vacuum_sync`) and sends a notice with a **Cancel** button; Cancel stops the run through Autovac `mode=stop` and keeps the downgrade until the vacuum docks. Away returns when the vacuum docks (after `vac_downgrade_dock_wait_min`, default 5 minutes, because a mop wash docks mid-run; one minute after a cancelled run, read from `autovac_docked` `cancelled`), or when the override expires. Away returns only if the house is empty. A hand disarm or re-arm ends the override and nothing is put back. If the alarm triggers during an override, the run is stopped, the override ends, a ledger line is written and the person is told; the panel is left alone.

A reload or restart during a run cannot strand the panel at Home: each minute the Sentry reconciles. A docked vacuum past the dock wait plus two minutes with Autovac idle ends the override and restores Away.

`auto_arm_config` returns a `vacuum` block saying whether these settings apply here: `ready`, `no_vacuum` (nothing labeled `autovac`), `autovac_missing`, `autovac_disabled` or `vacuum_unavailable`. Turning `vac_arm_override_enabled` on is refused with `vacuum_required` in the first three; turning it off is always allowed. Autovac's configure shows and edits the two vacuum settings but hands the edit here.

### Occupied while away

`notify_occupied_while_away` (off by default, set through `auto_arm_config`; `auto_arm_enabled` does not need to be on) sends one notice when the panel is armed Away and a room reads occupied, engaged or asleep for two minutes. It writes a ledger line (`occupied_while_away`) naming the rooms and sends a push; the panel is never changed. A vacuum downgrade or any override in force is skipped. A pet or a visitor can set it off, which is why it is off by default. The Auto-Arm Watcher fires it from a template trigger over the `alarm_panel` and `zen_room_state` labels; the read shows the setting under `notice_settings`. Room Manager `summary` reports the same condition as `occupied_while_away`.

### Dusk and dawn lighting

`dusk_lights_enabled` (off by default) turns every light or switch carrying the `dusk_to_dawn` label on at civil dusk and off at civil dawn. `dusk_offset_min` and `dawn_offset_min` shift each time by minutes (negative is earlier; default 0). Set them through `auto_arm_config`; the read shows them under `lighting_settings` with the labeled entities. Two template triggers on the Auto-Arm Watcher read `sun.sun` `next_dusk` and `next_dawn` every minute, so each change lands within about a minute of its time; a start trigger sets the lights from the time of day after a restart. Each change writes a ledger line (`security_lights`). Nothing happens while nothing carries the label. The panel is never touched, and `auto_arm_enabled` does not apply.

### Arrival lights

`arrival_lights_enabled` (off by default) turns on the lights labeled `landing_lights` when a resident arrives home after dark. The arrival is a device tracker labeled `arrival_tracker` (normally the resident's phone) entering `zone.home`. A resident is a human user on the household cabinet, or in a family that belongs to it (two levels, as `membership` walks them), whose cabinet names a person entity. `arrival_lights_residents` can narrow it to named people (person entities or names); empty means every resident. After dark means the sun at or below `arrival_lights_dark_elevation` (default -6 degrees).

Only the landing lights that were off are turned on, and the record of exactly those is kept in the household cabinet (`_security_arrival`). They are turned off again after `arrival_lights_minutes` (default 10), so lights that were already on, such as dusk-to-dawn porch lights, are never turned off by it. A second arrival inside the window extends the time. The off step acts on any record that has run out, whatever the setting, and again after a restart, so a light it turned on is never stranded. Each change writes a ledger line (`arrival_lights`) naming the resident. The read shows the settings under `arrival_settings` with the labeled lights and trackers.

It prepares only. It never touches the alarm panel, and presence never disarms it.

### Exterior covers (5.23.0)

The garage family, by label. **Watched covers** are every cover labeled `secure_on_arm` (the covers the house already closes when it arms Away) and every cover labeled `exterior` with device class `garage`, `gate` or `door`. All settings are in `auto_arm_config` and all are off by default. A cover's own obstruction sensing is the door's job; the policies below are the household's call.

| Setting | Default | What it does |
|---|---|---|
| `cover_watch_enabled` | off | Act when a watched cover is left open `cover_left_open_minutes` (default 15). |
| `cover_day_policy` | `ask` | While the house is awake: `ask` (announce and ask on the phone, close only on a Yes), `ask_then_close` (close unless someone says No within five minutes), `close` (close and say so), `notify` (only say so). |
| `cover_secure_close_enabled` | on | (5.25.1) The secure-stance close handler, acting only while `cover_watch_enabled` is on. On: `cover_armed_policy` applies in a secure stance, and a cover left open while the house sleeps, or still open at the Night transition, closes quietly. Off: a secure stance uses `cover_day_policy`, and nothing closes overnight unless `cover_night_close_enabled` is on. |
| `cover_armed_policy` | `ask_then_close` | (5.25.0) With `cover_secure_close_enabled` on, used instead of `cover_day_policy` in a **secure stance**: the alarm panel armed in any mode, or the home mode Away, Night or Night-Late. The same four choices. While the house sleeps, `ask_then_close` and `close` close quietly with a note on waking, and the Night transition closes an open cover the same way. `auto_arm_config` shows `secure_stance_now`. |
| `cover_night_close_enabled` | off | Close quietly when the home mode turns Night, or when one is left open while the house sleeps. A note goes out when the house wakes. |
| `cover_wind_close_enabled` | off | The weather close: act on bad weather lasting five minutes with a watched cover open, by `cover_wind_policy` (default `ask_then_close`). |
| `cover_wind_threshold` | 50 mph / 80 | For a `wind_speed` sensor labeled `cover_wind`. |
| `cover_camera_look_enabled` | off | (5.27.0) When a watched cover opens, every camera labeled `cover_camera` takes a look through the camera tool, so who arrived or left is on record. |

**What counts as bad weather (5.28.0):** any entity labeled `cover_close_signal` that reads `on`, in any domain. That is the household's own "close the garage" switch: a template, a rain sensor, a weather alert. Otherwise the wind source, in order: a `binary_sensor` labeled `cover_wind` (on means close), a `wind_speed` sensor labeled `cover_wind` against `cover_wind_threshold`, else the weather entity's gust or wind speed against the damaging threshold for its unit.

#### Example close signal: headwind into the door

Nothing direction-aware ships by default. Raw wind speed is a poor guide for a garage: wind along the house does little, wind blowing straight into an open door does a lot. This template scores only the wind blowing **into** the opening and turns on past a threshold. Put it in your own configuration, then label `sensor.garage_headwind_close` with `cover_close_signal`.

You need two helpers (Settings > Devices & services > Helpers > Number): `input_number.garage_facing` (the compass bearing the door opening faces, 0 to 359) and `input_number.garage_headwind_limit` (the headwind that should close the door, for example 10 mph). Use your own weather entity in place of `weather.home`.

```yaml
template:
  - sensor:
      # Wind speed scaled by how squarely it blows into the opening:
      # full speed head-on, zero at 90 degrees or more off the facing.
      - name: "Garage Headwind"
        unique_id: garage_headwind
        unit_of_measurement: "mph"
        state: >-
          {%- set spd   = state_attr('weather.home', 'wind_speed') | float(0) -%}
          {%- set wbear = state_attr('weather.home', 'wind_bearing') | float(0) -%}
          {%- set face  = states('input_number.garage_facing') | float(0) -%}
          {%- set ang   = (face - wbear) | abs -%}
          {%- set ang   = ang if ang <= 180 else 360 - ang -%}
          {{ [((90 - ang) / 90) * spd, 0] | max | round(1) }}

      # The close signal: on when the headwind reaches the limit.
      - name: "Garage Headwind Close"
        unique_id: garage_headwind_close
        state: >-
          {{ 'on' if states('sensor.garage_headwind') | float(0) >= states('input_number.garage_headwind_limit') | float(0) else 'off' }}
```

Then turn on `cover_wind_close_enabled`. Security closes by `cover_wind_policy` once the signal has been on for five minutes with a watched cover open. Note that `wind_bearing` is where the wind comes **from**, so a door facing the same bearing takes the wind head-on.

**How it runs:** triggers on the Auto-Arm Watcher (`cover_left_open`, `cover_closed`, `cover_night`, `cover_wake`, `cover_wind`, `cover_ack`). A question goes out through the dispatcher **without waiting**, because the watcher is queued and must never hold a security event behind a five-minute question; its answer comes back as a `postman_ack` (`ack_owner: security_manager`), and the policy rides in its `ack_context`. While the house sleeps (Postman's sleep rule) nothing is asked: a close is logged and told on waking. Every close writes a ledger line; while a watched cover stays open past its time the agent has a Priority Inject line (`security_cover_open`), cleared when every watched cover is closed. `auto_arm_config` (no patch) shows `cover_settings` with the watched covers and the wind source. A cover labeled through `zen_dojotools_labels` is picked up at once (the tool re-arms the watcher, Labels 5.11.0); one labeled in the Home Assistant UI is seen once the watcher is turned off and on, or after a restart (#10716).

### Appliance sentry (5.24.0)

`appliance_sentry_enabled` (default off) tells the household when an appliance has been on `appliance_sentry_minutes` (default 10) with **nobody home**. Nobody home is the same rule auto-arm uses: `zone.home` at 0, and an unknown count never counts as empty. **Watched:** entities labeled both `appliance` (the label Room Manager's `+appliances` slice already uses) and `appliance_sentry`. Running means the entity reads `on`. **Inform only:** a phone notice (urgency 7), a ledger line, and a Priority Inject line (`security_appliance_away`) until the appliance is off or someone is home. The house never switches the appliance off (a range can't be). `auto_arm_config` (no patch) shows `appliance_settings` with the watched appliances. Labeling through `zen_dojotools_labels` re-arms the watcher (Labels 5.11.1).

### Who runs as whom

The watchers and the Autovac controller call these actions from automations with no user. Identity resolution maps every caller to the default AI user, so their cert checks pass on what that user holds (`zenos.actuation.security_control` for `auto_arm_config`, `zenos.autovac.operate` for the Autovac calls).

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

Discovery uses `device_entities(device_id)` to walk the full device tree for each alarm panel device, including the panel's child devices (HA 2026.9+, Security Manager 5.7.0) — no entity IDs are hardcoded. Zones are classified by `device_class` and paired by `_fault` / `_bypass` `object_id` suffix convention.

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

- **Room Manager** — `mode=get context_slices=+security` returns per-room zone + camera inventory derived from Security Manager data. `home_overview` and `summary` carry the temporary override (`temporary_override`, `stuck_overrides`, `override_line` in the security block) from `lens`, only when one exists.
- **AlertManager** — use `notify_target: postman` with `zen_event(kind: alert_fire)` to route panel events through the standard alert pipeline.
- **Camera** — `zen_dojotools_camera tool=look` on any entity listed in `cameras_by_area` for visual analysis.

---

## KFC Registration (KF6)

`zen_dojotools_security_manager` self-registers via `mode=kfc_manifest`, declaring **4 components** — see [Building a KFC — KF6](../kung_fu/building_a_kfc.md#kf6-two-ways-in-one-contract):

| Component | Role |
|---|---|
| `security_manager` | Rollup — seeds from this tool's `mode=summary` (5.26.0; scored in code, cert `zenos.security.read` level 1, in the `zenos_agent_advanced` bundle since 5.26.2), then layers the panel, perimeter and camera caches and the camera analysis on top. The summary carries Room Manager v3's live room states in `rooms[]`, which the instructions weigh (emergency, asleep, engaged, vacant). |
| `security_perimeter` | Sub-component — exterior doors and windows, `ext_lock` locks and watched covers, by label; security-context filtered (panel armed or Night mode) |
| `security_panel` | Sub-component — the alarm panel's state machine (the panel labeled `alarm_panel`): `panel_state`, `arming_mode`, `zones`, `system_health` |
| `security_cameras` | Sub-component — which cameras triggered, `detected_subjects`, `activity_type` |

The three sub-components each declare `parent_component_key: security_manager` — the rollup component reads their cached katas (`panel_cache`, `perimeter_cache`, `camera_cache`) rather than re-querying each subsystem itself.
