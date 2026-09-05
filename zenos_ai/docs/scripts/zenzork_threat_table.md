# ZenZork Threat Table — DUNGEONMIND's Hostile Roster

<!--
  OBFUSCATION NOTICE, READ BEFORE YOU DO ANYTHING RASH:

  The "threat" row below is not the threat. It's md5(lowercase,
  trimmed string) of "THREAT NAME::Warning Text" — the text DUNGEONMIND
  gives you the moment a threat first makes itself known, since that's
  the actual spoiler (what's stalking you, in its own words) — same
  recipe family as the loot/quest tables, same non-security. This is a
  spoiler curtain, not a vault door.

  Same disclaimer as always: crackable, not trying to be otherwise.
  Turn the lights off somewhere you've already earned `in_the_dark`
  and find out the honest way.
-->

Threat stats (HP, AC, damage, XP, save DC) and the rest of the
narration set (attack/hit/miss/kill/escape/consequence text) aren't
part of this curtain — those are mechanical, needed to reason about
combat, and live in plaintext in the engine's own sidecar regardless.
This curtain only hides the "what is it and how does it introduce
itself" reveal.

| Threat hash (md5) |
|---|
| `91db0670583b02a6cb098b76a68e35c4` |

Real threat data — id, stats, and the full narration set — lives in
`packages/zenos_ai/dojotools/.zenzork_quests/threat_defs.json` (the
engine's actual data — that file has to stay plaintext, the engine
reads it every render a threat is active) and in the unlinked,
unhashed answer-key companion doc — ask for it by name if you want it.
