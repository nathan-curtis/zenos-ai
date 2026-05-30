# ZenOS-AI SpaMaster

**Version:** 3.12.0
**Script:** `zen_dojotools_spamaster`

---

## Overview

Zen DojoTools SpaMaster is the ESPHome hot tub control layer for ZenOS-AI. It provides governed, label-driven control of spa hardware — jets, lights, audio, cover, temperature, and water chemistry — with scene-based mood management and post-soak logging.

Key capabilities:

* Live spa status with self-bootstrapping (no config required for first read)
* Jets, lights, audio, cover, and temperature control
* Water chemistry evaluation against stored or default targets
* Named mood scenes (orchestrates ZenLux + Media Manager)
* Post-soak log with chemistry evaluation, calendar entry, and supply reminders
* Model preset library for common Watkins Wellness / Caldera / Hot Spring hardware
* Salt system support (FreshWater IQ 2020, FreshWater ACE)

SpaMaster orchestrates — it does not own light or audio state. ZenLux owns lights; Media Manager owns audio. SpaMaster calls them.

---

## Migration from calderaspas Plugin

The old `zen_dojotools_spa_manager` script (from `plugins/calderaspas/`) is retired as of ZenOS 2026.6.0. It has been replaced by `zen_dojotools_spamaster` in `dojotools/dojotools_spa_manager.yaml`.

**Migration steps:**

1. Apply `spa_*` labels to your existing spa entities (see Label Taxonomy below)
2. Call `mode=discover` — verify all 8 entity roles are found
3. Call `mode=setup` — creates labels, tags entities, seeds `spa_config` in cabinet
4. Done. Old entity storage is ignored; labels are now authoritative.

The legacy `action` field is still accepted (normalizes to `mode`) for transitional support. No entity ID is ever stored in the cabinet — all resolution is done live from labels at call time.

---

## Hardware Requirements

### ESPHome Integration

SpaMaster requires an ESPHome-connected hot tub controller. Tested with Watkins Wellness / Caldera Spas / Hot Spring hardware on ESPHome CirquitPython integration.

Minimum required entity:

* One `climate.*` entity on the device (setup anchor for discovery)

Everything else is discovered from there or via area scan.

### Supported Models (Preset Library)

Presets are in `dojotools/.spa_presets/`. Load via `mode=setup model_preset=<name>`.

| Preset name | Brand | Series / Trim | Volume | Salt System |
|------------|-------|---------------|--------|-------------|
| `caldera_utopia_florence_2024` | Caldera Spas | Utopia / Florence | 360 gal | FreshWater IQ 2020 |
| `hot_spring_highlife_envoy_nxt_2024` | Hot Spring Spas | Highlife / Envoy NXT | 383 gal | FreshWater ACE |
| `watkins_iq_dealer_mps` | — | MPS / fairy dust protocol | — | FreshWater IQ (MPS mode) |

To add a new model: create `dojotools/.spa_presets/<model_name>.yaml` following the existing format, then reference it in `mode=setup model_preset=<model_name>`.

---

## Label Taxonomy

All labels use the `spa_` prefix. Apply these to your ESPHome entities via `mode=setup` or manually via `zen_dojotools_labels`.

| Label | Domain | Purpose |
|-------|--------|---------|
| `spa_climate` | `climate` | Thermostat / climate entity. **Required.** Setup anchor for device scan. |
| `spa_power` | `switch` | Main I-want-to-use-this-now on/off control. Not the breaker. |
| `spa_jets_l` | `switch` | Left / dual-speed jet pump. Variable-speed capable. |
| `spa_jets_r` | `switch` | Right / single-speed jet pump. |
| `spa_jets_all` | `group` | Jet group entity controlling both pumps. |
| `spa_lights` | `light` / `group` | Light group entity. Falls back to first `light.*` on device. |
| `spa_audio` | `media_player` / `switch` | Audio control entity. |
| `spa_cover` | `cover` | Motorized cover entity. Domain-validated (rejects `update.*`). |

Discovery cascade: spa_climate anchor → device entity fallback → area scan → `hot_tub` area label search.

---

## Setup Flow

### Step 1 — Discover (recommended)

```
mode=discover
```

Scans ESPHome device via climate anchor. Maps all 8 entity roles. Reports missing. Returns entity map and `next_step` advisory. **Read-only — writes nothing.**

