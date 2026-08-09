# ZenOS Plant Codex — EG4 Web Monitor (Solar/Battery)

**Integration:** [`joyfulhouse/eg4_web_monitor`](https://github.com/joyfulhouse/eg4_web_monitor) (HACS custom component)
**Consumes into:** `zen_dojotools_plant` (Plant Manager) — `mode=electric`'s nested `solar` key
**Status:** New Plant Manager capability as of v5.7.0 (2026-08-09) — solar/battery had no slot at all before this

---

## Overview

EG4 Web Monitor talks to EG4 solar inverters/battery systems (FlexBOSS, 18kPV, 12kPV, XP
series, LXP models) — production, battery storage, and grid interaction, not a circuit panel.
This is a fundamentally different shape from the [SPAN](span_panel_codex.md)/[Emporia
Vue](emporia_vue_codex.md) codexes: those are pure label-mapping guides because Plant Manager
already had `main_panel`/`sub_panel`/`consumed_energy` slots built for circuit-level monitoring.
**Plant Manager had no solar or battery concept at all before this Codex** — building it required
a real code change first (five new `zen_plant_*` labels plus a nested `solar` object in
`mode=electric`'s response), not just a mapping table. That work is done; this document is now
the mapping guide on top of it, same as SPAN's.

**Scope, deliberately kept tight for v1:**
- **Read-only.** EG4's write-capable entities (quick charge, EPS/backup mode, AC-couple toggle,
  SOC/voltage/current limits, peak-shaving parameters) are out of Plant Manager's scope
  entirely — tag and use them directly via `zen_dojotools_number`/`select_control`/`boolean`,
  the same generic domain tools every other writable entity in this system goes through. Plant
  Manager is read/overview-focused everywhere else; this doesn't change that.
- **Site-level rollup only.** EG4 explicitly supports multi-battery systems and multiple
  inverter families with per-leg split-phase detail. v1 **sums** power across every matching
  entity (PV production, battery power, grid power) and **averages** battery SOC across banks
  (a simple average, not capacity-weighted — good enough for "how full is the pack overall,"
  not a precision BMS readout). Per-inverter/per-bank breakdown is real future scope, not
  modeled today.
- **Response cost is zero for households without a panel.** The `solar` key is only present in
  `mode=electric`'s output when at least one of the five labels below resolves to a real entity
  — omitted (`null`) otherwise, same `available`-gate convention every other Plant block uses.

---

## Entity → Label Mapping

| EG4 sensor concept | Plant Manager slot | Label | Notes |
|---|---|---|---|
| PV/solar power (W) | `solar.pv_power_w` | `zen_plant_pv_power` | Summed across every tagged entity — tag each inverter's PV power sensor if multiple |
| Battery charge/discharge power (W) | `solar.battery_power_w` | `zen_plant_battery_power` | Summed across every tagged entity (multi-bank systems) |
| Battery state of charge (%) | `solar.battery_soc_pct` | `zen_plant_battery_soc` | **Averaged**, not summed, across every tagged entity — a simple mean, not capacity-weighted |
| Grid power, import/export (W) | `solar.grid_power_w` | `zen_plant_grid_power` | Distinct from `zen_plant_site_power` (the whole-home total used by SPAN/Emporia installs) — this is the inverter's own grid-interaction reading |
| Off-Grid status (binary) | `solar.grid_connected` | `zen_plant_off_grid` | Tag the *off-grid* binary sensor (on = disconnected from grid). Plant Manager inverts it for you — `grid_connected` reads `true` when the panel is NOT off-grid |

`solar.inverter_count` and `solar.battery_bank_count` are also included in the response —
counts of tagged `zen_plant_pv_power`/`zen_plant_battery_soc` entities respectively, so you can
tell at a glance whether the sum/average is covering one device or several.

### Not currently modeled (genuine gaps, not bugs)

| EG4 sensor | Gap |
|---|---|
| Per-inverter/per-bank breakdown | v1 is site-level rollup only — see Scope above. A `detail=true` param on a future dedicated mode is the natural extension, same pattern as `mode=circuits`' own detail conventions elsewhere in Plant. |
| Battery cell-level voltage, cycle count | No slot. Genuinely diagnostic/BMS-depth data, not something Plant's overview-level response is trying to surface. Query directly via `zen_dojotools_inspect` if needed. |
| Inverter temperature, operating mode, fault/warning codes | No slot. Useful for a health/diagnostics view, not modeled in `mode=electric` today — real future scope if wanted. |
| Grid voltage/frequency, PV voltage/current (per-leg) | No dedicated slot beyond the power/SOC/grid rollup above — informational-only via direct query for now. |
| Energy accumulators (PV yield, consumption, import/export, charge/discharge — cumulative) | Doesn't map cleanly to `billing_daily/weekly/monthly` (those expect periodic-reset accumulators; EG4's energy sensors read lifetime-cumulative, same caveat as SPAN's Main Meter energy sensors). Wrap with HA's `utility_meter` helper if daily/monthly solar production tracking is wanted, then label *that* derived sensor. |
| Write-capable entities (switches/numbers/selects) | Out of scope by design — see Scope above, use the generic domain tools directly. |
| `Last Event` sensor (portal event log, cloud/hybrid modes) | No slot — informational/diagnostic, query directly if needed. |

---

## Setup Steps

1. **Install** `eg4_web_monitor` via HACS (custom repository —
   `https://github.com/joyfulhouse/eg4_web_monitor`).
2. **Configure**: Settings → Integrations → add "EG4 Web Monitor", authenticate against the
   EG4 cloud portal or local connection per the integration's own setup flow.
3. **Enumerate what got created:**
   ```
   zen_dojotools_inspect mode=integration_entities integration=eg4_web_monitor
   ```
4. **Auto-suggest labels** (device_class + name-pattern heuristics already know EG4's shape —
   `pv`/`solar` in the name → `zen_plant_pv_power`, `battery` + power device_class →
   `zen_plant_battery_power`, `battery` device_class → `zen_plant_battery_soc`, `grid` + power
   device_class → `zen_plant_grid_power`, `off_grid`/`offgrid` in a binary_sensor name →
   `zen_plant_off_grid`):
   ```
   zen_dojotools_plant mode=label_suggest integration=eg4_web_monitor
   ```
   Review the preview (multi-inverter installs: confirm every PV/battery/grid sensor you expect
   shows up as a candidate, not just the first one — v1's sum/average only works if every
   relevant entity actually gets tagged). Pass `confirm_action=true` to apply.
5. **Manual tag, if you'd rather do it by hand** (or `label_suggest` misses something — always
   possible on a new integration, verify names via step 3 first):
   ```
   zen_dojotools_labels mode=tag label_list=[zen_plant_pv_power] target_entities=[sensor.<inverter>_pv_power, ...]
   zen_dojotools_labels mode=tag label_list=[zen_plant_battery_power] target_entities=[sensor.<inverter>_battery_power, ...]
   zen_dojotools_labels mode=tag label_list=[zen_plant_battery_soc] target_entities=[sensor.<inverter>_battery_soc, ...]
   zen_dojotools_labels mode=tag label_list=[zen_plant_grid_power] target_entities=[sensor.<inverter>_grid_power, ...]
   zen_dojotools_labels mode=tag label_list=[zen_plant_off_grid] target_entities=[binary_sensor.<inverter>_off_grid]
   ```
6. **Validate:**
   ```
   zen_dojotools_plant mode=validate
   ```
   Confirms `pv_power`/`battery_power`/`battery_soc`/`grid_power` slots resolve, with
   `entity_count` per slot so you can see multi-inverter/multi-bank tagging actually landed on
   every device, not just one.
7. **Confirm the response:**
   ```
   zen_dojotools_plant mode=electric
   ```
   `entities.solar` should be present (not `null`) with real numbers. If it's `null` despite
   tagging, re-check step 6's `entity_count` fields — a label applied to the wrong domain
   (e.g. tagging a `number.*` settings entity instead of the `sensor.*` reading) silently
   won't resolve, same existence-checked-safe-no-op behavior as every other Plant slot.

---

## Related

- [Plant Manager](../components/plant_manager.md) — the consuming tool, full label reference and mode list
- [Plant Codex — SPAN Panel](span_panel_codex.md) / [Plant Codex — Emporia Vue](emporia_vue_codex.md) — companion Codexes for circuit-level panel monitoring, a different Plant Manager slot family from this one
