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
| `Breaker Rating` (A, diagnostic, v2) | `circuits`/`circuit_draw` draw-vs-capacity math | `sub_panel` — same tag as `Current`; `circuit_draw` (v5.9.0+) pairs each circuit's current sensor with its breaker_rating sibling by naming convention automatically, no separate label needed once both carry `sub_panel` |
| `tabs` attribute (breaker slot position) | `circuit_draw`'s pole-count (`poles: 1`/`2`) | Not a separate entity, just an attribute on the Power/Energy/Current sensors — `circuit_draw` reads it directly (two tab positions = 2-pole/240V), no ZenOS label needed |

### Trouble-signal support (Room Manager `trouble_active`)

SPAN qualifies. Room Manager's generic per-room `trouble_active` attribute (see [Plant Manager](../components/plant_manager.md#telegraphing-trouble-into-room-manager)) needs a provider that exposes real per-circuit current **and** a matching breaker rating — SPAN exposes both (`Current` + `Breaker Rating`, paired by naming convention), so any SPAN-covered room gets sustained->80%-of-rated-capacity trouble detection automatically, no tagging required. GFCI/AFCI protection type and special-purpose (surge suppression, EV charging, etc.) are separate, human-confirmed signals SPAN doesn't expose itself — tag the breaker's `switch` entity with `breaker_gfci`/`breaker_afci`/`surge_suppression` per [Plant Manager](../components/plant_manager.md).

### Not currently modeled (genuine gaps, not bugs)

| SPAN sensor | Gap |
|---|---|
| `Battery Power`, `PV Power`, BESS sensors (`Battery Level`, `Stored Energy`, `Nameplate Capacity`) | Plant Manager has no solar/battery slots at all today. If this household has PV or a BESS commissioned on the panel, this is real future scope — not something to force onto an existing `zen_plant_*` label. Worth a ticket if/when PV or battery hardware exists here. |
| `Binding Constraint`, `Import Limit`, `PCS Active` *(new, integration update — see below)* | Same PCS/BESS gap as the row above — these are Power Conversion System / inverter-side signals (grid-import limiting, active-constraint reporting) surfaced now that the integration exposes more of what the panel reports. No Plant Manager slot exists for PCS state today; same "worth a ticket if PV/battery hardware exists here" as `Battery Power`/`PV Power` above. Query-only via `zen_dojotools_inspect` for now. |
| EVSE sensors (`Charger Status`, `Advertised Current`, `Lock State`) | No `zen_plant_ev_charger` slot exists. If a SPAN Drive or other EVSE is commissioned, these are currently query-only via `zen_dojotools_inspect`, not surfaced through Plant Manager. |
| `Main Meter Consumed/Produced/Net Energy` (Wh, cumulative grid import/export, not periodic) | Doesn't cleanly fit `billing_daily`/`billing_weekly`/`billing_monthly` — those slots expect periodic-reset accumulators, and SPAN's meter energy sensors are lifetime-cumulative. If daily/monthly billing tracking is wanted, wrap these with HA's built-in `utility_meter` helper (creates a daily/monthly-reset derived sensor from any cumulative source) and label *that* derived sensor `zen_plant_billing_daily`/`_monthly`, not the raw SPAN sensor directly. |
| `Status Postal Code`, `Status Time Zone` | Panel physical-location metadata (config diagnostics), not operational data. Not a Plant Manager fit — nothing to label, nothing to model. Ships switched-off by the integration by default; leave them off unless you have a specific diagnostic reason to enable them. |

---

## Integration Update — eBus Data Model (flat → 1.0)

A firmware/integration update on this panel migrated its data model from the old flat format to eBus 1.0 (the integration reloads automatically when this happens — devices/entities realign to match what the panel now actually publishes). Two things worth knowing if you're re-checking labels after an update like this:

- **`DSM State` → `DSM Grid State`.** Same entity ID and history are preserved across the rename, so the existing `main_panel` label mapping above still holds with no action needed — but the *meaning* upgraded: on the old flat model this was inferred (from the battery when one was fitted, otherwise from dominant power source + whether power crossed the grid connection); on 1.0 it reads the real islanding state a Microgrid Interconnect Device (MID) senses directly. More trustworthy than before, same label.
- **`Grid Islandable`** also keeps working with no relabeling needed, but now reflects whether a MID is physically present (how backup capability is detected under 1.0) rather than a panel-level property the old model published directly.
- **New: Microgrid Interconnect Device, with its own `Grid State` entity** — reports the health of the utility supply itself, which the old flat model never exposed at all. This is a genuinely new signal, not a rename. It's panel-adjacent grid-health data, not a circuit — tag it `main_panel` alongside the other panel-level sensors in the table above if you want it covered by `zen_dojotools_plant mode=validate`; no dedicated Plant Manager slot exists for it yet, same "query-only for now" status as the PCS/BESS entities above.

If any of your existing SPAN entities show `unavailable` after an update like this, they were likely renamed/replaced rather than genuinely lost — per the integration's own advice, remove the stale unavailable ones manually rather than re-tagging them.

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
