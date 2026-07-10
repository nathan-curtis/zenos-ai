# ZenOS-AI SystemTools

**Version:** 5.2.0
**Package:** `packages/zenos_ai/dojotools/dojotools_systemtools.yaml`
**Primary script:** `zen_dojotools_systemtools`

---

## Overview

SystemTools is the household-state control surface: current home mode, quiet/work hours, the 6 scheduler anchor times that other automations key off of, and guest/entertaining toggles. It also carries the `zen_sutra_ha_api` internal REST bridge and the pre-existing repairs/notice-dashboard modes.

This doc covers the modes added in v5.2.0 (home_status, home_mode, quiet_hours, work_hours, scheduler_anchors, guest_mode, entertaining) plus KFC registration. For the older repairs/notice-dashboard modes, see the tool's own `mode=help`.

---

## home_status

Read-only rollup: current home mode, quiet/work hours active state, all 6 scheduler anchors, and whether the anchors are in valid ascending order.

```yaml
zen_dojotools_systemtools:
  tool: home_status
```

Includes `anchor_order_valid` (bool) and `anchor_conflicts` (list of `name/name` pairs currently out of order) — both skip any anchor currently `unknown`/`unavailable` rather than treating it as a false conflict.

---

## home_mode

Get or set the household mode (`input_select.zen_home_mode`).

```yaml
# Get
zen_dojotools_systemtools:
  tool: home_mode

# Set
zen_dojotools_systemtools:
  tool: home_mode
  value: "Home-Evening"
```

Valid values: `Home-Wake`, `Home-Morning`, `Home`, `Home-Evening`, `Night`, `Night-Late`, `Away`, `Paused`.

**Safety gates:**
- **`Paused` is a hard block on writes.** If the system is currently `Paused`, no agent can change `home_mode` under any circumstances — a human must change it directly in the HA UI. This is a deliberate `ALL STOP` — there is no `confirm` override for it.
- **Setting `Away` requires `confirm_action: true`.** The response includes a warning naming how many persons are currently detected home (`zone.home`) and an explicit alarm-arming warning if anyone is detected present.

---

## quiet_hours / work_hours

Get or set the quiet-hours / work-hours window (`input_datetime.zen_quiet_hours_start/end`, `input_datetime.zen_work_hours_start/end`).

```yaml
# Get
zen_dojotools_systemtools:
  tool: quiet_hours

# Set — value is 'HH:MM/HH:MM' (start/end)
zen_dojotools_systemtools:
  tool: quiet_hours
  value: "23:00/06:00"
```

Midnight wrap is supported (e.g. `23:00/06:00` for quiet hours spanning midnight). Same shape for `work_hours` (e.g. `09:00/17:00`). The get response includes `active` — the live state of `binary_sensor.zen_quiet_hours` / the work-hours equivalent.

---

## scheduler_anchors

Get all 6 named cutover times, or set one.

```yaml
# Get all anchors
zen_dojotools_systemtools:
  tool: scheduler_anchors

# Set one anchor
zen_dojotools_systemtools:
  tool: scheduler_anchors
  anchor: morning
  value: "07:30"
```

Anchor names, in required ascending order: `wake` < `morning` < `daytime` < `evening` < `night` < `late_night`.

**Write is blocked if it would break ascending order.** Setting an anchor to a time that conflicts with its neighbors returns `blocked: true`, the list of conflicting pairs, the house's current anchor times, and the proposed value — the caller (agent) is expected to suggest a corrected time rather than the system silently accepting an invalid ordering. The conflict check itself skips any neighbor anchor that's currently `unknown`/`unavailable` rather than treating that as a false block.

---

## guest_mode / entertaining

Simple on/off toggles (`input_boolean.zen_guest_mode`, entertaining's equivalent).

```yaml
zen_dojotools_systemtools:
  tool: guest_mode
  value: "on"
```

Accepts `on`/`true`/`1` and `off`/`false`/`0` (case-insensitive). Omitting `value` returns the current state (`get`).

---

## KFC Registration (KF5)

`zen_dojotools_systemtools` self-registers via `mode=kfc_manifest` — see [Building a KFC — KF5](../kung_fu/building_a_kfc.md#kf5-self-registering-tools). Declares one component, `zen_system` — a rollup-only kata. Per its own `component_instructions`: it deliberately does **not** expand individual component detail — if a specific subsystem is warn/critical, `zen_system` names it and stops, leaving detail to that subsystem's own monk.

`zen_sutra_ha_api` (the internal REST bridge in this same file) also gained a `tool_manifest` early-exit for UMP self-description — it is not a KFC/Lens Bus participant, just a standard self-describing internal tool.

---

## Source Notes

This page is derived from `packages/zenos_ai/dojotools/dojotools_systemtools.yaml`.
