# ZenOS Universal Media Player Spec
**Contract version:** 1.0.0  
**Owned by:** zen_dojotools_media_manager  
**Status:** Required for activity system participation

---

## What This Is

ZenOS Media Manager's activity system (`activity_apply`, `activity_snapshot`) routes all playback commands through a single entity per room labeled `zen_mm_universal`. That entity is your **Universal Media Player** — a household-built HA `media_player: platform: universal` (or equivalent) that knows your specific wiring: which HDMI port is which, how long your TV takes to boot, what to do when Friday interrupts.

ZenOS defines this contract. You build the implementation. ZenOS never touches your TV, AVR, or HDMI ports directly.

---

## The Contract

### 1. Required Label

Your UMP entity **must** carry the label `zen_mm_universal` in the room it serves.

```yaml
# Applied via zen_dojotools_labels or HA UI
# One entity per room — no duplicates
label: zen_mm_universal
label: living_room   # your room area label
```

### 2. Must Accept `select_source` with Activity Names

When ZenOS calls `activity_apply room="Living Room" activity_name="Xbox"`, it sends:

```yaml
service: media_player.select_source
target:
  entity_id: <your zen_mm_universal entity>
data:
  source: "Xbox"   # the activity name, verbatim
```

Your UMP must translate that source string into whatever multi-device sequence is needed. ZenOS does not know — and must not need to know — about HDMI ports, boot times, AVR inputs, or sequencing delays. That is entirely your responsibility.

### 3. Must Reflect Current Activity in `source` Attribute

After an activity is applied, `state_attr(entity, 'source')` should return the active activity name (or a string close enough that `activity_snapshot` can capture it meaningfully).

This is what `activity_snapshot` reads to propose a saved activity. If `source` is a raw FireTV package ID instead of "YouTube", the snapshot will save noise.

### 4. Must Enumerate Activities in `source_list`

`state_attr(entity, 'source_list')` should return the list of valid activity names. This is what `discover` shows users when they ask what's available in a room.

```python
source_list: ["Xbox", "YouTube", "Netflix", "Media Player", "AV Receiver UI", "Music", "HDMI2"]
```

### 5. Standard Media Controls Must Route Correctly

| Command | Expected behavior |
|---------|------------------|
| `turn_on` | Power on primary device(s) for the room |
| `turn_off` | Power off gracefully — your sequencing |
| `volume_up/down/set` | Route to the room's volume authority (typically AVR) |
| `volume_mute` | Route to volume authority |
| `media_play/pause/stop` | Route to currently active child |
| `media_next/previous_track` | Route to currently active child |

---

## Implementation Pattern

HA's built-in `media_player: platform: universal` is the recommended base. It gives you:
- `children:` — list of child players (state inheritance)
- `active_child_template:` — Jinja2 logic to pick which child is active right now
- `commands:` — map standard commands to services
- `attributes:` — override `volume_level`, `source`, `source_list` from any child

The `select_source` command is where your activity logic lives:

```yaml
media_player:
  - platform: universal
    name: Living Room Universal
    unique_id: your-unique-id-here

    children:
      - media_player.your_avr_zone
      - media_player.your_tv
      - media_player.your_ma_player
      # ... all room players

    attributes:
      volume_level: media_player.your_avr_zone|volume_level
      is_volume_muted: media_player.your_avr_zone|is_volume_muted
      source_list: media_player.your_avr_zone|source_list   # or hardcode

    commands:
      select_source:
        service: script.your_room_route_activity
        data:
          activity: "{{ source }}"   # ZenOS passes the activity name here

      turn_on:
        service: media_player.turn_on
        target:
          entity_id: media_player.your_avr_zone

      turn_off:
        service: script.your_room_power_off

      volume_up:
        service: media_player.volume_up
        target:
          entity_id: media_player.your_avr_zone

      # ... etc
```

### Your Route Script

`script.your_room_route_activity` receives the activity name and handles all sequencing:

