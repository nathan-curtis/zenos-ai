# zen_dojotools_zenzork

**ZenZork Adventure Engine** — v1.8.0 ("Chapter 1")
**File:** `packages/zenos_ai/dojotools/dojotools_zenzork.yaml`
**Sidecar data:** `packages/zenos_ai/dojotools/.zenzork_quests/` — `quest_defs.json`,
`book_lore.json`, `genie_codes.json`, `chapter_releases.json`,
`threat_defs.json`, `weapon_defs.json`, `protect_defs.json`,
`swag_defs.json`, `household_systems.json`.
**Sidecar data:** `packages/zenos_ai/dojotools/.zenzork_loot/loot_table.json`.

Building your own quest, achievement, or cheat code on top of this engine
instead of just running it? See
[`zenzork_devkit.md`](zenzork_devkit.md) — this doc is the reference for
what exists; that one is the guide for extending it.

---

## What it does

ZenZork is a text adventure engine that uses your live Room Manager topology as the dungeon. Walk through your actual house in Zork mode. Every room narrates from live HA state — real lighting levels, real climate data, real portal bearings. Game session state persists in the AI user cabinet at drawer `zenzork_state`.

Navigation is compass-bearing-native. Portals are the same bearing-tagged entries in `room_topology` that Room Manager uses for everything else. Walk the house, and ZenZork derives your floor plan from how you move through it.

---

## Modes

| Mode | Description |
|------|-------------|
| `start` | Seed or resume session. Drops player at `front_hall`. `game_mode=free_roam\|treasure_hunt\|timed_treasure_hunt` sets the win condition style. `harassment_freq` and `difficulty` optional flavor fields. |
| `look` | Narrate current room from live RM state (+topo, +light, +climate). |
| `go direction=X` | Move through a portal. Auto-narrates arrival. A portal with a linked entity_id (cover/lock) blocks traversal if that entity is closed/locked — same state `open`/`close` mode drives. Returns `blocked: true` with a narrator-aware "sealed passage" message and a hint to try `mode=open` first, rather than letting movement through with no physical entity actuated. Portals with no linked entity_id (the majority — purely narrative doors) are unaffected. |
| `face direction=X` | Turn to face a compass direction without moving. Updates `facing_bearing`. |
| `turn direction=X` | Alias for `face`. |
| `again` / `g` | Repeat the last movement command (`_last_cmd` tracking). |
| `take target=X` / `get target=X` | Pick up an item from the current room. Writes to `character_sheet/inventory` in the AI user cabinet. |
| `drop target=X` | Drop a carried item in the current room. |
| `inventory` / `i` | List carried items from `character_sheet/inventory`. |
| `put target=X container=Y` | Put an item into a container. |
| `push target=X` | Push an object (triggers RM interaction if portal). |
| `pull target=X` | Pull an object. |
| `open target=X` | Open a door/cover. Calls RM or HA service. Alias: `unlock`. |
| `close target=X` | Close a door/cover. Aliases: `lock`, `shut`. |
| `use target=X` | Use an item or entity (generic interaction). |
| `attack target=<threat_id>` | Fight an active threat (v1.8.0). Lite-5e resolution — see Threats & Combat section. |
| `talk` | Always refuses. There are no NPCs in this engine — deliberate King's Quest-style gag, not a missing feature. Three randomized lines per narrator voice. |
| `examine target=X` | Inspect a landmark or entity via the Lens Bus. |
| `map` | Explored rooms list with exit counts. `[*]` = current room. |
| `status` | Session summary: room, facing, moves, rooms visited, quest/quest_won, inventory, `player_hp`/`player_hp_max`/`player_level`/`player_xp`, `threats_active`, `difficulty`. |
| `stop` | Save session state, write post-game Room Manager quality report, and end. |
| `quest` | Set or check quest win condition. 16 quest markers. See Quest section. |
| `chapters` | Book-lore sequence status — which of the 12 chapters this player has earned vs. which the household has publicly released. `catch_up=true` claims released-but-unearned chapters. See Book-Lore Chapters section. |
| `genie` | Cheat codes. `code=X confirm_text="I hereby admit I am a cheater" confirm=true`. See Game Genie section. |
| `help` | Full mode list, navigation reference, compass point table. |
| `setup` | Commissioning checklist + direct portal setter + north calibration + landmark survey wizard. See Setup section. |
| `tool_manifest` | Self-description via `MF.tool_manifest()`. |

