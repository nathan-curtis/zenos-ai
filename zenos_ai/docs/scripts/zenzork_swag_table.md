# ZenZork Swag Table — DUNGEONMIND's Consolation Prizes

<!--
  OBFUSCATION NOTICE, READ BEFORE YOU DO ANYTHING RASH:

  The "item" column below is not the item. It's md5(lowercase, trimmed
  string) of "Item Name::Flavor Text" for each real entry (the
  DUNGEONMIND-voice flavor specifically — same recipe as the loot
  table, same non-security. This is a spoiler curtain, not a vault
  door.

  Same disclaimer as always: crackable, not trying to be otherwise.
  Go trigger a threat encounter and find out the honest way.
-->

One item, randomly picked from the matching pool, lands in your
inventory whenever a threat encounter resolves — `escape` (saved via
protect token or saving throw), `kill` (threat defeated in combat), or
`defeat` (knocked out or eaten — same non-fatal "you died" case, same
pool). Pure flavor loot, no combat stats of its own.

| Pool | Item hash (md5) |
|---|---|
| escape | `c4c4a2c42999e06c59c8793d4b13e0a9` |
| escape | `b098e2005deb46d8e93db8c0940d9433` |
| escape | `b4acc9515fae872fbf6f99b0f1a47782` |
| kill | `8c341190da4eaf77dc92784dddd4e50c` |
| kill | `e2963a32e70bf6440d68740c4dbe1935` |
| kill | `6351b1de2437a10ed50cd22de04e274f` |
| defeat | `7af8f4188e2eeac4cc366b67eac71acc` |
| defeat | `434101297c4a0fefe893129ad14a00cf` |
| defeat | `f3b9fd0caef3d11f80ca83afcab428b2` |

Real item names and flavor text live in `packages/zenos_ai/dojotools/
.zenzork_quests/swag_defs.json` (the engine's actual data — that file
has to stay plaintext, the engine reads it on every threat resolution)
and in the unlinked, unhashed answer-key companion doc — ask for it by
name if you want it.

Not covered by this curtain: the one-time, MPAA-gated first-death gift
(a "Did You Die? Backup Box" or "Gold Rebound Box," depending on your
household's content rating) — that's a single fixed reveal, not a
random pool, and it's already described in the readme.
