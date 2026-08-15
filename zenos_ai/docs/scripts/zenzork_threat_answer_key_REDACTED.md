# ZenZork Threat Table — Answer Key (REDACTED, scheduled release: +3)

**Full plaintext ships at release +3** — three versions out from
ZenZork `1.8.0` (current as of this doc), realistically next quarter.
Until then, this file holds the real methodology plus the one entry
that currently exists, fully solved — there's nothing left to redact
yet, since there's only one threat in the roster right now (`grue`).
This is the "walkthrough" version of the spoiler curtain in
[`zenzork_threat_table.md`](zenzork_threat_table.md) — same table,
same hash, one door left open on purpose so the method is provably
real and not security theater.

## The recipe (already public, repeated here for completeness)

```
md5(lowercase(trim("THREAT NAME::Warning Text")))
```

Source of truth for the real data: `packages/zenos_ai/dojotools/
.zenzork_quests/threat_defs.json` — this file describes it, doesn't
replace it. The `warn_dungeon` field is what's hashed (the moment a
threat first announces itself), not the rest of the narration set —
stats and combat text are mechanical, not curtained anywhere, see the
table doc for why.

## Worked example (fully solved — currently the entire roster)

| Field | Value |
|---|---|
| Hash | `91db0670583b02a6cb098b76a68e35c4` |
| Threat ID | `grue` |
| Warning text (`warn_dungeon`) | "*** THREAT DETECTED: GRUE (STALKING) ***\nSensor gap confirmed, this chamber. No photon count, no motion, no thermal — an absence of data shaped exactly like something that eats people in the dark. It has not moved on you yet. It will. Find a light switch, Crawler. I am not being dramatic. I am being thorough." |
| Canonical string | `GRUE::*** THREAT DETECTED: GRUE (STALKING) ***\nSensor gap confirmed, this chamber. No photon count, no motion, no thermal — an absence of data shaped exactly like something that eats people in the dark. It has not moved on you yet. It will. Find a light switch, Crawler. I am not being dramatic. I am being thorough.` |
| `md5(lowercase(trim(...)))` | `91db0670583b02a6cb098b76a68e35c4` ✓ matches |

## Release note

When a second threat ships, add its hash to `zenzork_threat_table.md`
and a corresponding `*[REDACTED — release +3]*` row here, same shape
as the loot/quest/swag answer keys. When ZenZork lands 3 versions past
whatever version a given threat first shipped in, replace that
threat's redacted row with the real data and drop the note for that
entry specifically — threats redact independently, not as one batch,
since they won't all ship in the same version.