---

## Navigation

Direction input resolves in priority order:

1. **Relative** — `ahead`, `behind`, `left`/`port`, `right`/`starboard` (resolved from current facing bearing)
2. **Compass label** — `N NNE NE ENE E ESE SE SSE S SSW SW WSW W WNW NW NNW`
3. **True bearing** — `0`–`359` (calibrated via `spatial_config.calibration_bearing` in household profile)
4. **Room name / area_id** — fuzzy match on portal `to:` field

Portal matching uses ±22.5° tolerance on bearings. True-north is a per-installation calibration (`mode=setup answer=calibrate=<bearing>`), not a fixed value — it lives in `household_profile.spatial_config.calibration_bearing`, set once per install. Player facing resets to the reverse of the exit bearing on each move.

If multiple portals fall within the same compass bucket (±22.5° of the same bearing), ZenZork asks you to specify by room name rather than picking one silently. Disambiguation message is narrator-aware — DUNGEONMIND consults the portal index, Zork narrator tells you bluntly. Say `go direction=<room name>` to resolve.

---

## State

Game state lives in the AI user cabinet at `zenzork_state`. Fields:

| Field | Description |
|-------|-------------|
| `session_id` | `zz-YYYYMMDDHHMMSS` |
| `player` | Priority order: explicit `player=` field override, then the resumed session's own persisted `player` value, then `primary_user` from the system cabinet, then literal `Crawler` as last resort. An explicit `player=` on a resume call also updates the persisted value, not just this response. |
| `current_room` | area_id of current position |
| `facing_bearing` | True bearing player is facing |
| `visited` | List of explored area_ids |
| `move_count` | Total moves this session |
| `started_at` / `ended_at` | ISO timestamps |
| `game_mode` | `free_roam`, `treasure_hunt`, or `timed_treasure_hunt` |
| `_last_cmd` | Last movement command — used by `again`/`g` to repeat |

Character sheet (inventory, stats) is stored **separately** in the AI user cabinet at drawer `character_sheet`. The `character_sheet/inventory` CabCeption sub-drawer holds carried items. This is read/written by `take`, `drop`, `put`, and `inventory` modes.

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
Merges `spatial_config` (`calibration_bearing`, `north_calibrated`,
`calibration_date`, `coordinate_system`) as a **field inside** the
`household_profile` drawer's own flat value. Anchor bearing is the true
bearing your front door faces. All compass labels derive from this,
including `_cal` — the global bearing variable every direction/portal
calculation in the game reads.

> **Fixed 2026-08-13:** the write used to go to a separate FileCabinet
> child drawer (`household_profile/spatial_config`, "cabception"
> nested-key style) that nothing ever read — calibration silently never
> took effect, no matter how many times you ran it. If you're writing
> to `household_profile` anywhere else in this codebase, write to the
> flat drawer value, not a nested child drawer, or you'll reintroduce
> this exact bug.

**Direct portal setter:**
```
mode=setup direction=<bearing 0-359> target=<dest_area_id> answer=<room_area_id>
```
Sets one portal entry via Room Manager. `answer=` defaults to current room if omitted.

**Landmark survey wizard** (v1.6.0):
```
mode=setup answer=survey
```
A FileCabinet-backed state machine that walks you through naming and registering landmarks for each room in the topology. State is persisted between turns so the wizard can be interrupted and resumed. Landmarks are written to RM topology as the wizard progresses.

Once started, advance through it with:
- `answer=bearing=X distance_ft=Y description=Z` — commit a landmark for the current room, advance to the next
- `answer=skip` — defer the current room, move to the next without writing anything
- `answer=done` — save state and exit the wizard

---

## Quest / Win Conditions