> **Before you write HDMI routing:** check your TV integration first — see [Platform Notes](#platform-notes) below. Many TV integrations cannot switch physical HDMI inputs programmatically.

```yaml
your_room_route_activity:
  fields:
    activity:
      selector:
        text:
  sequence:
    - choose:
        - conditions: "{{ activity == 'Xbox' }}"
          sequence:
            - service: media_player.turn_on
              target: { entity_id: media_player.your_tv }
            - wait_template: "{{ not is_state('media_player.your_tv', 'off') }}"
              timeout: "00:00:20"
            # NOTE: only works if your TV integration exposes HDMI inputs in source_list.
            # Fire TV / androidtv integrations do NOT — use CEC instead (see Platform Notes).
            - service: media_player.select_source
              target: { entity_id: media_player.your_tv }
              data: { source: "HDMI 1" }
            - service: media_player.turn_on
              target: { entity_id: media_player.your_avr_zone }
            - service: media_player.select_source
              target: { entity_id: media_player.your_avr_zone }
              data: { source: "TV Audio" }

        - conditions: "{{ activity == 'Music' }}"
          sequence:
            - service: media_player.turn_off
              target: { entity_id: media_player.your_tv }
            - service: media_player.turn_on
              target: { entity_id: media_player.your_avr_zone }
            - service: media_player.select_source
              target: { entity_id: media_player.your_avr_zone }
              data: { source: "HEOS Music" }

        # ... one block per activity
```

**Key pattern:** `wait_template` after power-on before source select. This is why ZenOS does not do this itself — only you know how long your TV takes to boot, and HA `wait_template` with a timeout is the correct primitive.

---

## What ZenOS Manages

| ZenOS owns | You own |
|-----------|---------|
| Activity name registry (cabinet) | HDMI port map |
| `activity_get/set/apply/snapshot` | Boot wait times |
| Label resolution (`zen_mm_universal`) | Multi-device sequencing |
| LLM-facing activity name surface | Power-on/off order |
| `discover` room topology | AVR input routing |
| Volume/skip routing via active_child | Conflict resolution (Friday interrupts music) |

---

## Registering With ZenOS

Once your UMP is built and labeled:

```
zen_dojotools_media_manager mode=activity_set room="Living Room" activity_name="Xbox" channel="Xbox"
```

Or use `activity_snapshot` after manually switching to an activity — ZenOS reads your UMP's `source` attribute and saves it.

To verify ZenOS can see your UMP:

```
zen_dojotools_media_manager mode=discover room="Living Room"
→ look for role: zen_mm_universal with your entity_id
```

---

## Know Your AV Router

Before writing a route script, run this to get your actual source strings:

```
zen_dojotools_media_manager mode=source_get room="Your Room" target_role=zen_mm_av_tuner
```

The `source_list` in the response contains the exact strings your router accepts. Use them verbatim. Do not guess.

### Why this matters

Different hardware uses different source string conventions:

| Hardware | Example source strings |
|----------|----------------------|
| Denon AVR (full receiver) | `TV Audio`, `HEOS Music`, `Bluetooth`, `Game` |
| Denon HomeCinema (soundbar) | `<Room> HomeCinema - HDMI OUT (ARC)`, `<Room> HomeCinema - AUX In` |
| Sony / Yamaha / Onkyo | Varies — always check |

A source string that works in one room will silently fail in another if the hardware is different. `select_source` with an unrecognized string does nothing and returns no error.

### Common wiring topologies

**AVR-centric (e.g. living room):**
```
TV ──HDMI──► AVR (HDMI input)
                └── "TV Audio" source on AVR
AVR ──HDMI ARC──► TV (display only)
Volume authority: AVR
```

**Soundbar-centric (e.g. bedroom):**
```
TV ──HDMI ARC──► Soundbar ("HDMI OUT (ARC)" input)
Volume authority: Soundbar
```

Both map to the same ZenOS contract — `zen_mm_av_tuner` is the volume authority regardless. Only the source strings and wiring differ. Your route script handles it; ZenOS doesn't need to know.

### `source_list` is checked before every activity

`activity_apply` checks the activity's channel against your UMP's live `source_list` before it calls `select_source`. A channel that is not in the list is skipped, and the response says so, instead of Home Assistant raising a raw `ServiceValidationError`. So `source_list` has to enumerate every activity name your route script accepts.

Templating a *list*-typed attribute on `platform: universal` (rather than a scalar like `source`) can silently fail: the attribute disappears from state with no error logged, even when the config check passes. If that happens, hardcode `source_list` on the UMP instead of templating it.

> **Not yet built.** A per-entity opt-out (a label that skips the `source_list` check for route-script UMPs that validate their own channels) has been discussed but is not in the code. Every UMP gets the check today.

### `active_child_template` priority is yours to define

ZenOS does not prescribe which child wins when multiple are active. Design it for your room:
- Example: Music Assistant wins immediately when playing (music takes over everything)
- Example: the voice assistant's AUX input wins first, then the TV over ARC, then Music Assistant

There is no wrong answer — just document your priority order in a comment.

---

## Platform Notes

### Fire TV / Amazon androidtv integration

The `androidtv` integration talks to Fire TV OS, not the physical TV hardware. This means:

- `source_list` contains **streaming app package IDs** (e.g. `com.amazon.tv.launcher`, `com.amazon.tv.inputpreference.service`) — not HDMI input names
- `current_source` reports the active Fire TV app or service, not the physical HDMI port
- `select_source: "HDMI1"` **does nothing** — the string is not in source_list and will be silently ignored
- Switching to physical HDMI inputs (Xbox, Blu-ray player, etc.) is handled by the TV's hardware, not Fire TV OS

**Diagnostic:** if `source_get` shows only package IDs in `source_list`, and the list doesn't change when you physically switch to an HDMI input — your TV is in this category.

**How to handle HDMI-connected devices (Xbox, game consoles, etc.):**

Option A — CEC auto-switch (recommended):
- Enable CEC on both the TV and the connected device
- Your route script just turns on the TV and sets up the AV stack
- When the device powers on, the TV switches HDMI input automatically
- ZenOS fires `activity_apply`, stack comes up, user presses device power button

```yaml
# Xbox activity — CEC handles HDMI switching
- conditions: "{{ activity == 'Xbox' }}"
  sequence:
    - service: media_player.turn_on
      target: { entity_id: media_player.your_fire_tv }
    - service: media_player.turn_on
      target: { entity_id: media_player.your_avr_zone }
    - service: media_player.select_source
      target: { entity_id: media_player.your_avr_zone }
      data: { source: "TV Audio" }
    # No select_source on the TV — CEC switches input when Xbox turns on
```

Option B — IR blaster:
- Use a Broadlink, Harmony, or similar IR blaster with an HA integration
- Add an `action: remote.send_command` step to your route script after TV is on

**Fire TV apps work fine** — `YouTube (FireTV)`, `Netflix`, `Spotify` etc. are package names that ARE in `source_list` and respond correctly to `select_source`.

### LG WebOS, Samsung SmartThings, Sony Bravia

These integrations typically DO expose physical HDMI inputs in `source_list` as human-readable strings (`"HDMI 1"`, `"HDMI 2"`, etc.). `select_source` works for input switching on these platforms. Verify by running `zen_dojotools_media_manager mode=source_get` with your TV on the HDMI input — if `current_source` shows the HDMI name, you're good.

---

## Compliance Checklist

- [ ] Entity labeled `zen_mm_universal` + room area label
- [ ] `select_source` routes to your activity script
- [ ] `source` attribute reflects current activity name after switching
- [ ] `source_list` enumerates valid activity names
- [ ] Volume commands route to AVR (or room volume authority)
- [ ] `turn_on` / `turn_off` work
- [ ] `active_child_template` returns correct child for play/pause/skip routing
- [ ] Activity names registered in ZenOS cabinet via `activity_set`
- [ ] If a templated `source_list` doesn't appear in state: hardcoded instead

---

## Why Not Just Let ZenOS Do It?

ZenOS calling your TV and AVR directly — with hardcoded delays — is fragile, household-specific, and wrong. Your TV takes 12 seconds to boot. Your neighbor's takes 3. Your AVR needs `TV Audio` selected *after* the TV is on, or the audio detection fails. ZenOS cannot know any of this.

The Universal Media Player is the right abstraction boundary. You build it once. ZenOS calls it forever. When you replace your TV with one that boots in 2 seconds, you update your script. ZenOS doesn't change.
