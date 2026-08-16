# ZenOS-AI ZenShade — Cover Manager

**Version:** 6.0.0
**Script:** `zen_dojotools_covers`
**Codename:** ZenShade

---

## Overview

ZenShade is the room-aware cover and blind control layer for ZenOS-AI. It provides governed, label-driven control of motorized blinds, shades, curtains, and vents — with tilt support, barrier exclusion, and dynamic light transmission calculation.

Key capabilities:

* Label-based role assignment — no hardcoded entity IDs
* Tilt support (bit-128 detection, auto-tilt per scene)
* Barrier exclusion — garage doors, entry doors, and gates are never included in window scenes
* Dynamic `light_tx` — actual transmission calculated from cover position (or live lux sensor)
* Privacy scene advisory — warns when open sensors detected
* ZenLux integration — called by `bleed_set` when `sync_shades=true`
* Identity-gated actuation: `cover_control` certification required on `discover`/`set`, asymmetric by risk (see Identity Gate below)
* Full SESE (single-entry/single-exit) + canonical `envelope()` response shape (2026-08-15)

ZenShade is stateless — no cabinet, no preferences stored.

---

## First-Time Setup

### Step 1 — Create label definitions

```
mode=setup
```

Creates 5 `zen_cv_*` label definitions in HA (teal, `mdi:blinds` icon). Idempotent.

### Step 2 — Apply labels

Tag each cover entity via HA UI or `zen_dojotools_labels`:

```
zen_dojotools_labels  action_type=tag  target_entities=cover.living_room_blind  label_list=zen_cv_blackout
```

### Step 3 — Verify

```
mode=discover  room=<room_name>
```

Confirms roles assigned, `is_barrier` correct, `effective_light_tx` calculated.

**Optional:** Set up an outdoor lux sensor with label `zen_outdoor_lux` for sensor-direct light_tx method (more accurate than position-estimate).

---

## Label Taxonomy

| Label | Cover Type | Base `light_tx` |
|-------|-----------|----------------|
| `zen_cv_blackout` | Blackout blind or shade | 0.02 |
| `zen_cv_sheer` | Sheer / solar-diffuser shade | 0.50 |
| `zen_cv_curtain` | Motorized curtain | 0.45 |
| `zen_cv_vent` | Smart HVAC vent (Flair, Keen). **Excluded** from all window scenes unless `include_vents=true`. | n/a |
| `zen_cv_primary` | Primary cover surface for the room. Wins tiebreaks. | — |

**Barrier detection** (no label needed): covers with `device_class` in `[garage, door, gate]` are automatically flagged `is_barrier=true` and excluded from all `scene_set` calls. Direct `set` with explicit `entity_id` still works.

**Outdoor lux reference (optional):** Apply `zen_outdoor_lux` to any outdoor illuminance sensor. Enables sensor-direct `light_tx` calculation.

---

## Modes

| Mode | Description |
|------|-------------|
| `discover` | Map all covers in a room. Returns per-cover: `position`, `tilt_supported`, `tilt_position`, `role`, `effective_light_tx`, `light_tx_method`, `device_class`, `is_barrier`, `is_primary`. |
| `set` | Set `position` (0–100) and/or `tilt_position` (0–100) for a specific cover. Either or both valid. `0=close`, `100=open`. Resolves via `room` + `target_role` or direct `entity_id`. Supports `dry_run=true`. Identity-gated — see below. |
| `scene_set` | Apply a named scene to all covers in a room (see Scenes). Barriers always excluded. Vents excluded unless `include_vents=true`. |
| `setup` | Creates 5 `zen_cv_*` label definitions. Idempotent. |
| `help` | Full schema: modes, label taxonomy, scene mappings, tilt details, light_tx methods, setup flow. |

---

## Scenes

`scene_set` applies a named scene to all non-barrier covers in a room. Tilt is applied automatically for covers that support it (bit-128 detection).

| Scene | Position | Tilt (tilt-capable) | Notes |
|-------|----------|---------------------|-------|
| `movie` | 0% (closed) | 0° (slats closed) | Blackout for video |
| `sleep` | 0% (closed) | 0° (slats closed) | Dark room |
| `privacy` | 50% | 45° (diffuse, no sightline) | Advisory fires if open doors/windows detected |
| `work` | 100% (open) | 100° (slats open) | Max light, productive |
| `morning` | 100% (open) | 100° (slats open) | Same as work |
| `natural` | 100% (open) | 100° (slats open) | Full natural light |

**Privacy advisory:** `scene=privacy` checks binary sensors in the area for open doors/windows (`device_class` in `[door, window, opening]`). Returns `privacy_advisory` with `open_sensors_in_area` list if any are open.

