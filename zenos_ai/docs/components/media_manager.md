# ZenOS-AI Media Manager (NyxMau5)

**Version:** 6.0.0
**Script:** `zen_dojotools_media_manager`
**Codename:** NyxMau5

---

## Overview

Zen DojoTools Media Manager is the whole-home AV control layer and **Lens Bus media provider** for ZenOS-AI. It surfaces structured media evidence to the Library and agents, controls all media players via label-driven resolution, and reports live playback state — including full Music Assistant context — to Room Manager and the agent prompt.

No two media setups are alike. MM is designed around this: discover your sources once, save your preferences in Profile Editor, and every future Lens call and room-context read auto-applies them forever. **Set once, use many.**

Key capabilities:

* Lens Bus media provider — music evidence with playback hints, `stacks_by_anchor` room-context injection
* `now_playing` — full playback snapshot consumed by Room Manager `+media` context slice and agent prompts
* Whole-home AV discovery and label-driven entity resolution
* Source and sound mode selection across all room hardware
* Per-user and per-household `media_source_prefs` — preferred sources float to top, excluded sources stripped
* `discovered_sources` returned on every search response — shows what MM found in this install
* HA label taxonomy for hardware role assignment
* Room Manager acoustic topology integration (sound bleed awareness)

---

## "Set Once, Use Many" — Onboarding Flow

No media center is the same. MM guides Friday toward saving your settings so every future call auto-applies them:

1. **`mode=health`** — returns `discovered_sources` list (what MA providers are live in this install)
2. **`mode=discover`** — whole-home AV map; confirms hardware roles are labeled; surfaces `setup_advisory` if labels are missing
3. **Save prefs** — tell Profile Editor (`zen_dojotools_profile_editor`) what sources you prefer and which to exclude: write `media_prefs` to the household cabinet (or `media_prefs_<person_slug>` per person)
4. **Every future call auto-applies** — `stacks_by_anchor`, `search`, `now_playing` all read `media_prefs` on every call and re-rank/filter sources automatically. Nothing to pass — the prefs are live.

`profile_target` layers per-person prefs on top of household base prefs. When a person is calling, their prefs win tiebreaks.

---

## Lens Provider Surface

MM registers as `stack=media` on the Lens Bus. The Library dispatches to it via `zen_stack_media`. **Never call MM directly for knowledge queries — route through the Library.**

```yaml
# Via Library (correct)
section: stacks
stack: media
mode: search
query: "dark jazz"
```

### Lens Modes