`verbose=true` returns full device entity inventory grouped by domain.

### Step 2 — Setup

```
mode=setup
```

Creates `spa_*` label definitions, tags discovered entities, seeds `spa_config` in household cabinet.

Optional:
- `model_preset=<name>` — loads model dimensions, jet config, temperature defaults
- `chem_preset=<name>` — loads chemistry targets and chem system type
- `config_json=<json>` — raw override for any cabinet field

To wipe and start over: `mode=setup  reset=true  confirm=true` (preserves labels and entity tags).

### Step 3 — Verify

```
mode=status
```

Verifies all readings are live. If no cabinet config exists, discover runs inline and returns `setup_advisory`. The tool is always readable even before setup.

---

## Modes

| Mode | Description |
|------|-------------|
| `status` | Live snapshot: temperature, jets, lights, audio, cover, chemistry. Self-bootstraps via inline discovery if not configured. |
| `scene` | Concierge: survey capabilities, define/apply/delete named moods (see Scenes below). |
| `discover` | Scan device entity map, identify 8 role assignments, report missing. Does not write. |
| `setup` | Create labels, tag entities, seed spa_config. Safe to re-run. |
| `lights` | `light_state` (on/off), `color`, `zone`, `brightness` (0–100). |
| `jets` | `jets_state` (on/off), `zone` (all/left/right/waterfall), `speed` (low/high for variable-speed pumps). |
| `audio` | `audio_state` (on/off), `audio_volume` (0–100), `audio_source`. Also: `treble`, `bass`, `balance`, `subwoofer` (0–100 each, device-dependent). |
| `temperature` | `target_temperature` in °F (80–104). |
| `cover` | `cover_action`: `open` | `close`. |
| `chemistry` | Strip reading evaluation (`salt_ppm`, `free_chlorine_ppm`, `ph`, `alkalinity_ppm`, `hardness_ppm`) against stored targets. Returns guidance per parameter. |
| `log` | Post-soak wrap-up: chemistry eval, calendar entry, oxidizer dose calc, optional `close_cover=true`, `set_hold_temp`. Requires `soak_duration_hours` and `soak_persons`. |
| `consumables` | ERP surface: provision catalog from preset, track stock, add to shopping list, log replacements and purchases. See Consumables below. |
| `help` | `help_topic`: `index` | `entities` | `chemistry` | `actions` | `models` |

---

## Lights

| Field | Values |
|-------|--------|
| `light_state` | `on` / `off` |
| `color` | `Violet` \| `Blue` \| `Cyan` \| `Green` \| `White` \| `Yellow` \| `Red` \| `Cycle` |
| `zone` | `all` \| `underwater` \| `exterior` \| `waterfall` \| `topside` |
| `brightness` | 0–100 |

ZenLux owns light scenes. Use `mode=scene` to combine lighting with jets, audio, and cover in a named mood.

---

## Scenes

Named moods combine lights + jets + audio + cover + temperature.

SpaMaster orchestrates — ZenLux fires the light scene, Media Manager handles audio.

```
mode=scene  action=define  mood_name=romantic  scene_entity=scene.hot_tub_warm_glow
  jets_zone=all  jets_speed=low  audio_source=Spotify  audio_volume=35  open_cover=true
```

```
mode=scene  action=apply  mood_name=romantic
```

```
mode=scene  action=survey
```

`survey` returns discovered HA scenes in the hot tub area (from ZenLux) and stored mood definitions from cabinet.

Actions: `survey` | `define` | `apply` | `delete`

Moods stored in cabinet key `spa_scenes`.

---

## Chemistry

### System Types

| System | Description |
|--------|-------------|
| `saltwater` | FreshWater IQ 2020 (Caldera Spas) — Watkins recommended baseline |
| `saltwater_ace` | FreshWater ACE (Hot Spring) |
| `saltwater_iq` | FreshWater IQ with MPS / fairy dust dealer protocol |

### Profiles

**`watkins_recommended`** — manufacturer published targets

| Parameter | Range | Target |
|-----------|-------|--------|
| Salt | 1500–2000 ppm | 1750 ppm |
| Free chlorine | 1–5 ppm | 3 ppm |
| pH | 7.2–7.8 | 7.4 |
| Alkalinity | 40–120 ppm | 80 ppm |
| Calcium hardness | 25–75 ppm | 50 ppm |
| ORP minimum | 600 mV | — |

