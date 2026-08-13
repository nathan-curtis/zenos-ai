# ZenZork Quest Table — DUNGEONMIND's Objectives

<!--
  OBFUSCATION NOTICE, READ BEFORE YOU DO ANYTHING RASH:

  The "quest" column below is not the quest. It's md5(lowercase,
  trimmed string) of "TITLE::Win Condition Summary" for each real
  quest_goal — same recipe as the loot table, same non-security. This
  is a spoiler curtain, not a vault door.

  Yes, this is crackable. Fourteen 32-character strings, a small
  vocabulary of plausible titles, and a text editor is all it takes.
  This is the "flip to the back of the sudoku book" option again. Same
  question as last time: who wins that?

  Go set a quest and find out what it wants the honest way:
  `zen_dojotools_zenzork mode=quest quest_goal=<id>` — the tool tells
  you what you accepted immediately. This table is for people who
  want the DISCOVERY, not the ID list.
-->

Fourteen of ZenZork's quest markers are catalogued here — the
fifteenth, `reach:<area_id>`, is an always-open navigation quest
(pick any mapped room, go there), not a hidden objective, so it isn't
part of the curtain.

| Quest hash (md5) |
|---|
| `89ca4708b6d88ded08733b3b4cd2e074` |
| `2a268d7a998e12403fb022f3cc30a266` |
| `cd776da2f8dd132fcde835b4911d0ac4` |
| `040909adb9992f97b783d195dbb5e416` |
| `697bbf33677e84a69016185ca34b4eda` |
| `74bf51d3596d23cb4585ddcd3938624f` |
| `214f4a04f88034f982d782382d3cba3d` |
| `3e2ca4cab0d08ad9d27180e78f8f327c` |
| `91834a987cb724f95503f2c066f373e1` |
| `f902e0faac402cae18475096ea1fad1a` |
| `1a8db1d54e0037344b182f0293a02139` |
| `3872df3e64f35ae013e56675bfb76de1` |
| `8ccb67f42c77f3faac8438ed5fba59d9` |
| `b6ff59dc7affcad4d7a0c0b5812d3043` |

Real quest IDs, titles, and win conditions live in
`packages/zenos_ai/dojotools/.zenzork_quests/quest_defs.json` (the
engine's actual data — that file has to stay plaintext, the engine
reads it every render) and in the unlinked, unhashed answer-key
companion doc — ask for it by name if you want it.