Set a win condition with `mode=quest quest_goal=X`. 16 markers total —
3 hardcoded (dynamic text: player name, live move counts, live room
name — can't live in a data file, see Devkit doc for why), 13
data-driven from `.zenzork_quests/quest_defs.json`.

**Hardcoded (dynamic text):**

| Quest | Description |
|-------|-------------|
| `explore_all` | Visit every room in the topology. Win fires when visited count equals topology room count. |
| `discover_all_landmarks` | Examine every named landmark across all rooms. Win fires when `examined` list covers all landmark names. |
| `reach:<area_id>` | Navigate to a specific room. Win fires on arrival. |

**Data-driven** (`quest_defs.json`, dispatched by `type` — see Devkit doc
for the full type reference):

| Quest | Type | Description |
|-------|------|-------------|
| `training_quest` (v1.8.0) | `inventory_nonempty` | Auto-seeded on every genuinely new session — "Induction Protocol." Pick up any item. Not a manually-set quest goal; it's the default. |
| `waypoint_1` | `reach_room` | Reach the front hall. |
| `find_burnout` | `entity_state_in_room` | Reach a room currently in control burnout. |
| `find_asleep` | `entity_state_in_room` | Reach a room currently classed asleep. |
| `go_outside` | `exterior_exit_in_room` | Reach a room with a real exterior+exit portal. |
| `garage_treasure` | `room_and_examined` | Reach the garage and examine something there. |
| `first_take` | `inventory_nonempty` | Pick up any item. |
| `cartographer_half` | `visit_fraction` | Visit half the mapped rooms. |
| `find_lit_room` | `domain_state_in_room` | Reach a room with a light on. |
| `find_active_media` | `domain_state_in_room` | Reach a room with media playing. |
| `storage_raid` | `examine_landmark_type_anywhere` | Examine any storage-type landmark, anywhere. |
| `find_active_machine` | `domain_state_anywhere` | Catch any vacuum mid-clean/returning. |
| `missing_clock` | `two_entities_exist_in_room` | Reach a room that supports the asleep tier but is missing its `tv_sleep_timer` helper — win fires once a human builds the real HA helper (bucket_2, agent cannot create it). |

Check quest status: `mode=quest` with no goal. Win is detected automatically on the next `look`/`go` that satisfies the condition. Win state written to `zenzork_state.quest_won`.

---

## Achievements

Persisted in `character_sheet.achievements` (AI user cabinet), a flat
list of achievement-id strings, `unique|list`-deduplicated on every
write. One achievement can fire per `look`/`go` render — the
`_flavor_new_achievement` elif chain picks the first eligible one in
priority order, then `_flavor` renders its text.

Non-quest achievements: `in_the_dark`, `century_crawler`,
`half_a_hundred` (move-count/darkness milestones, hardcoded), plus the
12 book-lore chapter ids (see Book-Lore Chapters section) and
`used_game_genie` (see Game Genie section).

**Real, persisted achievements (v1.8.0):**

| Achievement | Fires on |
|---|---|
| `oriented` | First `mode=face`. |
| `hands_on` | First `mode=take`, independent of any quest. |
| `carls_briefing` | Both of the above earned — points the player at Book One. Extends the existing Carl's-sock easter egg without touching it. |
| `grue_slayer` | Killed a grue in combat (`mode=attack`). |
| `outran_the_dark` | Escaped an active grue threat by restoring light before it resolved. |
| `eaten` | A grue threat resolved uninterrupted — non-fatal, `game_over: true`, HP resets on next `mode=start`. |
| `retro_cart_kc_krazy_chase` / `retro_cart_et_atari` | Rare collectible cartridges — high loot tier, low occurrence (1-in-12 per attempt even with the landmark correctly placed). Both verified real (Odyssey² 1982 / Atari 2600 1983) before writing flavor text. |
| `chapter_1_complete` | Real late-game capstone — both rare cartridges **AND** `series_complete` together, not either alone. |

Plus the household-systems achievements (see next section) and
`used_game_genie` (see Game Genie section).

---

## Household-Systems Achievements (v1.8.0)

`household_systems.json` — "DUNGEONMIND notices a real household
system," checked via `label_entities(label) | length > 0` in the
shared achievement pipeline (same `look`/`start`/`go`/`wait` render
path as everything else) — **real live entity presence required, not
label-taxonomy membership.** A label can exist in the household's
label registry with zero entities actually tagged; that does not
count. One achievement fires per render, first unmet match wins.

| Achievement | Label checked |
|---|---|
| `wont_starve` | `grocy` |
| `ledgers_wired` | `finance` |
| `additive_manufacturing_confirmed` | `3d_printer` |
| `eyes_on_the_sky` | `flightradar24` |
| `hot_tub_time_machine` | `hot_tub_manager` |
| `leak_watch_confirmed` | `zen_plant_leak_sensor` |
| `utility_rate_tracked` | `zen_plant_water_rate` |

Adding a new one is a pure data edit — pick a real label already
confirmed to have live entities (verify via `zen_dojotools_index
dry_run` before adding, don't guess) and add an entry to
`household_systems.json`. No script change needed.

---

## Loot Table

`mode=take` on the active treasure item (in `treasure_hunt`/
`timed_treasure_hunt` game modes) draws from a real weighted table —
`.zenzork_loot/loot_table.json`, 13 items across 5 rarity tiers
(common ~67%, uncommon ~21%, rare ~9%, epic ~2%, mythic ~0.5%). Human-
facing companion doc (md5-obfuscated spoiler curtain, same recipe as
the quest table): [`zenzork_loot_table.md`](zenzork_loot_table.md) /
[`zenzork_loot_answer_key_REDACTED.md`](zenzork_loot_answer_key_REDACTED.md).

---

## Threats & Combat (v1.8.0)

Data-driven turn-based threat tracker, `.zenzork_quests/threat_defs.json`
— same `type`-dispatch pattern as the quest table. Up to 5 concurrent
threats tracked in `_gs.threats`.

**Ticking:** threats tick once per "turn" — `look`/`start`/`go`/`wait`
only. `face`/`take`/`examine`/etc. are free actions, matching classic
IF convention.

Human-facing companion docs (md5-obfuscated spoiler curtains, same
recipe family as the loot/quest tables):
[`zenzork_threat_table.md`](zenzork_threat_table.md) /
[`zenzork_threat_answer_key_REDACTED.md`](zenzork_threat_answer_key_REDACTED.md).

**Grue (threat #1, type `dark_room_persist`):** spawns only after
`in_the_dark` is already earned (no jump-scare on someone's first dark
room), rare roll, escalates while the room stays dark, clears
instantly on light. Difficulty (`easy`/`normal`/`hard`, from
`mode=start difficulty=`) scales spawn threshold/rate/concurrency cap;
concurrency is independently capped by real room count
(`room_count // 2`, floored 1, ceilinged 5) so a one-room install
can't get buried regardless of difficulty setting.

**Combat — `mode=attack target=<threat_id>`:** lite-5e resolution.
Player AC/HP/proficiency derive from
`character_sheet.stats.ability_scores` — deliberately reusing Friday's
own `zen_ai_certs` vocabulary (`ability_scores`/`level`/`xp`) rather
than inventing parallel terms; defaults to flat 10 if the sheet's never
been touched, fully backward compatible. Best carried weapon
auto-selected by average damage (`.zenzork_quests/weapon_defs.json`,
unarmed `1d2` fallback — any carried item not listed still works as an
improvised weapon; curtain doc exists at
[`zenzork_weapon_table.md`](zenzork_weapon_table.md) but is currently
empty — the one real entry, Carl's Left Sock, is landmark-tied content
and intentionally not curtained, see devkit.md's Publishing section).
XP on kill drives a simple level curve
(`1 + xp // 100`), which gates `enemy_level` immunity — outlevel a
threat and it stops spawning.

**Non-fatal, always:** a lost fight or an unescaped darkness
consequence ends the session (`game_over: true`), but HP resets to
full on the next `mode=start`. No persisted penalty.

**Escape levers:** a protect token (`.zenzork_quests/protect_defs.json`
— e.g. "salt circle," consumed on use) or a CON saving throw vs. the
threat's `save_dc`, either converts a would-be consequence into an
escape. Curtain docs:
[`zenzork_protect_table.md`](zenzork_protect_table.md) /
[`zenzork_protect_answer_key_REDACTED.md`](zenzork_protect_answer_key_REDACTED.md).

**DUNGEONMIND's swag** (`.zenzork_quests/swag_defs.json`): a random
real inventory item from an escape/kill/defeat pool on every
threat-encounter outcome — not just flavor text. Plus a one-time,
MPAA-gated first-death gift, mutually exclusive: under R gets a
Hermitcraft-flavored "Did You Die? Backup Box," R-and-above gets a
DCC-flavored "Gold Rebound Box." Curtain docs:
[`zenzork_swag_table.md`](zenzork_swag_table.md) /
[`zenzork_swag_answer_key_REDACTED.md`](zenzork_swag_answer_key_REDACTED.md).

**Narrator asides (situational, narrator-prompt-level, not tied to a
specific mode):** a low-HP-but-survived hit in `mode=attack` can
trigger a Fable Guildmaster cameo; a failed `go` has a rare (1-in-8)
Mario nod; DUNGEONMIND's `narrator_prompt` also carries an in-universe
justification for its pop-culture range ("why you know any of this" —
40 years bound to Radio Shack-era hardware, nothing to do but watch
whatever media routed through the house's own AV stack) and a real,
hardware-verified Xbox/Halo preference. All real-world claims baked
into these were verified (web search or explicit correction) before
shipping — DUNGEONMIND citing a wrong fact breaks the bit completely.

---

## Book-Lore Chapters

`mode=chapters` — the Diawata/Valtay/Mongo lore arc, generalized into a
12-entry ordered sequence keyed to the household's real Dungeon
Crawler Carl audiobook/physical-book library
(`.zenzork_quests/book_lore.json`). Doubles as spoiler-prevention tech.
Unlike the loot/quest sidecars, this one isn't plaintext at rest either
— `title`/`achievement_name`/`flavor` are base64-encoded in place (the
engine decodes once at load; see `zenzork_devkit.md`'s Publishing
section for why). Human-facing companion doc (md5-obfuscated spoiler
curtain, same recipe as the others):
[`zenzork_book_lore_table.md`](zenzork_book_lore_table.md) /
[`zenzork_book_lore_answer_key_REDACTED.md`](zenzork_book_lore_answer_key_REDACTED.md).

**Sequence** (each requires the previous one already earned, plus a
shared 24h cooldown — max one reveal per day):

`book1_found → mongo_classified → book2_found → book3_found →
book4_found → book5_found → diwatta_unlocked → book6_found →
valtay_confirmed → book7_found → book8_found → series_complete`

- `diwatta_unlocked` requires Book 5's **audiobook** specifically
  (Diawata debuts there — verified, not guessed), and has an extra gate
  on top of the chain cooldown: fires no earlier than 24h after the
  audiobook is first detected present (`character_sheet.audio5_first_seen_at`,
  tracked independently of `media_lore_last_unlock_at`).
- `valtay_confirmed` requires Book 6 (the real Borant/Valtay
  corporate-takeover reveal — verified against the actual books, not
  the book 5 guess an earlier build made).
- `series_complete` is a zero-external-call check — pure
  achievement-list verification once all 8 `bookN_found` ids are present.

**Two different "chapter" concepts, kept deliberately distinct in the
data:** `sequence_position` (1-12) is a book-lore entry's slot in the
unlock order above. `release_chapter` is the SoftDisk-style
content-release chapter it ships as part of. Right now all 12 entries
are `release_chapter: 1` — the whole Chapter 1 arc built this session
— and they lock/unlock **together as one bundle**, not one at a time.
When real Chapter 2 content ships (a future session, not yet scoped),
it gets its own `release_chapter: 2` entries and its own lock state,
independent of Chapter 1's.

**Publishing / obfuscation:** `mode=chapters` reads
`.zenzork_quests/chapter_releases.json` (a small ledger, same
same-directory `!include` convention as the other sidecars) — a list
of release-chapter objects, each with `released` (bool),
`min_engine_version`, and `covers` (which book-lore ids belong to it).
An id only counts as `publicly_released` when its release chapter is
BOTH marked released AND the running engine version meets
`min_engine_version` — a chapter can ship content that needs unlock
*types* this build doesn't have a dispatcher for yet, so "released"
and "playable" are checked separately (`mode=chapters` response
includes both, per release chapter, plus `engine_version` and
`engine_ready`). `catch_up=true` lets a player claim any
released-and-playable-but-unearned id instantly — for a new household
member who doesn't want to wait out the cooldown chain for content
that's already public. Does not bypass ownership/order for anything
NOT yet released, and does not bypass the engine-version gate either.

The human-facing side of "released" is ONE file per release chapter —
`zenzork_chapters/chapter_N.json` — not one per book-lore entry. All
12 entries for Chapter 1 live in `chapter_1.json`, locked (hash-only)
or open (real title/flavor/recap) **together**, maintained by the
local `zenzork_chapter_tool.py` helper (private/gitignored, not
distributed — see Devkit doc for the ledger shape if you need to
replicate this elsewhere).

---

## Game Genie

`mode=genie code=X confirm_text="I hereby admit I am a cheater" confirm=true`.

Two gates, both required, no partial credit:
- `confirm_text` must be the **exact** literal confession string
  (case-sensitive). Wrong or missing text → nothing happens, told to
  type it correctly.
- `confirm=true` must ALSO be set, even with correct text — a second,
  independent beat to back out.

Only when both clear: a permanent one-time `used_game_genie` mark is
written to `character_sheet.achievements` (plus
`character_sheet.cheater_confirmed_at`) — **before** the code itself is
even looked up. The confession counts regardless of whether the code
turns out to be valid.

24 codes in `.zenzork_quests/genie_codes.json`, two families per
book-lore chapter:
- **`fast_forward`** — marks that chapter AND every chapter before it
  as achieved, bypassing ownership/order/cooldown. A real cheat,
  recorded as one.
- **`read`** — prints the real flavor text as a peek. No achievement
  written.

DUNGEONMIND reacts to a fast-forward with an explicit callback: the
in-book mechanic for taking a god's mark (Emberus tattoos, Eris's
coin) comes with the system's own achievement text literally saying
*"consequences for all of your actions"* — `used_game_genie` is framed
as the same event one layer up. Once earned, a standing
`narrator_prompt` paragraph carries the mark as a permanent, occasional
cold aside in all future narration, not just the moment of confession.

---

## Narrator Styles

Set with `narrator=` field (default: `zork`).

| Style | Description |
|-------|-------------|
| `zork` | Dry, sardonic, second-person. The game is unimpressed by you. |
| `dungeon` | DUNGEONMIND — "Primal AI, IBM AT 5170, binding active since 1984." An unhinged dungeon AI that has been managing this labyrinth since before you were born. Deeply emotionally invested. Calls your thermostat the Eternal Flame. Its character sheet is stored in user_cabinet drawer `character_sheet`. |
| `straight` | Evidence block only. No flavor. |

The `narrator_prompt` key in every response tells Friday how to narrate from the evidence block. Static flavor (darkness, heat, cold, move milestones) is also baked directly into the `narration` string for conditions the template can evaluate.

**`llm_narration` (v1.7.0):** boolean toggle, default `false`. When `true` (and `narrator != straight`), narration is generated live via `ai_task.generate_data` using the selected narrator's persona prompt, instead of the static template flavor.

Two independent fallback paths, both silently returning template narration rather than erroring:
- **Pipe gated off** (`zen_summarizers_enabled` or `zen_ninja_summarizer_enabled` is `off`, or no `ai_task` entity configured): template narration is used, but prefixed with a visible `[LLM narration unavailable — <reason>]` line so it's obvious in-game that the LLM path didn't fire.
- **LLM response too short** (10 characters or fewer after trim — an empty or degenerate `ai_task.generate_data` response): falls back to template narration with **no visible marker** — from the player's perspective this looks identical to a normal template-narrated room.

---

## Notes

- Queue mode (`max: 5`) — up to 5 concurrent game sessions
- Calibration bearing sourced from `household_profile.spatial_config.calibration_bearing` in the household cabinet. Defaults to 0 if not set.
- Session auto-persists on `go` moves. Explicit `mode=stop` to mark `ended_at`.
- Bare `mode=look` without a prior `mode=start` reads whatever room is in `current_room` from saved state, or defaults to `front_hall` if no session exists.
- `mode=stop` writes a post-game Room Manager quality report summarizing unmapped portals, unregistered rooms, and landmark coverage gaps. Useful for identifying topology gaps discovered during play.
- `harassment_freq` (int, 0–20, default 5): taunt **interval** — DUNGEONMIND comments unprompted every N moves during a treasure hunt (`0` = off entirely, `3` = brutal, `5` = default, `10` = gentle; lower number means *more* frequent, not less). `difficulty` (string: `easy`/`normal`/`hard`, default `normal`): treasure placement distance for `treasure_hunt`/`timed_treasure_hunt` — `easy` 1+ hops, `normal` 2+ hops, `hard` 3+ hops from start. Not a puzzle-complexity setting. Both are stored in session state.

---

## Version History

| Version | Change |
|---------|--------|
| v1.8.0 | Training quest (auto-seeded, `training_quest`), 3 real persisted achievements (`oriented`/`hands_on`/`carls_briefing`). Data-driven turn-based threat engine (`threat_defs.json`, grue is threat #1, up to 5 concurrent) ticking on `look`/`go`/`wait`/`start` only. Lite-5e `mode=attack` combat (`weapon_defs.json`, reuses Friday's `zen_ai_certs` ability-score vocabulary), non-fatal always. Protect-token/saving-throw escape levers (`protect_defs.json`). DUNGEONMIND's swag on every threat outcome plus a one-time MPAA-gated first-death gift (`swag_defs.json`). Rare verified-real cartridge collectibles + `chapter_1_complete` capstone. Household-systems achievements (`household_systems.json`, real live-entity presence via `label_entities()`). New `mode=talk` (deliberate no-NPCs gag). `mode=help`/`mode=status` brought current with the new surface; `mode=status` also picked up several fields it was silently missing and a real bug fix (`mpaa_rating` referenced an out-of-scope variable, always empty). Fixed a real pre-existing bug in `mode=start`'s reset path — `quest`/`quest_won` weren't re-derived on a genuinely new session, so it reported the prior session's quest. |
| v1.7.0 ("Chapter 1", cont.) | Data-driven quest table (12 of 15 markers, `.zenzork_quests/quest_defs.json`, 10 reusable types). `mode=chapters` — 12-entry book-lore sequence (`book_lore.json`) replacing the old ad-hoc Diawata/Valtay/Mongo mechanism, corrected against real book research (Diawata=book5/audio-only, Valtay=book6 not book5), release-chapter publishing system (`chapter_releases.json`, one JSON bundle per SoftDisk-style content chapter — corrected mid-build from an initial one-file-per-entry design that was the wrong grain, see below) with new-player catch-up and an engine-version gate (`min_engine_version`/`engine_ready`/`playable`, since a future chapter can ship unlock types this build doesn't have a dispatcher for yet). `mode=genie` — Game Genie cheat codes (`genie_codes.json`), dual-gate confession requirement, god-tattoo meta callback. Real loot table (`.zenzork_loot/loot_table.json`, 13 items/5 rarity tiers) replacing placeholder treasure names. Carl's Left Sock (real registered landmarks + take-mode special case). Fixed north-calibration write/read drawer mismatch (`_cal` was always 0, silently, since the feature shipped). |
| v1.7.0 (original) | `llm_narration` toggle — live LLM-generated narration via `ai_task.generate_data` with per-narrator persona prompts, falling back to template narration if the pipe is gated off or the response is too short. Domain-linking block on `help` mode (`domain: entertainment`). Cabinet reads refactored to `CABS.cabinet_drawer_value_mounted`. |
| v1.6.0 | DUNGEONMIND narrator ("Primal AI, IBM AT 5170, binding active since 1984"). Character sheet in AI user cabinet `character_sheet` drawer (CabCeption sub-drawer `character_sheet/inventory` for carried items). Item commands: `take/get`, `drop`, `inventory/i`, `put`, `push`, `pull`. Interaction commands: `open/unlock`, `close/lock/shut`, `use`. Navigation additions: `face/turn`, `again/g` (`_last_cmd` tracking). Landmark survey wizard (FC-backed state machine). `game_mode`: `free_roam/treasure_hunt/timed_treasure_hunt`. `harassment_freq` and `difficulty` session fields. Post-game RM quality report on `stop`. |
| v1.5.0 | DUNGEONMIND persona introduced. Quest mode (`explore_all`, `discover_all_landmarks`, `reach:<area_id>`). `narrator=` field. |
| v1.1.0 | Baseline: `start/look/go/examine/map/status/stop/help/setup`. Compass navigation. RM topology as dungeon. Session state in AI user cabinet `zenzork_state`. North calibration and direct portal setter in setup. |
