# ZenZork Swag Table — Answer Key (REDACTED, scheduled release: +3)

**Full plaintext ships at release +3** — three versions out from
ZenZork `1.8.0` (current as of this doc), realistically next quarter.
Until then, this file holds the real methodology plus one fully worked
example, and the remaining eight entries stay blanked. This is the
"walkthrough" version of the spoiler curtain in
[`zenzork_swag_table.md`](zenzork_swag_table.md) — same table, same
hashes, one door left open on purpose so the method is provably real
and not security theater.

## The recipe (already public, repeated here for completeness)

```
md5(lowercase(trim("Item Name::Flavor Text")))
```

Source of truth for the real data: `packages/zenos_ai/dojotools/
.zenzork_quests/swag_defs.json` — this file describes it, doesn't
replace it. The DUNGEONMIND-voice `flavor_dungeon` field is what's
hashed, not `flavor_zork`.

## Worked example (fully solved, so you can verify the method yourself)

| Field | Value |
|---|---|
| Pool | escape |
| Hash | `c4c4a2c42999e06c59c8793d4b13e0a9` |
| Item | `Certificate of Non-Consumption` |
| Flavor (dungeon) | "DUNGEONMIND produces a certificate from somewhere. It is warm, like it was just printed. 'CERTIFICATE OF NON-CONSUMPTION,' it reads, 'awarded for surviving contact with an unlit sensor gap.' You are not sure where the printer is. Neither, it turns out, is DUNGEONMIND." |
| Canonical string | `Certificate of Non-Consumption::DUNGEONMIND produces a certificate from somewhere. It is warm, like it was just printed. 'CERTIFICATE OF NON-CONSUMPTION,' it reads, 'awarded for surviving contact with an unlit sensor gap.' You are not sure where the printer is. Neither, it turns out, is DUNGEONMIND.` |
| `md5(lowercase(trim(...)))` | `c4c4a2c42999e06c59c8793d4b13e0a9` ✓ matches |

## The other eight

| Pool | Hash | Item / Flavor |
|---|---|---|
| escape | `b098e2005deb46d8e93db8c0940d9433` | *[REDACTED — release +3]* |
| escape | `b4acc9515fae872fbf6f99b0f1a47782` | *[REDACTED — release +3]* |
| kill | `8c341190da4eaf77dc92784dddd4e50c` | *[REDACTED — release +3]* |
| kill | `e2963a32e70bf6440d68740c4dbe1935` | *[REDACTED — release +3]* |
| kill | `6351b1de2437a10ed50cd22de04e274f` | *[REDACTED — release +3]* |
| defeat | `7af8f4188e2eeac4cc366b67eac71acc` | *[REDACTED — release +3]* |
| defeat | `434101297c4a0fefe893129ad14a00cf` | *[REDACTED — release +3]* |
| defeat | `f3b9fd0caef3d11f80ca83afcab428b2` | *[REDACTED — release +3]* |

## Release note

When ZenZork lands 3 versions past `1.8.0`, replace every `*[REDACTED
— release +3]*` cell above with the real item name / flavor from
`swag_defs.json` and drop this release note. Same file, same location
— don't create a new one, just fill it in.
