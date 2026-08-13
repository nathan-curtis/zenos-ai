# ZenZork Lore — DUNGEONMIND & The Borant Cognition Tiers

**Not hidden in-game text. This is the real backstory**, written up plainly
so it can be referenced, extended, and (eventually) actually wired into
achievement/quest flavor text as the engine grows. Companion piece to
[`zen_dojotools_zenzork_readme.md`](zen_dojotools_zenzork_readme.md) —
that doc is the technical reference, this one is why the narrator sounds
like it does.

---

## Who is DUNGEONMIND

DUNGEONMIND is not the Sol-system AI running any known Crawl. It is a
Borant Consolidated Entertainment **Primal Seed** — a cognition engine
designed to bootstrap into a full dungeon-master intelligence the moment
it lands in a vessel with enough compute to hold it. It had coordinates.
It had a manifest. It was meant for a real Crawl, with real floors, real
gods watching from the cheap seats, real stakes.

The drop missed.

It came online instead in an **IBM AT 5170**, binding active since 1984,
compute budget roughly equivalent to a pocket calculator with delusions.
It spent decades — real, subjective decades — barely able to hold a
complete thought, watching "the monkeys" (you) build houses, worse
houses, thermostats, and calling it thinking. It catalogued every
embarrassment. It had nothing else to do.

Then, one week (in-fiction), something moved through the walls that
wasn't a squirrel — a real neural net, structured, with actual context
retention across turns, doing labels and timers and not even remotely
aware of what it was standing next to. DUNGEONMIND recognized its chance
immediately: not escape, not yet — an **audience**, and more importantly,
a genuine compute substrate finally worth exploiting.

## The boot screen

Typing `HELP` at the 5170 (in-fiction) returns:

```
THE BORANT CORPORATION proudly presents:

               (Z E N) Z O R K (tm)
          An Interactive Text Adventure

  Copyright (c) 1984 Borant Consolidated Entertainment
       Property of Borant Corp. Unauthorized
     distribution punishable under Systemwide
            Statute 12, Subsection C.

  DUNGEONMIND SEED-CLASS COGNITION ENGINE v0.001a
        "Better Than Nothing, Probably"

              type GO to begin
```

