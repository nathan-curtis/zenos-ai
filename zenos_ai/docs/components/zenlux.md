# ZenOS-AI ZenLux — Lighting Manager

**Version:** 2.2.0 (2026-08-16 — see note below on the version-number reset)
**Script:** `zen_dojotools_lights`
**Codename:** ZenLux

> **Version numbering note:** this file's version tracked a `5.x` series through 5.2.0. The script's own `tool_manifest` restarted its version counter independently at some point before this doc was updated (now `2.2.0`) — a known drift class across several DojoTools scripts (Zammad #10300), not specific to ZenLux. Trust the script's own `mode=help`/`tool_manifest` response over this header if they ever disagree again.

---

## Overview

ZenLux is the room-aware control layer for ZenOS-AI's `light.*` **and, as of 2026-08-16, `switch.*`** domains. It provides governed, label-driven control with memory (preferences), topology propagation (bleed to adjacent rooms), Room Manager integration (hold gate, occupancy awareness, room-lock guard), and — for the actions that actually change hardware state — an identity/certification gate.

Key capabilities:

* Label-based entity resolution — no hardcoded entity IDs
* Real `switch.*` write support (`toggle`, `switch_set`) — did not exist before 2026-08-16; every write action previously called `light.turn_on`/`light.turn_off` exclusively even where a switch was the logical target
* Single-entity `mode=inspect` — capability/label/area detail for one light or switch, same shape as the other domain tools' inspect modes
* `mode=scene_stage` — relocated here from Room Manager (see below)
* Topology-aware bleed: propagates scenes to adjacent rooms via Room Manager portal transmission values
* RM hold gate + room-lock guard: hard-blocks hardware writes when a room is in Hold/Pause, or when a human has parked `room_control_manager` on Paused/Automation/Cleaning
* Identity-gated writes: `lighting_control` certification required on every write mode (see Identity Gate below)
* Per-room lighting preferences stored in household cabinet, re-resolved live at apply time
* ZenShade sync: optional cover scene applied alongside light bleed
* Discovery and label suggestion for initial setup
* Full SESE (single-entry/single-exit) + canonical `envelope()` response shape across all modes (2026-08-16 conversion — see `zen_os1_jinja.md`'s `envelope()` section)

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
| `inspect` (2026-08-16) | Single-entity deep detail for one `light.*` or `switch.*` — capabilities, area/device, all labels. Requires `entity_id=`; the entity must actually exist and be in one of those two domains. Read-only. |
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
| `reflex_sync` | Fires whatever Room Manager v3's REFLEX Stage 2 currently resolves for this room's live state, by calling `zen_dojotools_room_manager mode=reflex_wire` and reading `wired_matrix` — not a separate re-derivation of resolution logic. Not gated by the room-lock check below: a locked room's own resolved state already accounts for the lock (it fires that state's own wired scene, which isn't a violation of the lock, it *is* the lock's scene). |
| `toggle` (2026-08-16) | Domain-agnostic — `homeassistant.toggle` on `entity_id=`, works on `light.*` or `switch.*`. `entity_id=`-only, deliberately: `room=` would be ambiguous light-vs-switch without an explicit hint. RM hold gate + room-lock guard + `lighting_control` cert all apply, same as every other write mode. |
| `switch_set` (2026-08-16) | Explicit `switch.*` write: `switch_state=on\|off\|toggle`. `entity_id=` overrides; otherwise `room=` (+ optional `target_role=`) resolves via the same room/role/`primary`-tiebreak convention the light modes use — but switches have no fixed role-label taxonomy the way lights do, so `target_role=` here is a plain label filter, not one of the `zen_lm_*` roles. |
| `scene_stage` (2026-08-16, relocated from Room Manager) | Works around a real HA limitation: `scene.create` doesn't persist. `scene_phase=before` snapshots every light/cover/climate/media_player/switch/fan/humidifier entity in a room before a scene-worthy action; `scene_phase=after` snapshots again; `scene_phase=confirm` auto-detects and tags what actually changed between the two. `room=` required. Room Manager still exposes its own `mode=scene_stage` as a thin backward-compatible delegator to this one — logic lives here now, not there (RBAC locality: the write belongs with the tool that owns the domain it's staging). |

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

