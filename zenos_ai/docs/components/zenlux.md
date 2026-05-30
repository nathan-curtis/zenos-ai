# ZenOS-AI ZenLux — Lighting Manager

**Version:** 0.6.0
**Script:** `zen_dojotools_lights`
**Codename:** ZenLux

---

## Overview

ZenLux is the room-aware lighting control layer for ZenOS-AI. It provides governed, label-driven light control with memory (preferences), topology propagation (bleed to adjacent rooms), and Room Manager integration (hold gate, occupancy awareness).

Key capabilities:

* Label-based entity resolution — no hardcoded entity IDs
* Topology-aware bleed: propagates scenes to adjacent rooms via Room Manager portal transmission values
* RM hold gate: hard-blocks hardware writes when room is in Hold or Pause state
* Per-room lighting preferences stored in household cabinet, re-resolved live at apply time
* ZenShade sync: optional cover scene applied alongside light bleed
* Discovery and label suggestion for initial setup

---

## First-Time Setup

### Step 1 — Create label definitions

```
mode=setup
```

Creates 8 `zen_lm_*` label definitions in HA. Idempotent — safe to re-run.

### Step 2 — Discover

```
mode=discover
```

No `room` = whole-home overview. With `room` = full role map for that room with state, capabilities, scene inventory, RM status.

### Step 3 — Suggest labels

```
mode=label_suggest  room=<room_name>
```

Analyzes entity names and capabilities. Returns `zen_lm_*` role proposals with confidence. Does not apply — review and apply via `zen_dojotools_labels`.

### Step 4 — Apply labels

Tag entities via `zen_dojotools_labels action_type=tag`.

### Step 5 — Verify

```
mode=discover  room=<room_name>
```

Confirm all role slots resolve and advisory is clear.

**Optional:** `mode=auto_label  room=<room_name>  dry_run=true` — auto-tags unlabeled area lights with the room label.

---

## Label Taxonomy

### Hardware Role Labels (`zen_lm_*` — tool-owned)

| Label | Device Type | Auto-Resolution Priority |
|-------|-------------|--------------------------|
| `zen_lm_group` | Light group entity | 1st (highest) |
| `zen_lm_main` | Primary ambient / overhead | 2nd |
| `zen_lm_accent` | Accent / decorative (LED strips, bias lighting) | — |
| `zen_lm_task` | Task lighting (under-cabinet, desk lamp, reading) | — |
| `zen_lm_night` | Night / safety lighting (low-level, always reachable) | — |
| `zen_lm_outdoor` | Exterior / perimeter (porch, pathway, security) | — |
| `zen_lm_dusk_dawn` | Photosensor / auto-managed exterior. Tool warns before override. | — |
| `zen_lm_scene` | HA scene entity | — |

**Legacy fallback labels** (when `zen_lm_*` not present): `lighting` / `main_lighting` → main, `accent_lighting` → accent, `lights_task` → task, `night` → night, `lights_security` → outdoor, `light_group` → group.

**Tiebreak:** `primary` label wins within a role slot when multiple candidates match.

---

## Modes

### Discovery and Setup

| Mode | Description |
|------|-------------|
| `discover` | No `room`: whole-home overview — areas with lights, role coverage, setup advisory. With `room`: full inventory — role slots, state, capabilities (dimmable, color_temp, RGB, WLED detection), scene list, RM status, media controller and spa master pointers. |
| `setup` | Creates 8 `zen_lm_*` label definitions. Idempotent. |
| `label_suggest` | Analyzes entity names and effect lists. Proposes `zen_lm_*` roles with confidence scores. Does not apply. Requires `room=`. |
| `resolve_debug` | Traces entity selection chain (stored default → zen_lm_group → zen_lm_main → legacy fallbacks → first-in-room). Returns capability snapshot. Read-only. |
| `scene_list` | All HA scenes. Filter by `room=` and/or `search=`. Returns `entities_affected` per scene and `light_count`. |
| `auto_label` | Auto-tags unlabeled area lights with room label. `dry_run=true` previews (default). Pass `dry_run=false` to apply. |

