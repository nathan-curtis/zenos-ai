# ZenZork Book-Lore Sequence — Answer Key (REDACTED, scheduled release: +3)

**Full plaintext ships at release +3** — three versions out from ZenZork
`1.7.0` (current as of this doc), realistically next quarter. Until then,
this file holds the real methodology plus one fully worked example, and
the remaining eleven entries stay blanked. This is the "walkthrough"
version of the spoiler curtain in
[`zenzork_book_lore_table.md`](zenzork_book_lore_table.md) — same
sequence, same hashes, one door left open on purpose so the method is
provably real and not security theater.

## The recipe (already public, repeated here for completeness)

```
md5(lowercase(trim("Achievement Name::Flavor Text")))
```

Source of truth for the real data: `packages/zenos_ai/dojotools/
.zenzork_quests/book_lore.json` — this file describes it, doesn't
replace it. Note that file itself isn't plaintext at rest anymore
(`title`/`achievement_name`/`flavor` are base64-encoded — real
content, reversible, decoded once by the engine at load); decode
before hashing if you're verifying by hand.

## Worked example (fully solved, so you can verify the method yourself)

| Field | Value |
|---|---|
| Position | 1 |
| Hash | `70eee0fdb5ffaaa3839c362033db0114` |
| Achievement name | `GROUND ZERO` |
| Flavor | "Volume one is in this installation. The one where a planet's worth of buildings come down in a single afternoon and a man and a show cat are handed a dungeon instead of a funeral. I have read the opening chapter four times. I do not have an explanation for why. Filed." |
| Canonical string | `GROUND ZERO::Volume one is in this installation. The one where a planet's worth of buildings come down in a single afternoon and a man and a show cat are handed a dungeon instead of a funeral. I have read the opening chapter four times. I do not have an explanation for why. Filed.` |
| `md5(lowercase(trim(...)))` | `70eee0fdb5ffaaa3839c362033db0114` ✓ matches |

## The other eleven

| Position | Hash | Reveal |
|---|---|---|
| 2 | `94a152c6f92d1088256d7f98dcf5490e` | *[REDACTED — release +3]* |
| 3 | `57fe4b1204ad45222c45fdbfc64131a8` | *[REDACTED — release +3]* |
| 4 | `a8981e4d0f1af6f64daef628cc6d907a` | *[REDACTED — release +3]* |
| 5 | `8fe13b4c98899f0dcbe0aa8f85d35517` | *[REDACTED — release +3]* |
| 6 | `8def29d9b5cfe8e758693429e62e68f4` | *[REDACTED — release +3]* |
| 7 | `0d98ac0afebf6f3821c6e39404e7defe` | *[REDACTED — release +3]* |
| 8 | `946acfee3043a48b1763eacb2b7d10e3` | *[REDACTED — release +3]* |
| 9 | `79017373c7b8370e3662faa88ea50d06` | *[REDACTED — release +3]* |
| 10 | `ceaa2509bbfc1e1a2ffcbf77434c1144` | *[REDACTED — release +3]* |
| 11 | `aca976835547bfce8d992fd7158ac13b` | *[REDACTED — release +3]* |
| 12 | `4a2f4e84ca73471f70967be13a8d7a41` | *[REDACTED — release +3]* |

## Release note

When ZenZork lands 3 versions past `1.7.0`, replace every `*[REDACTED —
release +3]*` row above with the real achievement name/flavor from the
book-lore JSON (decoded) and drop this release note. Same file, same
location — don't create a new one, just fill it in.