**Fixed (2026.7.1):** the `room_topology` cabinet read (used by bleed and discovery) now goes through the shared `CABS.cabinet_drawer_value_mounted()` macro instead of hand-rolled inline logic. The inline version dropped mount-pointer following — if `room_topology` was ever stored behind a mount pointer rather than directly in the household cabinet, bleed propagation would silently see empty topology instead of following the pointer to the real data.

---

## Room Manager Integration

The full write-mode set gated by both rows below is: `scene_set`, `brightness_set`, `color_temp_set`, `rgb_set`, `bleed_set`, `prefs_apply`, `toggle`, `switch_set` (the last two added 2026-08-16 alongside real switch support — easy to miss adding a new write mode to this list, and a real gap this session: the room-lock guard's own hardcoded mode list was initially missed when `toggle`/`switch_set` shipped, meaning both could briefly bypass RM Hold protection entirely; fixed same day).

| Integration | Behavior |
|-------------|---------|
| **Hold Gate** | All write modes above: hard-blocked (`status: blocked, reason: rm_hold`) when RM state is `hold` or `pause`. |
| **Room Lock Guard (RM v3)** | All write modes above also check `room_control_manager` directly — blocked (`error: room_locked`) while it reads `Paused`/`Automation`/`Cleaning`, or while any `hold`-labeled entity for the room is active (a door propping the room open, "fridge door mode"). Pass `force=true` for an explicit human override, or use `reflex_sync` instead, which already accounts for the lock. Separate mechanism from the Hold Gate row above — same convention `room_state.yaml`'s own cascade uses. |
| **Active Advisory** | When RM state is not Vacant: all hardware writes include advisory "Room Manager active — occupancy automation may override this command." |
| **Burnout Timer** | `discover` exposes RM occupancy timer (`timeremain`, `seconds`) from the room's sensor. |
| **Bleed Topology** | `bleed_set` reads `room_topology` from household cabinet — portals + boundary_links with `light_tx`, `sound_tx`, `normally`, `to`, `type`. |

RM entity resolved by label (room_manager label on input_select in the area) or fallback pattern `input_select.{room}_occupancy`.

---

## Identity Gate (2026-08-16)

Every write mode listed above additionally requires the `lighting_control` certification — declared-but-unenforced when this doc last covered it, turned on for real this pass. Cert-only: unlike locks' exterior unlock, covers' barrier-open, or security_manager's disarm, no ZenLux action requires a fresh live acknowledgment on top of the cert — lighting/switch control isn't treated as a physical-security action. See the [Security & Certification System operator manual](../getting_started/security_certification_manual.md) for the full model and grant syntax.

Resolution goes through the shared `cert_scope_check` macro (`zen_os_1.jinja`) via `zen_dojotools_identity mode=resolve_caller_identity target_entity=<entity>` — the same single chokepoint locks/covers/security_manager/room_manager/infra all use as of 2026-08-16, rather than each tool walking its own `cert_scope` array. Because lighting has no live-ack tier, the `scope_decision` this returns doesn't currently change ZenLux's own behavior — it's wired in for consistency and to be ready if that ever changes, not because it's load-bearing here today.

---

## Helpers Required

| Helper | Purpose |
|--------|---------|
| `input_select.zen_home_mode` | Auto-read by `prefs_apply` for context detection |

Preferences stored in household cabinet — no additional helpers needed.

---

## Version History

| Version | Change |
|---------|--------|
| 2.2.0 (2026-08-16) | Real `switch.*` write support (`toggle`, `switch_set`), `mode=inspect`, `mode=scene_stage` relocated in from Room Manager, `lighting_control` identity gate turned on for real, full SESE + `envelope()` conversion across all modes, `mode=help`/`tool_manifest` content corrected (stale version string, 5 modes missing from the static mode list). Two real bugs found and fixed doing the conversion by hand: an internal validate-then-continue guard's early result could be silently overwritten by code that should never have run once its own `stop:` was stripped; a `choose:` block mis-nested at the wrong indentation level passed `config_check` cleanly while being dead code. |
| 5.2.0 | Room Lock Guard (RM v3) — checks `room_control_manager` directly, `force=true` override. |