**`mps_mode`** — dealer / owner MPS (fairy dust) protocol

| Parameter | Range | Notes |
|-----------|-------|-------|
| Salt | 2200–2600 ppm | Dealer band, above Watkins published |
| MPS dose | 0.5 tsp / bather / 30 min | 40% MPS strength assumed |
| Sensor distrust window | 24 h post-MPS | TDS elevated; reading advisory returned |
| Chloramine warning | 0.4 ppm combined | |

### Correction Guidance

| Problem | Action |
|---------|--------|
| pH high | pH Down (sodium bisulfate) to skimmer; retest 30 min |
| pH low | pH Up (sodium carbonate) to skimmer; retest 30 min |
| Chlorine low | Check salt system output; granules to boost |
| Salt low | Add salt directly; 24 h to dissolve |
| Salt high | Verify panel indicator first; partial drain if confirmed |
| Alkalinity low | Sodium bicarbonate to skimmer; adjust pH after |
| Alkalinity high | Aerate with jets; pH Down lowers both |
| Hardness low | Calcium hardness increaser (foam/etching risk if untreated) |

**Adjustment sequence:** alkalinity first → pH → sanitizer.

---

## Post-Soak Log

`mode=log` is the structured end-of-soak routine.

Required fields:
- `soak_duration_hours`
- `soak_persons`

What it does:
1. Evaluates current chemistry against profile targets
2. Calculates MPS oxidizer dose (if mps_mode active)
3. Optionally: `boost_chlorine=true`, `close_cover=true`, `set_hold_temp=<°F>`
4. Creates HA calendar entry with soak summary
5. Adds supply todos if chemistry correction needed

---

## Consumables

`mode=consumables` is the ERP surface for spa supply tracking. It integrates with Grocy for inventory, shopping lists, and scheduled maintenance chores.

For the shared Grocy inventory contract used by SpaMaster and AutoVac, see [Grocy Inventory Component](plugins/grocy.md).

### Actions

| Action | Description |
|--------|-------------|
| `provision` | Builds parts + chem catalog from a model preset. Creates Grocy products, location, and scheduled chores. Idempotent — safe to re-run; existing chores found by name and reused. |
| `status` | Returns catalog with current stock levels, last-replaced dates, reorder flags. |
| `add_to_shopping` | Adds flagged items to a Grocy shopping list. |
| `log_replaced` | Records a part replacement to the Grocy chore log. Requires `part`. |
| `log_purchased` | Records a supply purchase (quantity received). Requires `part`. `amount` defaults to 1; decimals supported. |

### Fields

| Field | Description |
|-------|-------------|
| `action` | Sub-action: `status` \| `provision` \| `add_to_shopping` \| `log_replaced` \| `log_purchased` |
| `part` | Supply key from spa catalog (required for `log_replaced`, `log_purchased`) |
| `amount` | Quantity for `log_purchased` (default 1, decimals supported) |
| `force` | Re-run provision even if catalog already exists. Stock is preserved. |

### Provision

Provision links a spa model preset to Grocy ERP. Run once after `mode=setup`:

```
mode=consumables  action=provision  model_preset=caldera_utopia_florence_2024
```

What it does:
1. Creates a Grocy location matching the HA area name (skips if exists)
2. Creates products for each part in the preset (brand-generic names; duplicate-safe)
3. Creates scheduled chores for maintenance parts; on-demand chems get `chore_id: null` by design
4. Stores the full catalog in cabinet key `spa_consumables`

Re-provision with `force=true` rebuilds the catalog entry; existing Grocy chores are found by name and reused rather than duplicated.

---

## Cross-Tool Awareness

When `spa_climate` is detected in any area by other tools, they surface a `spa_master` pointer in their response:

* **Room Manager** (`domain_routing.spa`): `zen_dojotools_spamaster — all spa intents: chemistry, jets, cover, lights, audio, temperature. area=<id>`
* **ZenLux**: surfaces `spa_master` note when spa climate entity detected
* **Climate Manager**: surfaces `spa_master` note
* **Camera**: surfaces `spa_master` note in area context

No explicit config needed — detection is automatic via `spa_climate` label.