### Control

| Mode | Description |
|------|-------------|
| `brightness_set` | Set brightness 0–100%. Supports `transition` (0–30s, default 1s). RM hold gate applies. |
| `color_temp_set` | Set color temperature in Kelvin (2700–6500K). Entity must support color_temp. RM advisory. |
| `rgb_set` | Set RGB color as `r,g,b` (0–255 each, e.g. `255,100,0`). RM advisory. |
| `scene_set` | Activate HA scene entity OR apply intent preset. Intent presets: `movie` / `sleep` / `morning` / `work` / `away` / `party` with pre-defined brightness + color_temp. Supports `transition`. |
| `bleed_set` | Primary room scene + propagate to adjacent rooms via Room Manager topology. See Bleed below. |

### Preferences

| Mode | Description |
|------|-------------|
| `prefs_get` | Read all stored lighting contexts for a room from household cabinet. |
| `prefs_set` | Teach a context: `room` + `home_context` + any of `target_role`, `brightness`, `color_temp_k`, `rgb_color`, `scene_entity`, `transition`. |
| `prefs_apply` | Apply stored context for a room. Auto-detects context from RM occupancy state (`Engaged` to `work`, `Asleep` to `sleep`, `Away` to `away`) or falls back to `zen_home_mode`. Re-resolves entity live. Supports `dry_run=true`. |
| `prefs_sweep` | Apply stored preferences across all rooms in one call. `home_context` optional -- defaults to current `zen_home_mode`. Skips rooms with no stored prefs for the context. |
| `room_default_get` | Read stored default role for a room and what it currently resolves to. |
| `room_default_set` | Store `target_role` as room default. Pass `target_role=auto` to clear. |

### Reference

| Mode | Description |
|------|-------------|
| `help` | Full schema, label taxonomy, intent presets, setup flow, example flows. |

---

## Bleed — Topology-Aware Propagation

`bleed_set` applies a scene to the primary room and propagates scaled versions to adjacent rooms based on Room Manager portal optical transmission values.

Propagation rules:

| `light_tx` value | Action |
|-----------------|--------|
| ≥ 0.65 | Full propagation (acoustic zone — light propagates freely) |
| 0.20–0.65 | Scaled propagation (brightness reduced proportionally) |
| < 0.20 | No propagation |

**Door state gating:** `normally-closed` portals are only propagated if the door is currently open (binary sensor in area with device_class `door`, `window`, or `opening`).

**ZenShade sync:** Pass `sync_shades=true` to call `zen_dojotools_covers scene_set` alongside the light bleed. Intent mapping: `movie→movie`, `work→work`, `morning→morning`, `sleep→sleep`, `party→natural`, `away→sleep`.

Requires Room Manager topology to be populated (portals with `light_tx` values). See Room Manager docs.

---

## Room Manager Integration

| Integration | Behavior |
|-------------|---------|
| **Hold Gate** | `brightness_set`, `color_temp_set`, `rgb_set`, `scene_set`, `prefs_apply`: hard-blocked (`status: blocked, reason: rm_hold`) when RM state is `hold` or `pause`. |
| **Active Advisory** | When RM state is not Vacant: all hardware writes include advisory "Room Manager active — occupancy automation may override this command." |
| **Burnout Timer** | `discover` exposes RM occupancy timer (`timeremain`, `seconds`) from the room's sensor. |
| **Bleed Topology** | `bleed_set` reads `room_topology` from household cabinet — portals + boundary_links with `light_tx`, `sound_tx`, `normally`, `to`, `type`. |

RM entity resolved by label (room_manager label on input_select in the area) or fallback pattern `input_select.{room}_occupancy`.

---

## Helpers Required

| Helper | Purpose |
|--------|---------|
| `input_select.zen_home_mode` | Auto-read by `prefs_apply` for context detection |

Preferences stored in household cabinet — no additional helpers needed.
