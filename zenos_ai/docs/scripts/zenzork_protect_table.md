# ZenZork Protect Token Table — DUNGEONMIND's Escape Levers

<!--
  OBFUSCATION NOTICE, READ BEFORE YOU DO ANYTHING RASH:

  The "token" row below is not the token. It's md5(lowercase, trimmed
  string) of "Item Name::Flavor Text" for each real entry (the
  DUNGEONMIND-voice flavor specifically) — same recipe family as the
  loot/swag tables, same non-security. This is a spoiler curtain, not
  a vault door.

  Same disclaimer as always: crackable, not trying to be otherwise. Go
  find out what protects you the honest way — carry something into a
  threat encounter and see what fires.
-->

A protect token is a carried item that auto-negates a threat's
consequence once, then is consumed. Even without one, a CON saving
throw is still rolled as a last chance — this curtain only covers the
"what item does this and what does it feel like" reveal.

| Token hash (md5) |
|---|
| `47f52b3e0623399f804d090f47066e21` |

Real token data lives in `packages/zenos_ai/dojotools/.zenzork_quests/
protect_defs.json` (the engine's actual data — that file has to stay
plaintext, the engine checks it automatically the moment a threat
consequence would fire) and in the unlinked, unhashed answer-key
companion doc — ask for it by name if you want it.
