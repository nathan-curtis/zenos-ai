# zen_dojotools_zenzork

**ZenZork Adventure Engine** — v1.1.0
**File:** `packages/zenos_ai/dojotools/dojotools_zenzork.yaml`

---

## What it does

ZenZork is a text adventure engine that uses your live Room Manager topology as the dungeon. Walk through your actual house in Zork mode. Every room narrates from live HA state — real lighting levels, real climate data, real portal bearings. Game session state persists in the AI user cabinet at drawer `zenzork_state`.

Navigation is compass-bearing-native. Portals are the same bearing-tagged entries in `room_topology` that Room Manager uses for everything else. Walk the house, and ZenZork derives your floor plan from how you move through it.

---

## Modes

| Mode | Description |
|------|-------------|
| `start` | Seed or resume session. Drops player at `front_hall`. |
| `look` | Narrate current room from live RM state (+topo, +light, +climate). |
| `go direction=X` | Move through a portal. Auto-narrates arrival. |
| `examine target=X` | Inspect a landmark or entity via the Lens Bus. |
| `map` | Explored rooms list with exit counts. `[*]` = current room. |
| `status` | Session summary: room, facing, moves, rooms visited. |
| `stop` | Save session state and end. |
| `help` | Full mode list, navigation reference, compass point table. |
| `setup` | Commissioning checklist + direct portal setter + north calibration. See Setup section. |
| `tool_manifest` | Self-description via `MF.tool_manifest()`. |

---

## Navigation

Direction input resolves in priority order:

1. **Relative** — `ahead`, `behind`, `left`/`port`, `right`/`starboard` (resolved from current facing bearing)
2. **Compass label** — `N NNE NE ENE E ESE SE SSE S SSW SW WSW W WNW NW NNW`
3. **True bearing** — `0`–`359` (calibrated via `spatial_config.calibration_bearing` in household profile)
4. **Room name / area_id** — fuzzy match on portal `to:` field

Portal matching uses ±22.5° tolerance on bearings. Front door calibration reference: 340° (NNW). Player facing resets to the reverse of the exit bearing on each move.

If multiple portals fall within the same compass bucket (±22.5° of the same bearing), ZenZork asks you to specify by room name rather than picking one silently. Disambiguation message is narrator-aware — DUNGEONMIND consults the portal index, Zork narrator tells you bluntly. Say `go direction=<room name>` to resolve.

---

## State

Game state lives in the AI user cabinet at `zenzork_state`. Fields:

| Field | Description |
|-------|-------------|
| `session_id` | `zz-YYYYMMDDHHMMSS` |
| `player` | Resolved from `primary_user` in system cabinet, or field override |
| `current_room` | area_id of current position |
| `facing_bearing` | True bearing player is facing |
| `visited` | List of explored area_ids |
| `move_count` | Total moves this session |
| `started_at` / `ended_at` | ISO timestamps |

---

## Room narration

`mode=look` (and the implicit look after `start` or `go`) calls Room Manager with `context_slices: +topo,+light,+climate`. Narration output includes:

- Room name and area note (if set)
- Exits: compass bearing, portal type, destination name, `normally:` tag if present
- Landmarks with bearing, zone, and description from RM spatial data
- Light: on/total count, average brightness (Bright / Dim / Very dim / Dark)
- Climate: current temp, HVAC mode
- Facing compass label

If Room Manager is unavailable (`continue_on_error: true`), narration degrades gracefully — room name from area_id, exits from cached topology if available.

---

## Examine

`mode=examine target=X` routes via the Lens Bus (`zen_dojotools_library` with `by_anchor` / `anchor_type: label`). Any registered Lens stack provider is reachable. Target can be a landmark name or entity_id.

---

## Map

`mode=map` reads explored rooms from session state and annotates each with area name and exit count from `room_topology`. Current room marked `[*]`, others `[ ]`.

---

## Prerequisites

- Room Manager topology populated with portal bearings (`zen_dojotools_room_manager mode=set area=X` with `bearing:` on each portal)
- `ai_cabinet` label: entity labeled `zen_default_ai_user_cabinet`
- `hh_cabinet` label: entity labeled `Zen Household Cabinet` (for topology and `spatial_config`)
- `sys_cabinet` label: entity labeled `Zen System Cabinet` (for `primary_user` resolution)

---

## Setup

`mode=setup` serves three functions:

**Commissioning checklist (no args):**
Shows every room in topology with portal commission status: `[OK]` = all portals have bearings, `[  ]` = needs work. Also shows north calibration status.

**North calibration:**
```
mode=setup answer=calibrate=340
```
Sets `spatial_config.calibration_bearing` in the household cabinet. Anchor bearing is the true bearing your front door faces. All compass labels derive from this.

**Direct portal setter:**
```
mode=setup direction=<bearing 0-359> target=<dest_area_id> answer=<room_area_id>
```
Sets one portal entry via Room Manager. `answer=` defaults to current room if omitted.

**Trojan interactive wizard** (play the game, it asks you questions as you find unmapped portals): v1.2 scope.

---

## Quest / Win Conditions

Set a win condition with `mode=quest quest_goal=X`:

| Quest | Description |
|-------|-------------|
| `explore_all` | Visit every room in the topology. Win fires when visited count equals topology room count. |
| `discover_all_landmarks` | Examine every named landmark across all rooms. Win fires when `examined` list covers all landmark names. |
| `reach:<area_id>` | Navigate to a specific room. Win fires on arrival. |

Check quest status: `mode=quest` with no goal. Win is detected automatically on the next `look`/`go` that satisfies the condition. Win state written to `zenzork_state.quest_won`.

---

## Narrator Styles

Set with `narrator=` field (default: `zork`).

| Style | Description |
|-------|-------------|
| `zork` | Dry, sardonic, second-person. The game is unimpressed by you. |
| `dungeon` | DUNGEONMIND — an unhinged dungeon AI that has been running this labyrinth too long. Deeply emotionally invested. Calls your thermostat the Eternal Flame. |
| `straight` | Evidence block only. No flavor. |

The `narrator_prompt` key in every response tells Friday how to narrate from the evidence block. Static flavor (darkness, heat, cold, move milestones) is also baked directly into the `narration` string for conditions the template can evaluate.

---

## Notes

- Queue mode (`max: 5`) — up to 5 concurrent game sessions
- Calibration bearing sourced from `household_profile.spatial_config.calibration_bearing` in the household cabinet. Defaults to 0 if not set.
- Session auto-persists on `go` moves. Explicit `mode=stop` to mark `ended_at`.
- Bare `mode=look` without a prior `mode=start` reads whatever room is in `current_room` from saved state, or defaults to `front_hall` if no session exists.
