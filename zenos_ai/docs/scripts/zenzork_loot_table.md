# ZenZork Loot Table — DUNGEONMIND's Chest

<!--
  OBFUSCATION NOTICE, READ BEFORE YOU DO ANYTHING RASH:

  The "item" column below is not the item. It's md5(lowercase, trimmed
  string) of "Canonical Item Name::Flavor Text" for each real entry.
  The real names/flavor live in a second file DUNGEONMIND has not
  admitted the location of yet.

  Yes, this is crackable. MD5 has been broken for actual security
  purposes since roughly forever, and this isn't even trying to be
  secure — it's a spoiler curtain, not a vault door. If you really want
  to, you can absolutely brute-force or dictionary-attack these thirteen
  32-character strings and know every item before you ever roll one.

  This is the "flip to the back of the sudoku book" option. This is
  "take the rubik's cube apart with a hammer and reassemble it solved"
  energy. Nobody is stopping you. But consider, seriously, for a moment:
  who wins that? You already know the answer. You knew it before you
  read this comment. Go roll the table like a person.
-->

Weighted like a Minecraft chest loot table — rarity tier determines the
pool, weight determines the odds within it. Roll across the table's full
weight total (215, the sum of every item's `weight` — not a fixed 1-100
scale), walk down the cumulative weight, land on an item, reveal it
(i.e., actually go look it up) only after the roll — don't pre-read the
answer key, that's the whole point of the obfuscation.

| Rarity | Weight | Item hash (md5) |
|---|---|---|
| common | 40 | `0ebe6c1124992746e149b1cf5e425da2` |
| common | 40 | `d2706ba0beed3e89c372be12f327f238` |
| common | 35 | `0d3b3bd6573a838a458691737ee310a1` |
| common | 30 | `ffb525d9875463e14c466be23a2667b0` |
| uncommon | 18 | `3ffd5b5080e06456351ba2a9fc691d7f` |
| uncommon | 15 | `6c0376dac5d8bdc0d26067a8fd05bda5` |
| uncommon | 12 | `250f075034fedd05814e6cbeeaeb7cad` |
| rare | 8 | `ce8f29e948538d006c61c5a4490ee849` |
| rare | 6 | `9636d802eca5ab64f5d293defbdc7c11` |
| rare | 5 | `920fda58befd7cc11c8d63c1e92b926c` |
| epic | 3 | `d6fb7d2440026fcfe3fefca40bd024b9` |
| epic | 2 | `28221a3f69630bd359ec9cef05c699a3` |
| mythic | 1 | `f66364ca17b39cd99df029bff6343366` |

**Tier odds** (sum of weights within tier / total): common 145/215
(~67%), uncommon 45/215 (~21%), rare 19/215 (~9%), epic 5/215 (~2%),
mythic 1/215 (~0.5%) — genuinely Minecraft-shulker-box rare.

Real names and flavor text exist in a second, unlinked file. Ask for it
by name if you want it — this file is deliberately not the spoiler.
