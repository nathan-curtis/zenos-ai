# ZenZork Protect Token Table — Answer Key (REDACTED, scheduled release: +3)

**Full plaintext ships at release +3** — three versions out from
ZenZork `1.8.0` (current as of this doc), realistically next quarter.
Until then, this file holds the real methodology plus the one entry
that currently exists, fully solved — there's nothing left to redact
yet, since there's only one protect token right now ("salt circle").
This is the "walkthrough" version of the spoiler curtain in
[`zenzork_protect_table.md`](zenzork_protect_table.md) — same table,
same hash, one door left open on purpose so the method is provably
real and not security theater.

## The recipe (already public, repeated here for completeness)

```
md5(lowercase(trim("Item Name::Flavor Text")))
```

Source of truth for the real data: `packages/zenos_ai/dojotools/
.zenzork_quests/protect_defs.json` — this file describes it, doesn't
replace it. The `flavor_dungeon` field is what's hashed, not
`flavor_zork`.

## Worked example (fully solved — currently the entire table)

| Field | Value |
|---|---|
| Hash | `47f52b3e0623399f804d090f47066e21` |
| Item | `salt circle` |
| Flavor (dungeon) | "*** PROTECT TOKEN CONSUMED: SALT CIRCLE ***\nThe circle activates. Whatever was closing in stops, recalculates, and reconsiders its options. One use. It is now just salt on the floor." |
| Canonical string | `salt circle::*** PROTECT TOKEN CONSUMED: SALT CIRCLE ***\nThe circle activates. Whatever was closing in stops, recalculates, and reconsiders its options. One use. It is now just salt on the floor.` |
| `md5(lowercase(trim(...)))` | `47f52b3e0623399f804d090f47066e21` ✓ matches |

## Release note

When a second protect token ships, add its hash to
`zenzork_protect_table.md` and a corresponding `*[REDACTED — release
+3]*` row here. Redact each token independently 3 versions out from
whichever version it first shipped in, same as the threat table.