| Mode | What It Does |
|------|-------------|
| `stacks_by_anchor` | Anchor-based music evidence. `area_id` anchors inject `room_context` only (what's playing, provider, volume) — they do NOT drive the query. First semantic anchor (mood, activity, concept) drives the MA search. Evidence confidence boosted when item provider matches room's current provider (`room_provider_match`). |
| `search` | Direct music search via MA. `media_source_prefs` applied: preferred sources float, excluded sources stripped. `discovered_sources` returned in every response. `library_only: true` forces MA library scope. |
| `now_playing` | Full playback snapshot. Pass `room=` (area_id) or `entity_id=`. Returns: state, title, artist, album, uri, provider, volume, shuffle, repeat, artwork_url, group_members, `search_metadata` (MA track match with artists + album_uri), `lyrics_hint` (call shape for lyrics — lyrics fetch is async, hint instead of inline). |
| `register` | Register MM in the Lens registry. |
| `unregister` | Remove from Lens registry. |
| `health` | Live health check. Returns `discovered_sources` (all MA providers found), config state, dependency guards. |
| `audit` | Audit MM's Lens registration and source coverage. |
| `tool_manifest` | UMP self-description. |

### Evidence Envelopes

Every `stacks_by_anchor` response contains evidence leaves with `playback_hint` — the caller gets both the item and the call shape to play it. The agent can hand the hint directly to Music Assistant without constructing the call.

```json
{
  "evidence": [
    {
      "title": "Neon Cathedral",
      "artist": "Macklemore",
      "provider": "spotify",
      "confidence": 0.87,
      "playback_hint": {
        "tool": "zen_dojotools_music_assistant",
        "mode": "play_media",
        "media_id": "spotify:track:...",
        "media_type": "track"
      }
    }
  ],
  "room_context": {
    "entity": "media_player.living_room_homecinema",
    "state": "playing",
    "title": "Lo-fi Beats",
    "provider": "spotify",
    "volume": 0.35
  }
}
```

### `media_source_prefs` Schema

Stored in household cabinet as `media_prefs` (or `media_prefs_<person_slug>` for per-person):

```json
{
  "preferred": ["spotify", "tidal"],
  "excluded": ["tunein"],
  "profile_target": "person.friday_user"
}
```

---

## `now_playing` — Room Manager Integration

Room Manager calls `mode=now_playing` as part of the `+media` context slice. Full playback block — including provider, `search_metadata`, and `lyrics_hint` — lands in the room context dict. This replaces the old thin state-read (entity + title + volume only).

Guard pattern:
```jinja
{% if states('script.zen_dojotools_media_manager') not in ['unavailable', 'unknown'] %}
```

Fallback if MM unavailable: `{'status': 'idle', 'room': <area_id>}`.

---

## First-Time Hardware Setup

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

### Step 5 — Save source prefs

Run `mode=health` to see `discovered_sources`. Tell Profile Editor which you prefer. Done — every future Lens call re-applies automatically.

### Step 6 — Verify

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

### Lens Provider (route through Library)

| Mode | Description |
|------|-------------|
| `stacks_by_anchor` | Anchor-based evidence. Area anchors = room context only. Semantic anchors drive search. |
| `search` | Direct MA music search with source prefs applied. |
| `now_playing` | Full playback snapshot: state, title, artist, album, uri, provider, volume, shuffle, repeat, artwork_url, group_members, search_metadata, lyrics_hint. |
| `health` | Health check + discovered_sources. |
| `audit` | Lens registration audit. |
| `register` / `unregister` | Lens registry management. |
| `tool_manifest` | UMP self-description. |

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
| `play_media` | Play media via Music Assistant. When `media_id` is blank and `query` is provided, `query` is passed directly as `media_id` to MA — MA resolves by name. |

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
| `help` | Full reference: label taxonomy, Lens surface, example flows, tips. |

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

**Room Manager v3 PAUSED-awareness** (2026-08-07, corrected 2026-08-09): `play_media` and `activity_apply` refuse to start new media in a room whose `room_control_manager` select reads `Paused` — a human has told Room Manager v3 to leave that room alone (`error: room_locked`-style response naming the paused entity). Read-only/query modes and queue navigation on already-playing media are untouched; only the "begin something new" entry points check this. A room with no `room_control_manager` select deployed is a safe no-op — never blocks.

---

## Preferences Architecture

Preferences are stored in the household cabinet under key `media_prefs` (household) or `media_prefs_<person_slug>` (per-person). Storage is role-based, not entity-ID-based — `prefs_apply` re-resolves the stored role label live at apply time. This means entity IDs can change (upgrades, renames, replacements) without breaking stored preferences.

`home_context` values match `input_select.zen_home_mode` states. If no context is passed at apply time, the current mode value is used automatically.

`media_source_prefs` (`preferred`/`excluded` source lists) are applied automatically on every Lens search and stacks_by_anchor call — no caller action required after initial setup.

---

## Helpers Required

| Helper | Purpose |
|--------|---------|
| `input_select.zen_home_mode` | Current home mode. Used by `prefs_apply` for context auto-read. Must be created manually. |

Preferences are stored in the household cabinet — no additional `input_text` or `input_number` helpers required.

---

## Music Assistant Notes

- `music_assistant.play_media` does **not** support `response_variable` — do not add it
- `music_assistant.queue_command` does **not** exist in this install — all queue control routes through standard `media_player.*` HA services (`media_play`, `media_pause`, `media_stop`, `media_next_track`, `media_previous_track`, `media_seek`, `shuffle_set`, `repeat_set`, `clear_playlist`)
- Tag the correct entity with `zen_mm_music_assistant` — the Music Assistant proxy, not a universal or dead player
