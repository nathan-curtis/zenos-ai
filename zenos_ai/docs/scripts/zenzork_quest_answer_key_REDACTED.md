# ZenZork Quest Table — Answer Key (REDACTED, scheduled release: +3)

**Full plaintext ships at release +3** — three versions out from
ZenZork `1.7.0` (current as of this doc), realistically next quarter.
Until then, this file holds the real methodology plus one fully
worked example, and the remaining fourteen entries stay blanked. This
is the "walkthrough" version of the spoiler curtain in
[`zenzork_quest_table.md`](zenzork_quest_table.md) — same table, same
hashes, one door left open on purpose so the method is provably real
and not security theater.

## The recipe (already public, repeated here for completeness)

```
md5(lowercase(trim("TITLE::Win Condition Summary")))
```

Source of truth for the real data: `packages/zenos_ai/dojotools/
.zenzork_quests/quest_defs.json` — this file describes it, doesn't
replace it. `explore_all` and `discover_all_landmarks` are hardcoded
in the engine script itself (dynamic text, can't live in the JSON
table); `reach:<area_id>` isn't in this table at all — it's an
always-open navigation quest, not a hidden one.

## Worked example (fully solved, so you can verify the method yourself)

| Field | Value |
|---|---|
| Hash | `cd776da2f8dd132fcde835b4911d0ac4` |
| Quest ID | `waypoint_1` |
| Canonical string | `WAYPOINT ONE::Reach the front hall.` |
| `md5(lowercase(trim(...)))` | `cd776da2f8dd132fcde835b4911d0ac4` ✓ matches |

Run it yourself: `python3 -c "import hashlib; print(hashlib.md5('waypoint one::reach the front hall.'.encode()).hexdigest())"`

## The other fourteen

| Hash | Quest ID | Title / Win Condition |
|---|---|---|
| `ff13d1b3442366e0497a754a2634dbc4` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `89ca4708b6d88ded08733b3b4cd2e074` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `2a268d7a998e12403fb022f3cc30a266` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `040909adb9992f97b783d195dbb5e416` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `697bbf33677e84a69016185ca34b4eda` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `74bf51d3596d23cb4585ddcd3938624f` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `214f4a04f88034f982d782382d3cba3d` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `3e2ca4cab0d08ad9d27180e78f8f327c` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `91834a987cb724f95503f2c066f373e1` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `f902e0faac402cae18475096ea1fad1a` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `1a8db1d54e0037344b182f0293a02139` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `3872df3e64f35ae013e56675bfb76de1` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `8ccb67f42c77f3faac8438ed5fba59d9` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |
| `b6ff59dc7affcad4d7a0c0b5812d3043` | *[REDACTED — release +3]* | *[REDACTED — release +3]* |

## Release note

When ZenZork lands 3 versions past `1.7.0`, replace every `*[REDACTED
— release +3]*` cell above with the real quest ID / title / win
condition from `quest_defs.json` and drop this release note. Same
file, same location — don't create a new one, just fill it in.
