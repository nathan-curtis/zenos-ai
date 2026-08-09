# ZenOS Plant Codex — SPAN Panel

**Integration:** [`SpanPanel/span`](https://github.com/SpanPanel/span) (HACS custom component)
**Consumes into:** `zen_dojotools_plant` (Plant Manager) — label-first discovery, no new code required
**Status:** This is the household's actual installed main panel — highest-priority Plant Codex

---

## Overview

SPAN Panel is a smart main electrical panel — not an add-on clamp monitor
like Emporia Vue, but the panel itself — providing circuit-level monitoring
*and* control (breaker on/off, load-shed priority) for the whole house.

Plant Manager's label vocabulary matches SPAN's own sensor naming almost
exactly (`l1_voltage`, `l2_voltage`, `main_breaker_rating`, upstream/
downstream L1/L2 current, `dsm_state`, `relay_state`) — strong evidence
Plant Manager's discovery waterfall was designed with a panel exactly like
this one in mind. **No new tool code is needed** — like the [Emporia Vue
Codex](emporia_vue_codex.md), this is the label-mapping guide, not a build.

> **Safety note, carried forward from the integration's own README:** this
> integration controls real relays. Circuit breaker switches and the
> load-shed priority control operate physical hardware with the same
> consequences as touching the panel directly. **Never wire an automation
> that flips a SPAN breaker or shed-priority control without an explicit
> human confirmation step** — same `confirm_action=true` discipline this
> system already applies everywhere else, but doubly so here since the
> blast radius is real household electrical circuits, not a tag or a
> label.

---

## Entity → Label Mapping

### Panel-level (exact-fit slots)

| SPAN sensor | Plant Manager slot | Label |
|---|---|---|
| `Current Power` | `site_power` | `zen_plant_site_power` (override) or `utility_main` |
| `Site Power` *(v2, PV/battery-aware total)* | `site_power` | Prefer this over `Current Power` if PV or battery is commissioned — it's the truer whole-home number (grid + PV + battery combined) |
| `L1 Voltage` | `l1_voltage` | `main_panel` — matches by name substring already, no override needed |
| `L2 Voltage` | `l2_voltage` | `main_panel` — same |
| `Main Breaker Rating` | `breaker_rating` | `main_panel` — SPAN's own sensor name literally contains `main_breaker_rating`, matches Plant's waterfall directly |
| `DSM State` | `panel_status.dsm_state` (in `electric` mode response) | `main_panel` |
| `Main Relay State` | `panel_status.relay_state` (in `electric` mode response) | `main_panel` |
| `Upstream L1/L2 Current`, `Downstream L1/L2 Current` | *(bus-level, informational only)* | `main_panel` — Plant's own `circuits sort_by=current` mode already explicitly excludes upstream/downstream/l1/l2 current and breaker_rating as bus-level, not circuit-level, so these need the label for `validate`/panel completeness but won't pollute the circuits list |

### Circuit-level (per breaker)

| SPAN sensor | Plant Manager slot | Label |
|---|---|---|
| `Consumed Energy` (Wh, cumulative) | `circuits`, `sort_by=energy` | `consumed_energy` — **exact unit match**, unlike Emporia's kWh (see Emporia Codex's unit caveat — doesn't apply here) |
| `Current` (A, v2 only, per circuit) | `circuits`, `sort_by=current` | `sub_panel` — exact fit, this is precisely the "raw amp reading" Plant's current-mode circuits list expects |
| `Power` (W, instantaneous) | *(no dedicated slot — see below)* | Not separately modeled by Plant Manager's `circuits` mode today; both `sort_by` options use accumulators (energy) or amps (current), not instantaneous watts per circuit. Available for direct query/dashboard use even if Plant Manager doesn't surface it in `circuits` output. |
| `Breaker Rating` (A, diagnostic, v2) | *(informational)* | No dedicated slot — circuit-level breaker rating isn't currently read by any Plant Manager mode |
| `tabs` attribute (breaker slot position) | *(informational)* | Not a separate entity, just an attribute on the Power/Energy sensors — useful context if manually cross-referencing a labeled circuit against the physical panel, no ZenOS label needed |

### Not currently modeled (genuine gaps, not bugs)

| SPAN sensor | Gap |
|---|---|
| `Battery Power`, `PV Power`, BESS sensors (`Battery Level`, `Stored Energy`, `Nameplate Capacity`) | Plant Manager has no solar/battery slots at all today. If this household has PV or a BESS commissioned on the panel, this is real future scope — not something to force onto an existing `zen_plant_*` label. Worth a ticket if/when PV or battery hardware exists here. |
| EVSE sensors (`Charger Status`, `Advertised Current`, `Lock State`) | No `zen_plant_ev_charger` slot exists. If a SPAN Drive or other EVSE is commissioned, these are currently query-only via `zen_dojotools_inspect`, not surfaced through Plant Manager. |
| `Main Meter Consumed/Produced/Net Energy` (Wh, cumulative grid import/export, not periodic) | Doesn't cleanly fit `billing_daily`/`billing_weekly`/`billing_monthly` — those slots expect periodic-reset accumulators, and SPAN's meter energy sensors are lifetime-cumulative. If daily/monthly billing tracking is wanted, wrap these with HA's built-in `utility_meter` helper (creates a daily/monthly-reset derived sensor from any cumulative source) and label *that* derived sensor `zen_plant_billing_daily`/`_monthly`, not the raw SPAN sensor directly. |

---

## Setup Steps

1. **Install** `span` via HACS (custom repository — `https://github.com/SpanPanel/span`).
2. **Configure**: Settings → Integrations → add "SPAN", enter panel IP, authenticate
   via passphrase (SPAN app → On-premise settings) or proof-of-proximity (open/close
   panel door 3×).
3. **Confirm firmware** is `spanos2/r202603/05` or later before upgrading past
   integration v1.1.x — check this first, the v2 breaking changes require it.
4. **Enumerate what got created:**
   ```
   zen_dojotools_inspect mode=integration_entities integration=span_panel
   ```
5. **Label the panel-level sensors** (exact names will include your panel's
   device name prefix — verify via step 4 before running these):
   ```
   zen_dojotools_labels mode=tag label_list=[main_panel] target_entities=[
     sensor.<current_power>, sensor.<l1_voltage>, sensor.<l2_voltage>,
     sensor.<main_breaker_rating>, sensor.<dsm_state>, sensor.<main_relay_state>,
     sensor.<upstream_l1_current>, sensor.<upstream_l2_current>,
     sensor.<downstream_l1_current>, sensor.<downstream_l2_current>
   ]
   zen_dojotools_labels mode=tag label_list=[zen_plant_site_power] target_entities=[sensor.<current_power_or_site_power>]
   ```
6. **Label every circuit's Consumed Energy and Current sensors:**
   ```
   zen_dojotools_labels mode=tag label_list=[consumed_energy] target_entities=[sensor.<circuit>_consumed_energy, ...]
   zen_dojotools_labels mode=tag label_list=[sub_panel] target_entities=[sensor.<circuit>_current, ...]
   ```
   Optionally add the matching `em_*` category label per circuit
   (`em_hvac`, `em_large_loads`, `em_spa`, `em_water`, `em_mechanical`,
   `em_power_critical`) alongside `consumed_energy`/`sub_panel` — same
   already-registered, currently-unused label set noted in the Emporia
   Vue Codex.
7. **Validate:**
   ```
   zen_dojotools_plant mode=validate
   ```
   Then confirm circuit-level coverage:
   ```
   zen_dojotools_plant mode=circuits sort_by=energy
   zen_dojotools_plant mode=circuits sort_by=current
   ```
   `sort_by=current` should show real per-circuit amps, not bus-level
   upstream/downstream/voltage/breaker_rating entries — if any of those
   leak into the circuits list, double-check they weren't accidentally
   tagged `sub_panel` instead of `main_panel`.

---

## Related

- [Plant Manager](../components/plant_manager.md) — the consuming tool, full label reference and mode list
- [Plant Codex — Emporia Vue](emporia_vue_codex.md) — companion Codex for the clamp-based add-on monitor; read its unit-mismatch caveat (kWh vs Wh) for contrast with SPAN's exact-unit fit