---

## Tilt Support

Tilt is auto-detected from `state_attr(entity, 'supported_features')` bit 7 (value 128). No label or configuration needed.

* `tilt_position: 0` = horizontal / slats closed
* `tilt_position: 100` = vertical / slats fully open
* `set` mode: pass `tilt_position=0–100` alongside or instead of `position`
* `scene_set` mode: tilt applied automatically per scene table above for any tilt-capable cover

---

## Light Transmission (`light_tx`)

Each cover's current optical transmission is reported in `discover` responses and used by ZenLux bleed propagation.

**Sensor-direct method** (preferred — requires `zen_outdoor_lux` sensor):
```
effective_light_tx = sensor_lux / outdoor_lux   (clamped 0–1)
```

**Position-estimate method** (fallback):
```
effective_light_tx = base_light_tx[role] × (position / 100)
```

Vent covers (`zen_cv_vent`) always return `light_tx = n/a`.

`light_tx_method` field in discover response indicates which method was used.

---

## Identity Gate (2026-08-15/16)

`discover` and `set` target resolution runs through the shared `zen_target_resolve.jinja` core (the same shared targeting macros `zen_dojotools_locks` uses — ZenShade was the other tool it was originally extracted from). `scene_set` keeps its own `area_entities` walk — it applies per-role scene positions across a whole area rather than resolving a caller-specified target set, so it isn't a targeting-core candidate.

`mode=set` requires the `cover_control` certification, asymmetric by risk:

- **Closing** any cover, and **opening** a non-barrier cover (blind/shade/curtain), is cert-only.
- **Opening** a barrier-class cover (`device_class` in `garage`/`door`/`gate`) requires the cert **and** a fresh live household-admin acknowledgment every single call — exact parallel to locks' exterior-unlock gate and security_manager's disarm gate.

An admin can exempt specific barrier targets from the every-call ack via `zen_dojotools_persona_editor mode=cert_grant cert_component=cover_control cert_scope=["cover.garage_door"]`. A mixed-coverage call (some targets scoped, some not) still asks for the whole set — no partial silent authorization. Scope resolution is centralized (2026-08-16): this tool passes `target_entity=<cover entity_id>` on `zen_dojotools_identity mode=resolve_caller_identity` and reads `scope_decision` back from the shared `cert_scope_check` macro, rather than testing `cert_scope` locally.

`cover_control` is self-published in this tool's own `tool_manifest.certs_required` — see `zen_dojotools_profile_readme.md`'s certification section for how that feeds the live cert catalog, and the [Security & Certification System operator manual](../getting_started/security_certification_manual.md) for the full model.

**Real bug found and fixed building the barrier gate:** `position`/`tilt_position` used Jinja's `default(-1, true)`, which substitutes on *any* falsy value, not just an undefined one — so `mode=set position=0` (a legitimate "close the cover" call) was silently coerced to the "no position given" sentinel for every caller, inherited from the v5.1.0 baseline. Fixed with explicit `is not defined`/`is none` checks. A follow-up `dry_run` bug was also found and fixed: the preview reported `would_require_live_ack: true` for a barrier cover even after a `cert_scope` exemption was granted for it, because the identity/scope resolve only ran in the actuation branch, after `dry_run` had already returned. `dry_run` output now includes `barrier_uncovered_by_scope` and `would_be_authorized` alongside the corrected `would_require_live_ack`.

---

## ZenLux Integration

ZenShade is called by `zen_dojotools_lights bleed_set` when `sync_shades=true`:

* ZenLux fires its light bleed to adjacent rooms, then calls `zen_dojotools_covers scene_set` for the primary room
* Intent mapping applied automatically: `movie→movie`, `work→work`, `morning→morning`, `sleep→sleep`, `party→natural`, `away→sleep`
* `bleed_set` response includes ZenShade result: `status`, `scene`, `applied_count`, `tilt_applied_count`

ZenShade has no return callback to ZenLux.

---

## Version History

| Version | Change |
|---------|--------|
| 6.0.0 (2026-08-15) | Identity-gate "cadillac" pass matching locks/security_manager. Migrated `discover`/`set` onto the shared `zen_target_resolve.jinja` core. Full SESE + `envelope()` conversion. New `cover_control` cert, asymmetric by barrier risk. `cert_scope` per-target ack override, centralized onto the shared `cert_scope_check` resolver on 2026-08-16. Fixed the `position=0` coercion bug and a `dry_run`/`cert_scope` blind-spot bug, both found live while building the gate. |
| 5.1.0 | Baseline — label taxonomy, tilt support, barrier auto-exclusion, `light_tx` calculation, ZenLux sync. |
