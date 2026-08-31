# ZenZork Devkit — Building on the Engine

For the "what exists and how it works today" reference, see
[`zen_dojotools_zenzork_readme.md`](zen_dojotools_zenzork_readme.md).
This doc is for the next layer: you have a platform, here's how to add
to it without touching the engine's core dispatch logic most of the time.

ZenZork's post-v1.6 content (quests, book-lore chapters, cheat codes)
is deliberately **data-driven**: a small number of reusable "type"
handlers in the script, and the actual content — ids, conditions,
flavor text — lives in JSON sidecars next to the script
(`packages/zenos_ai/dojotools/.zenzork_quests/`). Adding content that
fits an existing type is a data edit. Adding a genuinely new *kind* of
check is a script edit, but a small, contained one.

---

## The sidecar files

| File | Powers | Dispatched by |
|------|--------|----------------|
| `quest_defs.json` | `mode=quest` (13 of 16 markers) | win-check block in the `go`/`look` sequence |
| `book_lore.json` | `mode=chapters`, the Diawata/Valtay/Mongo unlock chain | same win-check block, ordered-sequence variant |
| `genie_codes.json` | `mode=genie` | the genie mode block directly |
| `chapter_releases.json` | `mode=chapters` release/engine-version gating | the chapters block's `publicly_released`/`playable` computation |
| `threat_defs.json` (v1.8.0) | The turn-based threat tracker (grue, etc.) | THREAT ENGINE block, ticked on `look`/`start`/`go`/`wait` |
| `weapon_defs.json` (v1.8.0) | `mode=attack`'s auto-selected weapon lookup | combat resolution block |
| `protect_defs.json` (v1.8.0) | Escape-lever consumable tokens | THREAT ENGINE consequence check |
| `swag_defs.json` (v1.8.0) | Reward items on threat-encounter resolution | THREAT ENGINE outcome block |
| `household_systems.json` (v1.8.0) | "DUNGEONMIND notices a real household system" achievements | shared achievement pipeline (`look`/`start`/`go`/`wait`) |

All of these are loaded via `!include .zenzork_quests/<file>.json` —
same directory as the script itself. **Don't** reach into
`zenos_ai/docs/` or any other subtree with `!include` for engine data;
stick to same-directory sidecars. It's the only path resolution
that's actually been proven to work reliably here — see the
`chapter_releases.json` ledger for why (the human-facing chapter docs
live in `docs/`, but the engine reads a small local ledger instead of
reaching across the tree for a live value).

---

## Adding a quest of an existing type

