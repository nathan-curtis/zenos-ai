# ZenOS-AI Media Manager (NyxMau5)

**Version:** 5.1.0
**Script:** `zen_dojotools_media_manager`
**Codename:** NyxMau5

---

## Overview

Zen DojoTools Media Manager is the whole-home AV control layer for Home Assistant. It provides governed, label-driven access to media players across all rooms — resolving intents to the correct physical device without hardcoded entity IDs.

Key capabilities:

* Whole-home AV discovery and mapping
* Source and sound mode selection
* Per-room preference storage and application
* HA label taxonomy for hardware role assignment
* Room Manager acoustic topology integration (sound bleed awareness)

---

## First-Time Setup

### Step 1 — Create labels

```
mode=setup
```

Creates 16 label definitions in HA:
- 9 hardware role labels (`zen_mm_*`)
- 5 intent/capability labels (`watch`, `listen`, `announce`, `video`, `voice`)
- 1 system label (`zen_tts`)
- 1 priority label (`primary`)

Idempotent — safe to re-run. Does not modify existing entity assignments.

### Step 2 — Discover

```
mode=discover
```

No `room` parameter = whole-home survey. Returns per-area summary showing:
- Which hardware roles are labeled per room
- Count of unlabeled media players
- Which entity would be chosen by the auto-resolution chain
- `setup_advisory` if no `zen_mm_*` labels exist yet

### Step 3 — Suggest labels

```
mode=label_suggest  room=<room_name>
```

Analyzes entity names and integration type. Returns `zen_mm_*` role suggestions with confidence ratings. Does **not** apply labels — review and apply via `zen_dojotools_labels`.

### Step 4 — Apply labels

Use `zen_dojotools_labels` to tag entities with the suggested roles. Example:
```
zen_dojotools_labels  action_type=tag  target_entities=media_player.living_room_tv  label_list=zen_mm_television,watch
```

### Step 5 — Verify

```
mode=discover  room=<room_name>
```

Confirms all roles are resolved and `setup_advisory` is clear.

---

## Label Taxonomy

### Hardware Roles (`zen_mm_*` — tool-owned)

Apply one of these per device. `mode=setup` creates the definitions; you apply them to entities.

| Label | Device Type | Resolution Priority |
|-------|-------------|---------------------|
| `zen_mm_universal` | Primary command surface (TV with built-in smarts, universal controller) | Top of chain |
| `zen_mm_television` | TV or display. Source list = streaming app names when on. | — |
| `zen_mm_av_tuner` | AV receiver. Volume, sound mode, physical inputs. | — |
| `zen_mm_audio_output` | Dedicated audio zone: amp, zone player, audio-only device. | — |
| `zen_mm_video_output` | Streaming device: Chromecast, Fire Stick, Apple TV. | — |
| `zen_mm_speaker` | Standalone in-room speaker (not grouped). | — |
| `zen_mm_speaker_group` | Grouped speaker entity (`group_size` > 1). | — |
| `zen_mm_music_assistant` | Music Assistant proxy entity. | — |
| `zen_mm_voice_assistant` | Voice interface (Alexa, Google Home). Companion only — not primary AV. | — |

### Intent / Capability Labels

Apply to any entity that serves this purpose. Multiple tools (Postman, Dispatcher) route intents via these labels.

| Label | Purpose |
|-------|---------|
| `watch` | Designated for video/streaming content |
| `listen` | Designated for music/audio playback |
| `announce` | Designated for announcements and TTS delivery |
| `video` | Entity provides video display or output |
| `voice` | Entity is a voice interface |
| `zen_tts` | ZenOS TTS output for the room. One per room preferred. Used by Postman and announcement flows. |
| `primary` | Highest-priority entity within a labeled group. Wins tiebreaks. |

---

## Modes

### Discovery and Setup

| Mode | Description |
|------|-------------|
| `discover` | No `room`: whole-home AV map — all areas, role coverage, unlabeled count, default resolution. With `room`: full role map with state, source lists, group membership. |
| `setup` | Creates all 16 label definitions in HA. Idempotent. |
| `label_suggest` | Analyzes room entities by name and integration. Proposes `zen_mm_*` assignments. Returns suggestions only — does not apply. |
| `resolve_debug` | Traces entity selection chain for a room + optional intent/role/source. Returns `selection_chain`, selected entity, reason, warnings. Read-only. |

