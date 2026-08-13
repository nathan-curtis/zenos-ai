# ZenZork Book-Lore Sequence — DUNGEONMIND's Reading List

<!--
  OBFUSCATION NOTICE, READ BEFORE YOU DO ANYTHING RASH:

  The "reveal" column below is not the reveal. It's md5(lowercase,
  trimmed string) of "Achievement Name::Flavor Text" for each real
  entry — same recipe as the loot table and quest table, same
  non-security. This is a spoiler curtain, not a vault door.

  Same disclaimer as always: this is crackable, nobody's pretending
  otherwise. Go earn these the honest way — read the actual household
  library, let DUNGEONMIND notice.
-->

Twelve entries, one ordered chain — each requires the previous one
already earned before its own check even fires. This is the household's
spoiler-prevention tech as much as it's a reward system: DUNGEONMIND
finds out things in roughly the order a real reader would.

| Position | Reveal hash (md5) |
|---|---|
| 1 | `70eee0fdb5ffaaa3839c362033db0114` |
| 2 | `94a152c6f92d1088256d7f98dcf5490e` |
| 3 | `57fe4b1204ad45222c45fdbfc64131a8` |
| 4 | `a8981e4d0f1af6f64daef628cc6d907a` |
| 5 | `8fe13b4c98899f0dcbe0aa8f85d35517` |
| 6 | `8def29d9b5cfe8e758693429e62e68f4` |
| 7 | `0d98ac0afebf6f3821c6e39404e7defe` |
| 8 | `946acfee3043a48b1763eacb2b7d10e3` |
| 9 | `79017373c7b8370e3662faa88ea50d06` |
| 10 | `ceaa2509bbfc1e1a2ffcbf77434c1144` |
| 11 | `aca976835547bfce8d992fd7158ac13b` |
| 12 | `4a2f4e84ca73471f70967be13a8d7a41` |

Real data — `id`, real book title, achievement name, and flavor text —
lives in `packages/zenos_ai/dojotools/.zenzork_quests/book_lore.json`,
the engine's actual sidecar. Unlike the loot and quest tables, that
file is NOT plaintext at rest: `title`/`achievement_name`/`flavor` are
base64-encoded in place (real content, reversible, the engine decodes
it once at load — see `zenzork_devkit.md`'s Publishing section for why
this one gets a second layer the others don't). The unlinked,
unhashed answer-key companion doc has the real recipe and one worked
example — ask for it by name if you want it.
