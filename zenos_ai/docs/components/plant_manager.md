# ZenOS-AI Plant Manager

**Version:** 5.3.0
**Script:** `zen_dojotools_plant`

---

## Overview

Physical Plant + Energy Manager. Surfaces live state for all major utilities — electric, HVAC, water, gas, and mechanical systems — via a label-first discovery model.

Key capabilities:

- Live whole-home power, daily/monthly billing, grid carbon intensity
- Water usage and billing rate
- Gas consumption (if sensor labeled)
- HVAC climate units snapshot
- Mechanical: water heater + sump pump + motors
- Inventory attachment: optional Grocy `room_brief` per load area (`include_inventory=true`)
- Circuit-level breakdown (by lifetime Wh or live amps)
- Slot validation report — identify what's wired, what needs a label
- Utility registry injected from Room Manager household cabinet
- `zen_plant_*` label overrides to pin any sensor to any slot

All sections return `available: false` when entities are missing or unavailable. No halts, no exceptions.

---

## Modes

| Mode | Description |
|------|-------------|
| `overview` | Top-line snapshot: electric, hvac, thermal, water, gas, mechanical, utility registry |
| `electric` | Full electrical: live watts, billing (daily/weekly/monthly), tariff, grid carbon, panel status |
| `hvac` | Climate units: mode, setpoint, current temp. Gas utility contact from utility_index. |
| `water` | Water usage sensor + billing rate + `usage_since` timestamp |
| `gas` | Live gas sensor if labeled, else graceful N/A + utility contact |
| `mechanical` | Water heater + sump pump + motors + water_management subnodes (softener, auto-shutoff, leak sensors). Optional `include_inventory=true` calls Grocy `room_brief` for each load area. |
| `thermal` | Thermal-managed loads distinct from HVAC — hot tub, freezers, generic thermal. Not room air. |
| `circuits` | Circuit breakdown. Params: `circuit_limit` (default 10), `sort_by` (`energy`\|`current`) |
| `managed` | All Grocy-provisioned machines — chores due, stock summary, products grouped by `ha_labels` root. Any machine bootstrapped via `provision_bom` appears automatically. `managed_labels` scopes to specific machines (CSV). |
| `validate` | Slot resolution report — entity_id, pinned, raw_state, ok |
| `ignore` | Tag entity with `zen_plant_ignore` (creates label if missing). Param: `target_entity`. |
| `unignore` | Remove `zen_plant_ignore` from entity. Param: `target_entity`. |
| `help` | Full discovery reference (returned inline, no docs needed) |

---

## Discovery Waterfall

Each slot resolves in priority order — first match wins. Apply `zen_plant_*` labels to pin any sensor directly.

| Slot | Waterfall |
|------|-----------|
| `site_power` | `zen_plant_site_power` → `label:utility_main` *power\|watt* (excl. produced/reactive/pv) → `label:main_panel` *site_power* |
| `tariff` | `zen_plant_tariff` → `label:utility_billing` *tariff\|rate\|price* → `label:utility_meter` *tariff\|rate\|price* |
| `billing_daily` | `zen_plant_billing_daily` → `label:utility_billing` *daily* → `label:utility_meter` *daily* |
| `billing_weekly` | `zen_plant_billing_weekly` → `label:utility_billing` *weekly* → `label:utility_meter` *weekly* |
| `billing_monthly` | `zen_plant_billing_monthly` → `label:utility_billing` *monthly* → `label:utility_meter` *monthly* |
| `grid_carbon` | `zen_plant_grid_carbon` → `label:energy_management` *fossil* |
| `water` | `zen_plant_water` → `label:water_usage` (excl. fee/rate/cost) → `label:droplet` (excl. fee/rate/cost) |
| `water_rate` | `zen_plant_water_rate` → `label:water_usage` *fee\|rate\|cost* → `none` if unlabeled |
| `gas` | `zen_plant_gas` only — no generic fallback |
| `l1_voltage` | `label:main_panel` *l1_voltage* |
| `l2_voltage` | `label:main_panel` *l2_voltage* |
| `breaker_rating` | `label:main_panel` *main_breaker_rating* |
| `hvac` | `label:main_thermostat` (preferred) → `label:hvac` (fallback) — climate.* domain |
| `water_heater` | `label:water_heater` — water_heater.* + sensor.*available* + binary_sensor.*running* + sensor.*power* |
| `sump_pump` | `label:sump_pump` — binary_sensor.* |

**`zen_plant_ignore`** — tag any entity to suppress it from all waterfalls.

---

## zen_plant_* Override Labels

Pin a sensor to any slot by applying the matching label. Overrides always take first priority.