### Device Control

| Mode | Description |
|------|-------------|
| `source_get` | Current source + full `source_list` for a device. Call before `source_set` to see exact source strings. |
| `source_set` | Switch input source. Exact string match required — copy from `source_list`. Use `target_role=zen_mm_television` for streaming apps, `target_role=zen_mm_av_tuner` for physical inputs. |
| `sound_mode_get` | Current `sound_mode` + full `sound_mode_list`. Call before `sound_mode_set`. |
| `sound_mode_set` | Set AVR or TV sound mode. Use `target_role=zen_mm_av_tuner` for the receiver. |
| `state_get` | Full device snapshot: source, sound mode, volume, mute, media title, group membership, adjacent room co-playing detection, acoustic zone context. |

### Preferences

| Mode | Description |
|------|-------------|
| `prefs_get` | Read all stored preferences for a room — see every context and what source/volume it maps to. |
| `prefs_set` | Teach a preference: `room` + `home_context` + `source` (+ optional `volume` + `target_role`). Stores role as the resolver — not entity ID — so it survives entity renames. |
| `prefs_apply` | Apply stored preference for a room. Reads `input_select.zen_home_mode` automatically if no `home_context` given. Re-resolves device from stored role label live. Supports `dry_run=true`. |
| `room_default_get` | Read stored default role for a room. Shows what it resolves to and what auto-chain would pick. |
| `room_default_set` | Store a `target_role` as the room's default resolver. Pass `target_role=auto` to clear back to automatic chain. |

### Reference

| Mode | Description |
|------|-------------|
| `help` | Full reference: label taxonomy, example flows, tips. |

---

## Entity Resolution Chain

When no `target_role` is specified, the tool walks this chain and returns the first match:

1. Stored room default (`room_default_set`)
2. `zen_mm_universal`
3. `zen_mm_television`
4. `zen_mm_av_tuner`
5. `zen_mm_audio_output`
6. `zen_mm_video_output`
7. `zen_mm_speaker_group`
8. `zen_mm_speaker`
9. Any unlabeled `media_player.*` in the area (fallback)

Use `resolve_debug` to trace why a specific entity was (or wasn't) selected.

---

## Source Lists

Source list values come directly from `state_attr(entity_id, 'source_list')`. **Exact string matching required** — no fuzzy matching. Device must be on for streaming apps to appear in the list.

Typical source types by role:

| Role | Source type |
|------|-------------|
| `zen_mm_television` | Streaming app names (`Netflix`, `Disney+`, `Plex`) when powered on |
| `zen_mm_av_tuner` | Physical inputs (`HDMI 1`, `Optical`, `Coax`) |
| `zen_mm_video_output` | Streaming apps (`Prime Video`, `Apple TV+`) |

Always call `source_get` before `source_set` to retrieve the exact strings.

---

## Room Manager Integration

`state_get` reads acoustic topology from the household cabinet (Room Manager `room_topology` key):

* Pulls portals and boundary links for the room
* For adjacent rooms with `sound_tx >= 0.65`: checks if that room is in a guarded state (`hold`, `sleep`, `sleeping`, `pause`)
* Returns advisory if high-transmission neighbor is held or sleeping
* Returns co-playing media in adjacent rooms (entity, state, media title, volume)

This is passive — no writes to Room Manager. Configure transmission values via `zen_dojotools_room_manager mode=link`.

---

## Preferences Architecture

Preferences are stored in the household cabinet under key `media_prefs_<room_slug>`. Storage is role-based, not entity-ID-based — `prefs_apply` re-resolves the stored role label live at apply time. This means entity IDs can change (upgrades, renames, replacements) without breaking stored preferences.

`home_context` values match `input_select.zen_home_mode` states. If no context is passed at apply time, the current mode value is used automatically.

---

## Helpers Required

| Helper | Purpose |
|--------|---------|
| `input_select.zen_home_mode` | Current home mode. Used by `prefs_apply` for context auto-read. Must be created manually. |

Preferences are stored in the household cabinet — no additional `input_text` or `input_number` helpers required.
