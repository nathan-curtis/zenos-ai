# 7. Vocabulary

The graph (Part II) is only nodes and edges. What makes it readable is a shared vocabulary, and in ZenOS that vocabulary is labels. A label is a claim about a node: this light is a room's main light, this sensor detects motion for the living room, this script is a DojoTool. Tools find what they act on by asking which nodes carry which labels. They never hardcode an entity ID.

This chapter describes the label families and the rules for using them. The full catalog is in the appendix.

## 7.1 Why labels

Home Assistant gives every entity a domain, a device class, an area, and a name. None of those says what an entity is for. A `light.*` entity could be the main overhead light, an accent strip, a nightlight, or a porch light, and the right behavior differs for each. Names are worse: they vary by vendor and by whoever set up the house.

Labels are the one place a household can state purpose directly, in a form every tool can read with `label_entities()`, and in a form that survives a hardware swap. Replace a thermostat, give the new one the same label, and every tool that used the old one finds the new one.

## 7.2 The families

### Rooms

Every room has a label named for its area, which ties the room's entities together independent of HA's own area assignment. `zen_room_state` marks each room's state sensor, and tools find a room's state by intersecting that label with the room's own label, never by guessing an entity ID.

### Signals (Room Manager v3)

These say what an entity contributes to a room's state (Chapter 12):

| Label | Meaning |
|---|---|
| `motion`, `occupied`, `bed_occupancy` | Presence evidence |
| `engaged` | Evidence of focused activity (media, work) |
| `asleep` | Sleep evidence |
| `hold`, `wasp_door`, `wasp_enabled` | Doors and the enclosed-room ("wasp in a box") model |
| `smoke`, `carbon_monoxide`, `moisture`, `siren` | Emergency detectors |
| `vibration`, `vibration_completion` | Appliance activity and load completion |
| `room_timer`, `room_control_manager` | The room's decay timer and its control surface |

### Scenes

`scene_<state>` (`scene_occupied`, `scene_asleep`, `scene_vacant`, and the rest) marks a scene as the one to fire when a room enters that state. Combined with the room label and an optional home-mode label, it gives REFLEX a three-part key: room, state, time of day. `scene_nightlight` marks the scene for motion while a room is asleep.

### Tool-owned role taxonomies

Several tools own a label family describing the roles their devices play. Each tool creates its own family with `mode=setup`, and each offers a suggestion mode that proposes labels without applying them:

| Family | Tool | Examples |
|---|---|---|
| `zen_lm_*` | ZenLux (lights) | `zen_lm_main`, `zen_lm_accent`, `zen_lm_task`, `zen_lm_night`, `zen_lm_outdoor`, `zen_lm_group` |
| `zen_cv_*` | ZenShade (covers) | `zen_cv_blackout`, `zen_cv_sheer`, `zen_cv_curtain`, `zen_cv_vent`, `zen_cv_primary` |
| `zen_mm_*` | Media Manager | `zen_mm_television`, `zen_mm_speaker`, `zen_mm_speaker_group`, `zen_mm_music_assistant` |
| `zen_plant_*` | Physical Plant | `zen_plant_leak_sensor`, `zen_plant_auto_shutoff`, `zen_plant_battery`, `zen_plant_generator`, `zen_plant_solar` |
| `autovac_*` | AutoVac | `autovac_schedule`, `autovac_current_room`, `autovac_battery` |

`primary` breaks ties inside a role: when two entities fill the same role in a room, the one labeled `primary` wins.

### Security

`security_manager` feeds the security summary. `security_camera` marks the preferred stream of each physical camera. `alarm_panel` marks the panel. `ext_lock` marks exterior locks, which is what puts an unlock behind a live acknowledgement (Chapter 16).

### Master switches and opt-outs

`zenos_roomstate_master_switch` and `zenos_reflex_master_switch` let a household or integration add a switch entity that turns the room-state engine or REFLEX on and off. Opt-out labels (`autosleep_disable`, `asleep_window_disable`, `zen_no_mode`) disable a behavior for the room or entity that carries them.

### Structure

Some labels describe ZenOS itself rather than the house. Cabinets carry type labels (`zen_household_cabinet`, `zen_family_cabinet`, `zen_user_cabinet`, `zen_ai_user_cabinet`, and the `zen_cabinet` parent). Tools carry tier labels (`dojotools`, `admintools`, `sutra`, `stacks`) and domain labels describing what they cover (`calendar`, `finance`, `tickets`, `media_player`, and so on). `zen_kfc_provider` marks a tool that self-registers a Kung Fu Component (Chapter 9).

## 7.3 Rules

**Tools resolve targets through labels.** A tool that needs "the main light in the kitchen" intersects `label_entities('zen_lm_main')` with the kitchen's entities. An explicit entity ID is honored when a caller passes one, but a tool never invents one.

**A tool declares the labels it depends on.** Each tool lists `required_labels` and `optional_labels` in its manifest (Chapter 8). The manifest macro computes `missing_required_labels` and `missing_optional_labels` against the live label registry on every call, and `zen_dojotools_manifest mode=label_audit` aggregates that across the whole system and can create the missing definitions, behind a confirmation.

**Suggest, then apply.** Discovery modes (`label_suggest` in ZenLux, Media Manager, and Plant, `label_discover` in Room Manager) propose labels with a confidence and a reason. Applying them is a separate, confirmed step. A name-pattern match is never enough to label something that controls physical equipment: Plant's suggestion for `zen_plant_auto_shutoff`, for example, is deliberately medium-confidence and warns that a flow sensor with a plausible name is not a valve.

**Labels live on entities, not areas, for resolution.** Home Assistant lets you label an area, but `label_entities()` does not propagate an area's labels to the entities inside it. A label applied only to an area is invisible to entity resolution. Index reports this case explicitly instead of returning a silent empty result.

**Reuse before inventing.** A new behavior should use an existing label wherever one already means the right thing. The label vocabulary is shared across every tool, so a new label has a cost for every future reader of the graph.