| Label | Slot | Notes |
|-------|------|-------|
| `zen_plant_site_power` | Whole-home live watts | sensor.* |
| `zen_plant_tariff` | Billing tariff | Unit must be USD/Wh — converted to USD/kWh for display |
| `zen_plant_billing_daily` | Daily billing accumulator | sensor.* |
| `zen_plant_billing_weekly` | Weekly billing accumulator | sensor.* |
| `zen_plant_billing_monthly` | Monthly billing accumulator | sensor.* |
| `zen_plant_grid_carbon` | Grid fossil fuel % | sensor.* |
| `zen_plant_water` | Whole-home water flow/usage | sensor.* |
| `zen_plant_water_rate` | Water billing rate | sensor.* |
| `zen_plant_gas` | Gas consumption (therms) | sensor.* |
| `zen_plant_ignore` | Suppress from all waterfalls | Any domain |
| `zen_plant_hot_tub` | Thermal: hot tub setpoint + temp | `climate.*` (setpoint+temp) or `sensor.*temp*` (read-only) |
| `zen_plant_freezer` | Thermal: freezer temp (one node per entity) | `sensor.*` |
| `zen_plant_thermal` | Thermal: generic thermal-managed load | Any — surfaced as `thermal_load` nodes |
| `zen_plant_motor` | Mechanical: motor entity (cover, fan, binary_sensor, etc.) — surfaces in `motors[]` list | Any domain |
| `zen_plant_water_softener` | Water management: softener/conditioner sensors | `sensor.*`, `binary_sensor.*` |
| `zen_plant_auto_shutoff` | Water management: auto shutoff valve | `binary_sensor.*` or `switch.*` |
| `zen_plant_leak_sensor` | Water management: leak detection probes | `binary_sensor.*` |

---

## Standard Labels

Primary discovery path when `zen_plant_*` overrides are absent.

| Label | Used by | Matches |
|-------|---------|---------|
| `utility_main` | Electric `site_power` | sensor.* power/watt (excl. produced/reactive/pv) |
| `utility_meter` | Billing | sensor.* daily/weekly/monthly, tariff/rate/price |
| `consumed_energy` | Circuits (energy mode) | sensor.* sorted by lifetime Wh |
| `main_panel` | Electric, circuits | sensor.* site_power, voltage, breaker, current |
| `sub_panel` | Circuits (current mode) | sensor.* current |
| `water_usage` | Water | sensor.* (non-fee); fee/rate/cost → water_rate slot |
| `droplet` | Water fallback | sensor.* (Droplet integration) |
| `energy_management` | Grid carbon | sensor.* fossil |
| `water_heater` | Mechanical | water_heater.*, sensor.*available*, binary_sensor.*running*, sensor.*power* |
| `sump_pump` | Mechanical | binary_sensor.* |
| `main_thermostat` | HVAC (preferred) | climate.* |
| `hvac` | HVAC (fallback) | climate.* |
| `utility_billing` | Electric billing, tariff | Integration sensors for daily/weekly/monthly kWh, tariff/rate, peak status |

---

## mode=validate — Slot Resolution Report

Identifies what Plant Manager resolved (or did not) for each of the 15 slots.

```
zen_dojotools_plant  mode=validate
```

Response `slots{}` — one entry per slot:

| Field | Notes |
|-------|-------|
| `entity_id` | Resolved entity, or empty string |
| `pinned` | `true` if resolved via `zen_plant_*` label override |
| `raw_state` | Current state value, or `null` if no entity |
| `ok` | `true` if entity resolved and state is readable |

Summary includes `resolved`, `unresolved`, `total`, `unresolved_slots[]`, and a tip.

Run validate after setup. Any unresolved slot that matters should get a `zen_plant_*` label applied to the correct entity.

---

## mode=circuits — Circuit Breakdown

```
zen_dojotools_plant  mode=circuits  sort_by=energy  circuit_limit=10
```

| Param | Default | Notes |
|-------|---------|-------|
| `sort_by` | `energy` | `energy` = lifetime Wh (`label:consumed_energy`), `current` = live amps (`label:main_panel` + `sub_panel` *_current) |
| `circuit_limit` | 10 | Max circuits returned (1–50) |

`sort_by=energy` returns circuits sorted descending by cumulative lifetime energy. Best for identifying top consumers.

`sort_by=current` returns circuits sorted by live amps with estimated power (`est_power_w = amps × voltage`). Bus-level sensors (upstream/downstream, l1/l2 current, breaker_rating) are excluded.

---

## Utility Registry

On every response, `utilities` (overview, electric) or `utility` (hvac, gas, water, mechanical) is injected from the Room Manager household cabinet `utility_index` drawer.