1. Open `quest_defs.json`. Add an entry to `sequence` (wait — that's
   `book_lore.json`; for quests it's the `quests` array):
   ```json
   {
     "id": "my_new_quest",
     "type": "domain_state_in_room",
     "param": {"domain": "switch", "target_state": "on"},
     "accept_dungeon": "*** QUEST ACCEPTED: ... ***\n...",
     "accept_zork": "...",
     "win_dungeon": "*** QUEST COMPLETE: ... ***\n...",
     "win_zork": "..."
   }
   ```
2. `ha_reload_scripts`. That's it — the win-check dispatcher, the
   accept-message dispatcher, and the win-narration dispatcher all do
   a table lookup by `id`, so a new entry with an existing `type` needs
   zero script changes.
3. Update the help text listing (`dojotools_zenzork.yaml` — the
   `mode=help` Quest Commands section and the compact top-level tool
   description) so players can discover it. This part IS a script
   edit, but it's just adding a line to two doc strings, not logic.

### Full type reference (as of Chapter 1)

| Type | Param shape | Checks |
|------|-------------|--------|
| `reach_room` | `{room}` | `_current_room == param.room` |
| `visit_fraction` | `{fraction}` | visited rooms ≥ ceil(total_rooms × fraction) |
| `entity_state_in_room` | `{domain, suffix or suffixes[], target_state}` | `states(domain + '.' + current_room + suffix) == target_state`, any suffix matches |
| `two_entities_exist_in_room` | `{e1_domain, e1_suffix, e2_domain, e2_suffix}` | both entities exist (not unknown/unavailable) |
| `exterior_exit_in_room` | none | current room's topology has a portal with `exterior:true` AND `exit:true` |
| `room_and_examined` | `{room}` | `current_room == param.room` AND `_gs.examined` non-empty |
| `inventory_nonempty` | none | player has taken ≥1 item |
| `domain_state_in_room` | `{domain, target_state}` | any entity of that domain in current room (via `label_entities`) has that state |
| `domain_state_anywhere` | `{domain, target_states[]}` | any entity of that domain, anywhere, has a state in the list |
| `examine_landmark_type_anywhere` | `{landmark_type}` | any landmark of that type, anywhere in topology, has been examined |

This list is authoritative in `quest_defs.json`'s own `types_reference`
key — keep both in sync if you add a type.

## Adding a genuinely new type

You need this when nothing in the table above expresses the check you
want. In `dojotools_zenzork.yaml`, find the win-check dispatcher (the
big `{%- if _t == 'reach_room' -%} ... {%- elif _t == '...' -%}`
chain, inside the go/look block's `_win_now` computation) and add one
more `{%- elif _t == 'my_new_type' -%}` branch. The accept/win text
stays entirely in the JSON — you're only ever adding the *condition*
logic in the script, never the flavor text. Test on T first, same
discipline as everything else in this repo: config-check, reload,
live-test with seeded achievement data, then port to H.

---

## Adding a book-lore entry (sequence_position, not release_chapter)

The 12-entry sequence in `book_lore.json` is built around the
household's real 8-book library, so it's not really "add more entries
whenever" — it's tied to real content the household owns. If a 9th
book releases, or you want to add a keystone unlock tied to a
different real book event, follow the same shape as the existing
entries: `{id, book, title, format, achievement_name, flavor}`,
inserted at the correct position in `sequence` (order matters — each
entry requires the previous one already earned). `format: "audio"`,
`"either"`, or the special `"all_books_owned"` (used only by
`series_complete`, zero external calls).

If the new entry needs a bespoke extra gate (like `diwatta_unlocked`'s
"24h after audio first detected" rule, which can't be expressed in
the JSON), that's a script-level special case keyed on the entry's
`id` — see how `_diwatta_gate_ok` is computed for the pattern to copy.

**Which release chapter does it belong to?** New book-lore entries
default to whatever release chapter is currently in development — add
the id to that release chapter's `covers` list in
`chapter_releases.json`. Don't create a new release chapter just for
one entry; a release chapter is a whole content DROP (the SoftDisk
model), not a wrapper around a single unlock.

## Starting a new release chapter (Chapter 2, whenever that happens)

A release chapter is the unit `mode=chapters` locks/unlocks as a
bundle — it is NOT the same thing as a `sequence_position` in
`book_lore.json`. Don't confuse the two, and don't create a new public
doc file per book-lore entry; that's the exact mistake this system
briefly shipped with and then un-shipped (see version history).

1. Add a new object to `chapter_releases.json`'s `release_chapters`
   array: `{chapter: 2, released: false, min_engine_version: "X.Y.Z",
   covers: [...]}`. `min_engine_version` matters if the new chapter's
   content needs a new dispatcher `type` this build doesn't have yet
   — set it to whatever version WILL have that type, so the chapter
   can be marked `released` (content revealed) while still correctly
   reporting `playable: false` until the engine actually catches up.
   `mode=chapters`' response surfaces both `released` and
   `engine_ready` per release chapter for exactly this reason — check
   `playable` (the AND of both), not `released` alone, before assuming
   a chapter's content is actually usable.
2. Add the new `sequence_position` entries to `book_lore.json` as
   above, ids matching what you put in `covers`.
3. `zenzork_chapter_tool.py generate` builds `chapter_2.json` (one
   file, whole bundle, locked) from the ledger + `book_lore.json`.
4. `zenzork_chapter_tool.py release 2` flips it open — ALL of Chapter
   2's entries deobfuscate together, same call, same moment. There is
   no per-entry release inside a chapter.

---

## Adding a Game Genie code

`genie_codes.json` — add `{code, type, target, label}`. `type` is
`fast_forward` (marks `target` + everything before it in `book_seq` as
achieved) or `read` (peek only, no achievement). `target` must be a
real id from `book_lore.json`'s sequence. No script changes needed —
the genie dispatcher already handles both types generically.

Codes are meant to be simple and typeable, not cryptographically
meaningful — this is a household toy, not a real cheat device. Keep
them short, keep them thematic if you want, but don't overthink the
"security" of the code string itself.

---

## Adding threat/combat/reward content (v1.8.0)

All four of these are pure data edits for an existing type/shape — no
script change needed unless noted.

**A new threat of an existing type:** add an entry to
`threat_defs.json`'s `threats` array — `{id, type, param, spawn_roll_denominator,
hp, ac, to_hit_bonus, damage_dice, xp_value, enemy_level, save_dc}` plus
the narration string set (`attack_dungeon`/`attack_zork`,
`hit_dungeon`/`hit_zork`, `miss_dungeon`/`miss_zork`,
`kill_dungeon`/`kill_zork`, `warn_dungeon`/`warn_zork`,
`escape_dungeon`/`escape_zork`, `consequence_dungeon`/`consequence_zork`,
`save_success_dungeon`/`save_success_zork`). A genuinely new threat
*type* (grue's is `dark_room_persist`) needs a script-side branch in
the THREAT ENGINE block (`dojotools_zenzork.yaml`, ~2230).

**A new weapon:** add `{item, weapon_class, damage_dice, to_hit_bonus,
damage_bonus, flavor}` to `weapon_defs.json`'s `weapons` array, keyed
by the item's lowercased name exactly as it appears in inventory
(landmark names from room topology, as taken). Anything not listed
still works as an improvised weapon via the `unarmed` fallback entry —
nothing needs to be a "real" weapon to swing it.

**A new protect token:** add `{item, flavor_dungeon, flavor_zork}` to
`protect_defs.json`'s `tokens` array. Checked automatically the moment
a threat's consequence would fire; consumed (removed from inventory)
on use.

**A new swag item:** add to the matching pool (`escape`/`kill`/`defeat`)
in `swag_defs.json` — `{item, flavor_dungeon, flavor_zork}`. Pure
flavor loot, no combat stats of its own. A future `weapon_defs.json`
entry can reference a swag item's name directly if a specific piece of
swag should later become a real weapon.

**A new household-systems achievement:** see the Household-Systems
Achievements section in the readme — pick a real label already
confirmed to have live entities (`zen_dojotools_index dry_run` first,
don't guess), add `{id, label, achievement_name, flavor_dungeon,
flavor_zork}` to `household_systems.json`.

---

## Publishing content (the obfuscation/release convention)

There are **two** obfuscation patterns in this repo, for two different
kinds of content. Don't reach for the wrong one — mixing them up is
exactly the mistake `book_lore.json` briefly shipped with before this
section was written.

### Pattern 1 — irreversible docs curtain (loot, quest, swag, threat, protect tables)

Anything that would spoil the discovery experience if browsed in the
repo, where the engine's own copy can just stay plaintext forever (the
content doesn't carry ongoing narrative weight — a loot roll or a quest
win-blurb reads fine cold), gets: an md5-hashed public doc
(`Name::Flavor` or `Title::Summary`, lowercased, trimmed) with a
REDACTED answer-key companion that reveals content on a schedule (see
`zenzork_loot_table.md`/`zenzork_quest_table.md`/`zenzork_swag_table.md`/
`zenzork_threat_table.md`/`zenzork_protect_table.md` + their answer-key
companions for the exact pattern — the recipe's second field varies
by content shape: loot/swag/protect hash the item name against its
DUNGEONMIND-voice flavor, quest hashes a display title against a win
summary, threat hashes a threat's display name against its `warn_dungeon`
text specifically, not the full narration set, since that's the actual
spoiler). The engine's own sidecar JSON stays fully plaintext in
`packages/` — obfuscation only touches the human-facing docs tree
(`zenos_ai/docs/scripts/`).

**Not every sidecar needs this.** `weapon_defs.json` has a curtain doc
too (`zenzork_weapon_table.md`) but nothing in it yet — its one real
entry (Carl's Left Sock) is landmark-tied content, same precedent as
the retro cartridge collectibles: it requires real household setup to
surface in play, so plaintext in the repo doesn't spoil a discovery
the way a random loot/swag roll does. Judge new sidecars the same way:
does browsing the repo hand someone a discovery they'd otherwise earn
by playing, or does it just describe mechanics/setup? The former gets
curtained, the latter doesn't.

### Pattern 2 — reversible sidecar obfuscation (book-lore only, so far)

`book_lore.json` is different: DUNGEONMIND has to narrate the real
achievement name and flavor text out of it every time a player earns
an entry, forever — there's no point where the engine's copy can be a
one-way hash. It still shouldn't be casually plaintext-readable by
someone browsing the repo either. So it gets BOTH layers:

- **In `packages/`**: `title`/`achievement_name`/`flavor` are
  base64-encoded per entry in `book_lore.json` itself — real content
  (fully reversible, not a hash), just not sitting there in plaintext.
  `id`/`book`/`format` stay plain — they're mechanical, needed to
  reason about the gating logic, not spoilers on their own. The engine
  decodes ALL THREE fields exactly once, right after the `!include`,
  building a plain-string `_book_seq` that every downstream reader
  (unlock dispatcher, `mode=chapters`, `mode=genie`) consumes
  unchanged — decode at load, not at each call site, or you'll end up
  threading `| base64_decode` through a dozen places and someone will
  eventually miss one.
- **In `docs/scripts/`**: same irreversible md5 curtain as Pattern 1
  anyway (`zenzork_book_lore_table.md` +
  `zenzork_book_lore_answer_key_REDACTED.md`) — the docs tree doesn't
  care that the engine's copy is also obfuscated, it curtains the same
  way regardless.

If you're adding a new sidecar and it's unclear which pattern applies:
does the engine need the real value on every future render (ongoing
narration), or does it only need it once at the moment of the event
(a roll, a win)? Ongoing narration → Pattern 2. One-shot → Pattern 1.

---

## Testing discipline (same as everywhere else in this repo)

1. Edit on T first — never H directly.
2. `ha_config_check`, then `ha_reload_scripts`. Reloads land async;
   if a change doesn't seem to have taken effect, wait longer before
   assuming it's a bug — this has produced real false alarms.
3. Live-test with real tool calls. Seed `character_sheet.achievements`
   directly via `zen_dojotools_filecabinet` when you need to test a
   state that's hard to reach organically (a specific point mid-chain,
   a cooldown edge case).
4. Reset any test achievements/game-state you seeded before moving on
   — T is disposable but should still be left in a sane default state.
5. Copy the exact same file to H, config-check, reload there too.
6. Commit on T first, then H, with H's commit referencing T's hash
   (`Mirrors T commit <hash>`).
7. New MCP tool params (new `mode=` values, new fields) won't be
   callable through the MCP interface until its schema cache catches
   up — this is independent of the HA script reload and can lag by a
   noticeable amount. `mode=help` still reflects the reloaded script
   immediately; use that to confirm the script-side change landed
   even when the tool-call interface hasn't caught up yet.
