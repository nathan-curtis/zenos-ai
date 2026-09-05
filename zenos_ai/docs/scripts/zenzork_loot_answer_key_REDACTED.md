# ZenZork Loot Table — Answer Key (REDACTED, scheduled release: +3)

**Full plaintext ships at release +3** — three versions out from ZenZork
`1.7.0` (current as of this doc), realistically next quarter. Until then,
this file holds the real methodology plus one fully worked example, and
the remaining twelve entries stay blanked. This is the "walkthrough"
version of the spoiler curtain in
[`zenzork_loot_table.md`](zenzork_loot_table.md) — same table, same
hashes, one door left open on purpose so the method is provably real and
not security theater.

## The recipe (already public, repeated here for completeness)

```
md5(lowercase(trim("Canonical Item Name::Flavor Text")))
```

Source of truth for the real data: `packages/zenos_ai/dojotools/
.zenzork_loot/loot_table.json` — this file describes it, doesn't replace
it.

## Worked example (fully solved, so you can verify the method yourself)

| Field | Value |
|---|---|
| Hash | `0ebe6c1124992746e149b1cf5e425da2` |
| Rarity | common |
| Weight | 40 |
| Canonical string | `Slightly Suspicious Cold Air Return Grate::You are almost certain this was mentioned in a spatial config file. You are correct.` |
| `md5(lowercase(trim(...)))` | `0ebe6c1124992746e149b1cf5e425da2` ✓ matches |

Run it yourself: `python3 -c "import hashlib; print(hashlib.md5('slightly suspicious cold air return grate::you are almost certain this was mentioned in a spatial config file. you are correct.'.encode()).hexdigest())"`

## The other twelve

| Rarity | Weight | Hash | Item |
|---|---|---|---|
| common | 40 | `d2706ba0beed3e89c372be12f327f238` | *[REDACTED — release +3]* |
| common | 35 | `0d3b3bd6573a838a458691737ee310a1` | *[REDACTED — release +3]* |
| common | 30 | `ffb525d9875463e14c466be23a2667b0` | *[REDACTED — release +3]* |
| uncommon | 18 | `3ffd5b5080e06456351ba2a9fc691d7f` | *[REDACTED — release +3]* |
| uncommon | 15 | `6c0376dac5d8bdc0d26067a8fd05bda5` | *[REDACTED — release +3]* |
| uncommon | 12 | `250f075034fedd05814e6cbeeaeb7cad` | *[REDACTED — release +3]* |
| rare | 8 | `ce8f29e948538d006c61c5a4490ee849` | *[REDACTED — release +3]* |
| rare | 6 | `9636d802eca5ab64f5d293defbdc7c11` | *[REDACTED — release +3]* |
| rare | 5 | `920fda58befd7cc11c8d63c1e92b926c` | *[REDACTED — release +3]* |
| epic | 3 | `d6fb7d2440026fcfe3fefca40bd024b9` | *[REDACTED — release +3]* |
| epic | 2 | `28221a3f69630bd359ec9cef05c699a3` | *[REDACTED — release +3]* |
| mythic | 1 | `f66364ca17b39cd99df029bff6343366` | *[REDACTED — release +3]* |

## Release note

When ZenZork lands 3 versions past `1.7.0`, replace every `*[REDACTED —
release +3]*` row above with the real name/flavor from the loot table
JSON and drop this release note. Same file, same location — don't create
a new one, just fill it in.
