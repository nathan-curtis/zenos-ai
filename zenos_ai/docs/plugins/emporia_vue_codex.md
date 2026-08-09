# ZenOS Plant Codex — Emporia Vue

**Integration:** [`magico13/ha-emporia-vue`](https://github.com/magico13/ha-emporia-vue) (HACS custom component)
**Consumes into:** `zen_dojotools_plant` (Plant Manager) — label-first discovery, no new code required
**Status:** First Plant Codex — reference for wiring any panel-level energy monitor into Plant Manager's existing waterfall

---

## Overview

Emporia Vue is a clamp-based panel/circuit energy monitor. The HA integration
(`ha-emporia-vue`) reads the Emporia cloud API and creates a **Power** (W) +
**Energy** (kWh) sensor pair for every monitored channel — the main panel
feed itself, plus each individual breaker channel you've clamped.

Plant Manager (`zen_dojotools_plant`) already has a label-first discovery
waterfall built to consume exactly this shape of data (see
[Plant Manager](../components/plant_manager.md#discovery-waterfall) and
[`circuits` mode](../components/plant_manager.md#modecircuits--circuit-breakdown)).
**No new tool code is needed** — this codex is purely the mapping guide from
Emporia's entity shape onto ZenOS's existing label taxonomy, so it's a label
job, not a build.

---

## What Emporia Vue Creates

Per monitored channel, at whatever scale(s) you configure in the integration
setup (`1MIN`, `1D`, `1MON`, etc — multiple scales can coexist per channel):

| Entity | `device_class` | `state_class` | Unit | Name pattern |
|---|---|---|---|---|
| Power sensor | `power` | `measurement` | W | `Power {scale}` (e.g. "Power 1M") |
| Energy sensor | `energy` | `total` | **kWh** | `Energy {scale}` (e.g. "Energy 1D") |

Each channel is its own HA *device*, named whatever you called it during
Emporia app setup (e.g. "Main", "HVAC", "Kitchen", "EV Charger", "Hot Tub").
With `has_entity_name` on, HA slugifies device name + sensor name into the
final `entity_id` — expect something like `sensor.hvac_power_1min` /
`sensor.hvac_energy_1d`, but **always verify via `zen_dojotools_inspect
mode=integration_entities integration=emporia_vue`** rather than guessing —
Emporia app channel names are user-chosen and will vary household to
household.

**Unit caveat:** Emporia's Energy sensor is **kWh**, not Wh. Plant Manager's
`consumed_energy` label description says "sorted by lifetime Wh" — read that
as "cumulative energy, unit-aware," not literally raw watt-hours. If you
apply `consumed_energy` to an Emporia kWh sensor, sanity-check the resulting
`circuits` mode ordering makes sense (it should — sort order is unaffected by
a consistent unit scale factor across all channels) rather than assuming the
displayed magnitude is watt-hours.

**No true "lifetime" scale exists.** Emporia's largest scale is `1Y`
(yearly), not a true all-time cumulative counter. For Plant Manager's
"lifetime Wh" framing, `1MON` or `1Y` scale Energy sensors are the closest
practical fit — pick one scale per channel and label it consistently, don't
mix scales across channels feeding the same `circuits` mode call.

---

## Mapping Onto Plant Manager Labels

| Emporia channel | Plant Manager label | Slot / mode |
|---|---|---|
| Main panel feed (Power sensor) | `zen_plant_site_power` (override) or `utility_main` | `electric` → `live_power_w` |
| Main panel feed (Energy sensor, daily scale) | `zen_plant_billing_daily` or `utility_billing` | `electric` → `daily_kwh` |
| Main panel feed (Energy sensor, monthly scale) | `zen_plant_billing_monthly` or `utility_billing` | `electric` → `monthly_kwh` |
| Any individual circuit channel (Energy sensor) | `consumed_energy` | `circuits` mode, `sort_by=energy` |
| Any individual circuit channel (Power sensor) | *(no direct current/amps equivalent — see caveat below)* | — |

**`sort_by=current` doesn't fit Emporia directly.** Plant's `circuits
sort_by=current` mode expects raw amp readings (`sub_panel` label, `*_current`
suffix) so it can estimate power via `amps × voltage`. Emporia gives you
power/energy directly, not amps — use `sort_by=energy` (the default) for
Emporia-fed circuits instead of trying to force a current-mode fit.

### Optional: circuit-category labels

ZenOS already has an `em_*` label set (`em_hvac`, `em_large_loads`,
`em_spa`, `em_water`, `em_mechanical`, `em_power_critical`, `em_electrical`,
`em_panel`, `em_breaker_slot`) pre-existing in the label vocabulary,
currently unused by any tool logic. These aren't consumed by Plant Manager's
own modes today, but are a natural, already-registered place to additionally
tag individual Emporia channels for finer-grained circuit categorization
than the generic `circuits` list gives you (e.g. "which channels are
`em_hvac`" as a manual/future query) — apply them alongside `consumed_energy`,
not instead of it.

---

## Setup Steps

1. **Install** `ha-emporia-vue` via HACS (custom repository —
   `https://github.com/magico13/ha-emporia-vue`, category `Integration`).
2. **Configure** via Settings → Integrations → add "Emporia Vue", sign in
   with your Emporia account (see the integration's own README for the
   Google/Apple token workaround if you don't have a password login).
3. **Pick scales deliberately.** Configure at minimum: one scale on the main
   panel feed for live power (`1MIN` or `1S`), one daily-scale Energy sensor
   on the main feed for `billing_daily`, and a consistent single scale
   (recommend `1MON`) across every circuit channel's Energy sensor so
   `circuits` mode comparisons stay apples-to-apples.
4. **Enumerate what got created:**
   ```
   zen_dojotools_inspect mode=integration_entities integration=emporia_vue
   ```
5. **Label the main feed:**
   ```
   zen_dojotools_labels mode=tag label_list=[zen_plant_site_power] target_entities=[sensor.<main_power_entity>]
   zen_dojotools_labels mode=tag label_list=[zen_plant_billing_daily] target_entities=[sensor.<main_daily_energy_entity>]
   ```
6. **Label each circuit channel's Energy sensor** with `consumed_energy` (and
   optionally the matching `em_*` category label):
   ```
   zen_dojotools_labels mode=tag label_list=[consumed_energy, em_hvac] target_entities=[sensor.<hvac_energy_entity>]
   ```
7. **Validate:**
   ```
   zen_dojotools_plant mode=validate
   ```
   Confirms `site_power` and billing slots resolved. `consumed_energy`-tagged
   circuit channels aren't individual named slots — verify them instead via:
   ```
   zen_dojotools_plant mode=circuits sort_by=energy
   ```
   and confirm every labeled channel appears with a sane ordering.

---

## Related

- [Plant Manager](../components/plant_manager.md) — the consuming tool, full label reference and mode list
- [Firefly III — Depreciation Codex](depreciation.md) — the prior Codex-tier reference this doc's structure follows
