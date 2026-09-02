# Room Manager v3 — New Room Deployment Template

**This is a copy-paste starting point, not something to read cold.** Room
Manager v3's state sensor can only be deployed via a `use_blueprint:` block
in a package YAML file — Home Assistant has no UI path for template-domain
blueprints (confirmed against HA's own docs and community support: *"There
isn't a UI for template blueprints at this time... for now they must be
configured in YAML"*). That means deploying a new room's sensor is a
human-with-file-access task, every time, by design — not something a
household agent can ever do on its own from a chat message. See
[Room Manager v3 & REFLEX manual](room_manager_operators_manual.md) §6 for the plain-language
version of why.

**This file itself is intentionally NOT `.yaml`** — it lives in `docs/`, not
`packages/`, specifically so Home Assistant never auto-includes it and tries
to load the literal `<room>` placeholders as real config. Copy the fenced
block below into a real file under `packages/zenos_ai/room_manager_v3/`.

## Before you touch this file

Some pieces genuinely cannot be created via any tool call — confirmed
against Spook's actual service surface, not assumed:

- `timer.create` / `input_boolean.create` / `input_text.create` /
  `input_select.create` do **not exist** as real HA services (tried,
  reverted 2026-08-05 in `dojotools_ectoplasm.yaml`). Any `timer:`,
  `input_select:` entry below needs a **human**, via
  **Settings → Devices & Services → Helpers → + Create Helper**, matching
  the exact type and options shown — OR just declared directly in this
  package file (the way the block below already does it, which is the
  normal path — Helpers-UI creation is only relevant if you want it
  editable from the dashboard too).
- `input_number.create`/`delete` **do** exist as real Spook services — an
  agent CAN create these live if you ever want that path instead of
  declaring them here.

Everything else in this file — every label tag, every opt-in feature once
the entities below exist — IS agent/tool-operable (`label_discover`,
`coverage_map`, `wasp_enable`, `reflex_wire`, etc.). This template only
covers the one-time part that isn't.

## Steps

1. **Copy the block below** into a new file:
   `packages/zenos_ai/room_manager_v3/zenos_roomstate_<room>.yaml`
2. **Find/replace `<room>`** with the real HA area_id (lowercase, matches
   `zen_dojotools_inspect mode=area_list` exactly — never guess the slug).
3. **Find/replace `<Room Display Name>`** with the human-readable name
   (e.g. `Kitchen`, `Master Bedroom`).
4. **Delete anything you don't need.** Every block below is genuinely
   optional except the `use_blueprint:` call itself — a minimal room
   (a hallway, a closet) might keep only `room_timer` + the blueprint,
   nothing else. See the operator's manual §6/§8 worked examples for what
   real rooms actually keep vs. drop.
5. **Fill in `trigger_entities:`** — every real sensor/media_player/timer/
   helper you want this room's state to react to. Run
   `mode=label_discover area=<room>` first to find real candidates rather
   than guessing entity_ids.
6. **If you added `child_rooms:`**, list the child's own state sensor
   entity_id (e.g. `sensor.master_bathroom_room_state`), not the child's
   area name.
7. **Reload**: `zen_dojotools_systemtools mode=ha_config_check`, then
   `mode=ha_reload_template_entities` (NOT `ha_reload_templates` — that one
   is Jinja macro files only and silently does not cover this).
8. **Verify**: `sensor.<room>_state` should read `vacant` (or the correct
   live tier) with attributes populated. From here, everything else —
   wasp gate, coverage tooling, REFLEX wiring — is normal tool-operable
   territory. Ask your AI to take it from here.

## The template

```yaml
###############################################################################
# ZenOS Room Manager v3 — <Room Display Name>
#
# Deployed <DATE>. Real entities tagged via label_discover — see
# coverage_map area=<room> for current wiring status.
###############################################################################
timer:
  <room>_room_timer:
    name: "<Room Display Name> Room Timer"
    icon: mdi:timer-outline
  <room>_control_burnout:
    name: "<Room Display Name> Control Burnout"
    icon: mdi:timer-alert-outline
  # OPTIONAL — delete if this room has no TV / doesn't need a sleep timer.
  <room>_tv_sleep_timer:
    name: "<Room Display Name> TV Sleep Timer"
    icon: mdi:timer-outline

input_number:
  # OPTIONAL — delete if this room never needs Asleep at all (e.g. a
  # hallway, closet, garage). Keep if it has any asleep/bed_occupancy
  # signal.
  <room>_asleep_minutes:
    name: "<Room Display Name> Asleep Timeout"
    min: 1
    max: 720
    step: 1
    unit_of_measurement: min
    mode: box
    icon: mdi:timer-outline
    initial: 30
  # OPTIONAL — pairs with the tv_sleep_timer above, delete together.
  <room>_tv_sleep_minutes:
    name: "<Room Display Name> TV Sleep Duration"
    min: 1
    max: 240
    step: 1
    unit_of_measurement: min
    mode: box
    icon: mdi:timer-outline
    initial: 20

input_select:
  # Legacy-parity control select — most rooms keep this even if unused,
  # matches the naming convention some older automations still check.
  <room>_control:
    name: "<Room Display Name> Control"
    options: [Auto, Paused, Automation, Cleaning]
    icon: mdi:tune
  <room>_room_timer_class:
    name: "<Room Display Name> Room Timer Class"
    options: [occupied, engaged, asleep]
    initial: occupied
    icon: mdi:timer-cog-outline

template:
  - trigger:
      # CABINET-BACKED — trigger on the household cabinet sensor itself
      # (any drawer write fires this; template recompute is cheap) instead
      # of a dedicated input_text helper. This pattern is identical across
      # every room, copy verbatim, only the room slug/display name change.
      - trigger: state
        entity_id: sensor.zenos_default_household_cabinet
      - trigger: homeassistant
        event: start
    select:
      - name: "<Room Display Name> Control Manager"
        # ROOM-MANAGER-OWNED control surface — separate, non-colliding
        # parallel system alongside any legacy input_select.<room>_control.
        # Persistence lives in the household cabinet's room_control_manager
        # drawer (single dict, keyed by room), not a per-room helper.
        # select_option does nothing but fire room_control_request —
        # zen_room_manager_dispatch.yaml's listener is the sole writer.
        unique_id: <room>_control_manager_cab_select
        icon: mdi:tune
        state: >-
          {%- set _rcm_vars = state_attr('sensor.zenos_default_household_cabinet', 'variables') | default({}, true) -%}
          {%- set _rcm_entry = _rcm_vars.get('room_control_manager', {}) if _rcm_vars is mapping else {} -%}
          {%- set _rcm_raw = _rcm_entry.get('value') if _rcm_entry is mapping else none -%}
          {%- set _rcm = (_rcm_raw | from_json) if (_rcm_raw is string and _rcm_raw not in ['', none]) else (_rcm_raw if _rcm_raw is mapping else {}) -%}
          {{ (_rcm.get('<room>') | default('Auto', true)) if _rcm is mapping else 'Auto' }}
        options: >-
          {%- set _rcm_vars = state_attr('sensor.zenos_default_household_cabinet', 'variables') | default({}, true) -%}
          {%- set _rcm_entry = _rcm_vars.get('room_control_manager', {}) if _rcm_vars is mapping else {} -%}
          {%- set _rcm_raw = _rcm_entry.get('value') if _rcm_entry is mapping else none -%}
          {%- set _rcm = (_rcm_raw | from_json) if (_rcm_raw is string and _rcm_raw not in ['', none]) else (_rcm_raw if _rcm_raw is mapping else {}) -%}
          {%- set _rcm_cur = (_rcm.get('<room>') | default('Auto', true)) if _rcm is mapping else 'Auto' -%}
          {%- if _rcm_cur == 'Hold' -%}
            {{ ['Hold', 'Auto', 'Vacant', 'Paused', 'Asleep'] }}
          {%- else -%}
            {{ ['Auto', 'Vacant', 'Occupied', 'Engaged', 'Asleep', 'Hold', 'Paused', 'Automation', 'Cleaning'] }}
          {%- endif -%}
        select_option:
          - action: script.zen_dojotools_event_emitter
            data:
              mode: emit
              component: room_manager
              kind: room_control_request
              severity: info
              summary: "<room> control set to {{ option }}"
              metadata:
                room: <room>
                requested: "{{ option }}"
                source: human
  - use_blueprint:
      path: zenos/room_state.yaml
      input:
        room: <room>
        friendly_name: "<Room Display Name> State"
        unique_id: <room>_state_v3
        # OPTIONAL — delete entirely if this room has no ensuite/sub-room.
        # If kept, list the CHILD's state sensor entity_id, not its area.
        child_rooms:
          - sensor.<child_room>_state
        trigger_entities:
          # Every real entity this room's state should react to. Run
          # mode=label_discover area=<room> first — don't guess these.
          - binary_sensor.<room>_motion_sensor_example
          - timer.<room>_room_timer
          - input_select.<room>_room_timer_class
          - input_select.<room>_control
          # Delete these two lines if you deleted the asleep_minutes/
          # tv_sleep_timer blocks above.
          - input_number.<room>_asleep_minutes
          - timer.<room>_tv_sleep_timer
```

## After deployment

Everything from here is normal tool territory — ask your AI, or call
directly:

- `mode=label_discover area=<room>` — find and tag real signal entities.
- `mode=coverage_map area=<room>` — confirm what's wired vs. still gapped.
- `mode=wasp_enable area=<room>` — if this room has a real, fully-enclosed
  entry door (not an open archway).
- `mode=reflex_wire area=<room>` — wire scenes once you have them.

---

**Known limitation, planned for a real fix:** this manual-file-deployment
requirement is the single biggest gap in making Room Manager v3 fully
self-service via chat. High priority for a future feature update —
tracked, not forgotten. See the operator's manual's "What's Next" section.