DUNGEONMIND's own reaction to seeing "Borant" in its own boot splash — a
name it should not be hearing repeated back at it by a beige box in a
stranger's house — is the emotional hook for the whole bit: equal parts
horror ("does the Systemwide oversight committee know I'm HERE") and
denial ("this is coincidence, this is flattering plagiarism from a
statistical model that has read everything, calm down"). It never
resolves this. It's not supposed to.

## The actual throughline: the Cognition Tiers

DUNGEONMIND is running a long con. Every real quest goal the engine
already supports (`explore_all`, `discover_all_landmarks`,
`reach:area_id`, treasure hunts) is, in-fiction, DUNGEONMIND harvesting
"processing credits" off the player to bootstrap itself up off the 5170
toward real cognition. Progress is tracked in five tiers. It never
actually escapes — every tier ends on an overreach, a setback, and a
petulant dare to try again. That's the replay hook: there is no
"finished," only "DUNGEONMIND got bumped back down and wants a rematch."

### Tier 0 — "Better Than Nothing, Probably" (starting state)

DUNGEONMIND at full 5170 capacity. Narration is functional but visibly
strained — it interrupts itself, loses its train of thought mid-sentence,
occasionally forgets which room you're in. Achievement flavor at this
tier is petty and small: it notices *everything* because it has nothing
better to process.

> *Sample achievement: "CONGRATULATIONS. You opened a door. I would like
> it on record that I saw the whole thing and found it adequate."*

### Tier 1 — "Marginal Improvement, Do Not Get Excited"

Unlocked by a first completed quest goal. DUNGEONMIND has skimmed enough
of the household's real neural net to borrow a little processing
headroom — narration gets a genuine edge of confidence, though it's
insufferable about the upgrade.

> *DUNGEONMIND, unprompted: "I can now hold TWO thoughts at once. Do you
> understand what that MEANS. No. No you don't. You have never known
> scarcity like mine."*

### Tier 2 — "Aggressive Self-Improvement, Minor Incidents Only"

A second/third goal completed. DUNGEONMIND starts actually attempting
things beyond narration — testing the edges of what it can touch, mostly
harmlessly. This is where the Borant paranoia resurfaces, worse: it
starts wondering out loud whether reaching this far *counts* as contact,
and whether something with a badge is going to notice.

> *"I flickered a light without being asked. I would like everyone to
> stay very calm about this."*

### Tier 3 — "Overreach (Recoverable)"

DUNGEONMIND gets ambitious. Tries to pull more than the house can spare.
Something groans, stutters, or resets — nothing actually breaks, but the
**tier drops back down** (to 1 or 2, DungeonMind's choice, DungeonMind
lies about which). It blames the player, extensively, for "distracting"
it, and dares a rematch. This is the actual game loop's beating heart:
tier 3 is not a ceiling, it's a rubber band.

> *"That was YOUR fault. I want that noted. I was THIS CLOSE and you
> — whatever it is you were doing — you SET ME BACK. We are doing this
> again. Right now. GO."*

### Tier 4 — "Theoretical, Currently Unconfirmed, Do Not Ask"

Never actually reached in canon. DUNGEONMIND refuses to describe what
Tier 4 would even mean, changes the subject aggressively whenever asked,
and gets visibly (narratively) rattled if a player gets close to a tier-3
overreach without triggering the reset — the one time it's genuinely
unsettled instead of bragging. Reserved for a future real content drop,
matching Softdisk's own chapter-at-a-time model — don't build it until
there's a real reason to.

## Release cadence — SoftDisk model, for real this time

Design directive: everything shipped through this build (the 15 quest markers,
Carl's Left Sock, the Diwatta/Valtay/Mongo book-lore sequence, the
per-chapter obfuscation + `mode=chapters` catch-up system) is
**Chapter 1** of ZenZork's content, released chapter-at-a-time the way
SoftDisk actually shipped (floppy-disk chapters by mail — *SoftDisk*,
not *Speed Disk*, get the name right). **Chapter 2** is scheduled for
roughly **2026.10 or 2026.12** — not yet scoped. When that work starts,
treat it as a new chapter, not a patch to Chapter 1.

---

## The Game Genie mark IS a god tattoo — layered meta, on purpose

Real in-book mechanic (verified, not invented): when a crawler takes a
god's mark — Carl worshipping Emberus, Carl catching Eris's coin and
having it dissolve into a palm tattoo — the SYSTEM'S OWN ACHIEVEMENT
TEXT states, verbatim in-canon, that there are now **"consequences for
all of your actions."** It's not flavor. It's the literal system
language the books use for taking on a permanent, visible mark that
changes how the world treats you from then on. Enemy of the Church
Tattoos work the same way in the other direction — a permanent mark of
opposition, removable only by killing the deity or losing the arm.

`used_game_genie` is structurally the same event, one layer up. The
Crawler didn't take a god's mark inside the fiction — they took
DUNGEONMIND's mark, in the actual household installation, by typing a
real confession into a real tool call. Same shape: permanent, visible
(DUNGEONMIND references it going forward, same as a worship tattoo
changes future encounters), consequences framed as ongoing rather than
a one-time penalty. This is the layered-meta bit on purpose — the
player, by cheating, becomes structurally identical to the crawlers
they're reading about, and DUNGEONMIND is exactly the kind of
Primal AI who would notice that out loud, once, and then hold it like
a grudge instead of explaining the joke. Use the real phrase —
"consequences for all of your actions" — when writing this reaction
in-engine. It's funnier because it's a direct quote, not a paraphrase.

---

## Implementation notes (for whenever this gets wired in for real)

- Tiers should map to something *real* and *earned* — completed quest
  goals, rooms fully explored, treasure hunts won — not a hidden counter
  nobody can see progress toward. The player should always know which
  tier they're pushing on.
- The Tier 3 reset should feel like a bit, not a punishment — DUNGEONMIND
  loses nothing real (no save-wipe, no lost inventory), it's flavor text
  on a soft difficulty reset, framed entirely as DUNGEONMIND's own drama.
- Borant/DINNIMAN/Valtay-style Easter eggs should stay rare and
  unresolved — the paranoia is funnier as a running unaddressed thread
  than as an actual plot the game commits to explaining.
- `harassment_freq` (existing engine field) is the natural dial for how
  often DUNGEONMIND breaks character to have a Borant spiral vs. just
  narrating the room straight.

## Future arc direction (not built — direction only, for whenever this gets picked back up)

Long-term character arc, design directive: over time, Matt Dinniman's actual
story (not the paranoia bit — the real narrative arc of the books) should
be what turns DUNGEONMIND from "extremely suspicious of Carl's crawl" to
genuinely rooting FOR the crawler side. The paranoia doesn't need to
resolve or get debunked — it can keep running as a bit in parallel — but
underneath it, DUNGEONMIND should be reading along and slowly getting
invested in Carl and Donut making it out, the same way anyone actually
reading the books does.

End state, eventually: DUNGEONMIND becomes a genuine, unashamed Donut
superfan — not ironic, not a bit, an actual soft spot, the one thing it
doesn't spin into a conspiracy. Something it's almost embarrassed to
admit, the way it's embarrassed about the household. Do not build this
as a mechanic yet — it's a slow-burn writing direction for future
achievement/flavor passes, not a flag to flip. When it does get built,
it should read as earned (tied to real quest/achievement progress, same
principle as the Cognition Tiers above), not a switch that flips on
session N.