Structure: `electric{}`, `gas{}`, `water{}` — each with provider name, emergency phone, cutoff location, and service entry location.

Write utility_index via `zen_dojotools_room_manager mode=utility utility_action=set`.

---

## Response Field Reference

### overview

| Field | Content |
|-------|---------|
| `electric{}` | `available`, `live_power_w`, `live_power_kw`, `daily_kwh`, `weekly_kwh`, `monthly_kwh`, `tariff_usd_kwh`, `peak_billing_month`, `grid_fossil_pct`, `l1_voltage`, `l2_voltage`, `main_breaker_amps`, `source` |
| `hvac{}` | `available`, `units[]`, `count` |
| `water{}` | `available`, `usage_gal`, `rate_per_1000gal_usd`, `source` |
| `gas{}` | `available`, `live_consumption_therms`, `note`, `source` |
| `mechanical{}` | `available`, `water_heater{}`, `sump_pump{}`, `motors[]`, `water_management{}` |
| `utilities` | Full utility_index from household cabinet |

### electric

Adds `panel_status{}` (`dsm_state`, `relay_state`) and `utility{}` (electric entry from utility_index).

### water

Adds `usage_since` — sensor `last_reset` attribute as ISO timestamp, or `null`.

### mechanical — water_heater{}

| Field | Content |
|-------|---------|
| `entity_id` | water_heater.* entity |
| `mode` | HA water heater mode string |
| `current_temp_f` | Current water temperature |
| `setpoint_f` | Target temperature |
| `available_hot_water_pct` | Hot water availability % |
| `running` | bool — heating element active |
| `live_power_w` | Current draw in watts |
| `daily_kwh` | Today's energy use in kWh |

### mechanical — sump_pump{}

| Field | Content |
|-------|---------|
| `entity_id` | binary_sensor.* entity |
| `active` | bool — pump running (`on` state) |

### mechanical — motors[]

Populated from entities labeled `zen_plant_motor`. Each entry: `{entity_id, name, domain, area, state}`. Empty list if no entities labeled. `available` flag on the mechanical block reflects motor presence.

### mechanical — water_management{}

Present when any `zen_plant_water_softener`, `zen_plant_auto_shutoff`, or `zen_plant_leak_sensor` entity is labeled. Covers the mechanical water infrastructure outside the main distribution path.

| Subnode | Content |
|---------|---------|
| `water_softener` | List of softener/conditioner sensor nodes — `entity_id`, `name`, `state`. `null` if none labeled. |
| `auto_shutoff` | `{entity_id, state}` for the shutoff valve. State `on` = cutoff engaged (normally-open convention). `null` if none labeled. |
| `leak_sensors` | List of leak probe nodes — `entity_id`, `name`, `state`. `null` if none labeled. |

Grocy chores for water management maintenance are accessible via `chores_by_area homeassistant_area_id=<area>`.

### mechanical — include_inventory

Pass `include_inventory: true` to attach Grocy `room_brief` output to the water heater and hot tub nodes. Calls are made per unique HA area (one call if both share an area, two if separate). Adds `inventory` key to each load node. Requires Grocy to be configured and reachable.

### mode=thermal — Response

Returns `thermal` block — thermal-managed loads distinct from HVAC (not room air).

| Key | Content |
|-----|---------|
| `hot_tub` | `{entity_id, name, setpoint, current_temp, area}` if `zen_plant_hot_tub` labeled, else `{available: false, note: ...}` |
| `freezers` | List of freezer nodes `{entity_id, name, temp}` from `zen_plant_freezer` entities |
| `other_loads` | List of `{type: thermal_load, entity_id, name, state}` from `zen_plant_thermal` entities |

### mode=ignore / mode=unignore

```
zen_dojotools_plant  mode=ignore   target_entity=binary_sensor.garage_lights_sump_pump_active
zen_dojotools_plant  mode=unignore target_entity=binary_sensor.garage_lights_sump_pump_active
```

`mode=ignore` creates the `zen_plant_ignore` label if it doesn't exist, then tags the entity in one call. `mode=unignore` removes the tag. Use for ghost entities or any sensor that should never appear in plant discovery.

---

## home_overview Integration

Room Manager `mode=home_overview` includes a `plant{}` snapshot derived from the same waterfall as `zen_dojotools_plant`. It is summary-only — for depth, call Plant Manager directly.

| Field | Content |
|-------|---------|
| `live_kw` | Whole-home live draw in kW (null if no entity resolved) |
| `water_gal` | Current water reading in gallons (null if no entity resolved) |
| `detail` | `"zen_dojotools_plant — electric | water | hvac | gas | circuits | validate"` |

Entities tagged `zen_rm_ignore` are excluded from the home_overview plant snapshot.
