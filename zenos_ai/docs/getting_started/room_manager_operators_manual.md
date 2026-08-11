<img src="images/cover_boxart.png" alt="ZenOS-AI Home Systems presents Room Manager — retro box art" width="480">

# ZENOS-AI HOME SYSTEMS presents: ROOM MANAGER

### Operator's Manual

> "Your devices know where they are. Now your rooms know what they're doing."

**Model:** Room Manager v3 &nbsp;•&nbsp; **Engine:** REFLEX &nbsp;•&nbsp; **Players:** 1 household &nbsp;•&nbsp; **Skill level:** Set & forget

> KEEP THIS MANUAL FOR FUTURE REFERENCE. It contains important
> operating instructions and troubleshooting information you will
> need if you ever wonder "why did the office lights just do that."

---

## TABLE OF CONTENTS

1. [Before You Begin](#1-before-you-begin)
2. [What Is Room Manager, And Why Is It Here](#2-what-is-room-manager-and-why-is-it-here)
3. [How It Sees Your House](#3-how-it-sees-your-house)
4. [The Nine States of a Room](#4-the-nine-states-of-a-room)
5. [The Control Panel (your room_control_manager switch)](#5-the-control-panel-your-room_control_manager-switch)
6. [Special Moves (opt-in features)](#6-special-moves-opt-in-features)
7. [REFLEX: Turning a State Into a Scene](#7-reflex-turning-a-state-into-a-scene)
8. [Patterns & Practices (playing it well)](#8-patterns--practices-playing-it-well)
9. [Hints, Tips & Tricks](#9-hints-tips--tricks)
10. [Troubleshooting](#10-troubleshooting)
11. [What's Next](#11-whats-next)
12. [How to Request New Features](#12-how-to-request-new-features)
- [For the Technically Curious](#for-the-technically-curious)
- [Addendum: For Agents Reading This Doc](#addendum-for-agents-reading-this-doc)

---

## 1. BEFORE YOU BEGIN

You don't need to read this whole manual to enjoy Room Manager. In fact,
if Room Manager has already been installed in your home, it's already
running. Every configured room has been quietly watching, thinking, and
reacting since the day it was turned on.

This manual is for the curious. The kind of person who reads the back of
the cereal box. If that's you, welcome. Grab a controller.

**No batteries required. No cartridge to insert.** Room Manager lives
inside your Home Assistant system already. You're not installing
anything by reading this.

---

## 2. WHAT IS ROOM MANAGER, AND WHY IS IT HERE

Picture your house before Room Manager: every light, every vent fan,
every "should the TV shut off now" decision was either (a) something
*you* had to do by hand, forever, or (b) a pile of separate automations
that didn't talk to each other and didn't agree on what a "room" even
was.

Room Manager fixes this by giving **every room in your house a single
source of truth about what it's doing right now.** Not "is the light
on." Not "did a sensor fire." The actual, plain-English answer to
*"what is happening in the office at this exact moment?"* It updates
that answer instantly whenever something real changes.

Once a room knows what it's doing, everything downstream gets simple.
Scenes fire themselves. Fans turn on and off on their own schedule.
Left-on TVs go dark. Nobody has to remember to do anything, because the
room already knows.

> **Think of it like this:**
>
> **Old way:** "if motion sensor = on, turn on light" — a pile of these, one per room, forever.
>
> **New way:** "this room IS occupied" — one sensor says so, everything else reads that ONE answer and reacts.

---

## 3. HOW IT SEES YOUR HOUSE

Every room that's been set up gets **one status sensor.** Think of it as
that room's own little scoreboard. You can look at it any time, ask
your AI, or peek at it directly in Home Assistant, and it will tell you
in one word what that room believes about itself right now.

It figures this out by watching a handful of signals for that room:
motion, presence, media playback, doors, whatever's been wired up. It
combines them using a fixed set of house rules (Section 4).

Room Manager derives its state from configured signals. It does not ask
the AI to guess. If nothing in a room is tagged to tell it about motion,
that room simply never reports "Occupied" from motion. It's not broken.
It just has nothing to say on that subject.

**Rooms can also talk to each other.** A bathroom attached to a bedroom
can tell the bedroom "someone's active in here," which keeps the bedroom
from going fully dark and quiet while someone's brushing their teeth at
2am. This is opt-out, not opt-in. It's on by default for any room that's
connected to a parent room.

---

## 4. THE NINE STATES OF A ROOM

<img src="images/states_ladder.png" alt="Priority ladder: Emergency, Paused, Automation, Cleaning, Asleep, Engaged, Hold, Occupied, Vacant — higher wins" width="210" align="right">

Every room is always in exactly one of nine states. Higher on this list
always beats lower. If two things are true at once, the room reports the
more important one.

| State | What it means |
|---|---|
| 🚨 **Emergency** | Smoke, CO, water leak, or an alarm. Wins, always, even over Paused. Safety is never silenced. |
| ⏸ **Paused** | A human hit the brakes. Only a human can release it. The room ignores automation while this is set. |
| 🤖 **Automation** | Your AI (or another automation) has taken the wheel for this room on purpose, temporarily. |
| 🧹 **Cleaning** | The robot vacuum is in here right now. |
| 😴 **Asleep** | Someone's asleep in here. Beats Engaged: a device staying active nearby never overrides a real Asleep signal in the same room. |
| ⚡ **Engaged** | Someone is ACTIVELY using this room. Media is playing, a desk is in active use. Stronger than just "someone's here," but not stronger than the room's own Asleep. |
| ⏳ **Hold** | One of exactly three lines is currently high: the wasp flag (motion with no door-open to confirm entry, only for rooms opted in — see Section 6), Entertaining mode (room opted in), or Guest mode (room opted in). Whichever one is true, the room reports Hold — it stays Hold exactly as long as that source stays true, clears the moment it goes false. Never a timer. |
| 🟢 **Occupied** | Somebody's in here. Presence detected, no specific activity known. |
| ⚪ **Vacant** | Nobody's in here. The default. Nothing wrong with it. Most rooms are Vacant most of the day. |

**A note on "why does the room think that?"** Most states can tell you
why they're active, including which exact sensor caused it and when.
Just ask your AI "why is the office engaged right now?" and it can pull
the real answer, not a guess. (Vacant is the one exception: it's the
default when nothing else is true, so there's no specific sensor to
point at.)

---

## 5. THE CONTROL PANEL (your room_control_manager switch)

Some rooms have a control switch. Think of it as the room's own override
lever, sitting right there on the dashboard. It includes AUTO, plus
every state from Section 4. The four positions you'll use or see most
often are:

| AUTO | PAUSED | AUTOMATION | CLEANING |
|---|---|---|---|
| Normal operation | Stop all automation for this room until a human flips it back | "I've got this room" until released or a human takes it back | Vacuum is working here right now (set for you, don't touch it) |

<img src="images/control_panel.png" alt="Room Control panel: Auto, Paused, Automation, Cleaning" width="500">

**AUTO is the only position that means "figure it out normally."** Every
other position, whether it's one of the four above or a state picked
directly from Section 4, forces the room's state and overrides whatever
the sensors are saying. Think of AUTO as "let the room decide" and
everything else as "I've decided for it."

**PAUSED** is your stop button. Flip it and the room ignores automation
entirely until *you*, specifically, flip it back. Anything, even your
AI, is allowed to pause a room if something odd is happening, but only a
human can un-pause it. That's on purpose. (Paused shows up both as a
control panel position and as a state in Section 4; they're the same
thing seen from two sides, "I picked this" and "this is what the room is
now.")

**AUTOMATION** means an agent has deliberately taken over this room for
a task, such as running a scene sequence or testing something, and will
hand it back when done. Some rooms have a built-in safety net (Section
6) that automatically hands it back if it's left in this state too long
and forgotten.

**CLEANING** is set automatically the moment your vacuum rolls into that
room, and cleared the moment it leaves. You never need to touch this one
yourself.

**Manually setting Occupied, Engaged, Asleep, Hold, or Vacant** is also
available on the same switch, pick any of them directly and the room
holds exactly that until you change it back, no automatic clock
involved. This is the "put this room to sleep until I say otherwise"
button: pick Asleep and it stays Asleep regardless of what any sensor
sees, until someone picks something else.

---

## 6. SPECIAL MOVES (opt-in features)

Not every room needs every feature. Turn these on room by room, only
where they make sense. A garage doesn't need a TV sleep timer, and a
closet doesn't need a vent fan.

**⏱ Control Burnout** — "Don't let a room get stuck in Automation forever." If a room is handed to Automation and nobody releases it within the timer window, it snaps back to Auto by itself. A safety net, not a nag.

**📺 TV Sleep Timer** — "Falling asleep with the TV on? Not anymore." While a room is Asleep, media playing starts (or restarts) a countdown. When it hits zero, the TV shuts off and the room's normal sleep timer takes over.

**🌬 Vent Fan Auto On/Off** — "The fan turns itself on shortly after you walk in, and stays on a little while after you leave." No switch-flipping required.

**🌙 Nightlight** — "A dim, gentle scene fires if you get up in the night," then quietly reverts once you settle back down without ever fully waking the room's Asleep state.

**🌃 Sleep Window** (on by default, per room) — "A room only falls asleep automatically at night." A bed sensor tripping at 2pm (someone folding laundry on the bed, say) won't put the room to sleep. Automatic sleep only fires between night and wake. Can be turned off per room if you want round-the-clock auto-sleep, or automatic sleep can be disabled entirely for a room (manual Asleep, Section 5, always still works either way).

**🕐 Autosleep Schedule** — "No standard sleep pattern? No problem." Shift worker, firefighter, anyone whose "night" isn't actually at night — this room doesn't have to follow the house's clock at all. Point it at a calendar, an HA Schedule helper, or a toggle you flip yourself instead, and that becomes this room's own private night, completely replacing the house-wide clock check for it. Everybody else's rooms keep working off the normal night-to-wake window; this one just marches to its own schedule.

**😴 Asleep Hold** — "Sleep now — no bed sensor required." Tag a toggle, calendar, or schedule, and while it reads on, the room simply IS Asleep, full stop — no clock, no waiting on a sensor to catch up, no window check. Flip it back off (or let the calendar event end) and the room lets go instantly, no lingering.

**🍸 Entertaining Hold / Guest Hold** — "While entertaining mode or guest mode is on, opted-in rooms stay conservative instead of guessing." Prevents a busy house from flickering a room between Occupied and Vacant. Opt in per room; a room not opted in is unaffected.

**🐝 Wasp (Hold from an unconfirmed entry)** — "Motion with no door-open to explain it holds the room, instead of guessing." Requires TWO things before it does anything: at least one door tagged `wasp_door` for that room (Section 8's "the door, not the lock" tip), AND the room itself opted in with the `wasp_enabled` label — either on the room's Area directly, or on any entity in that room. Both, not just one. This is opt-in on purpose: a room with an always-open archway instead of a real door (a connected front hall, say) can't safely tell "someone's inside with the door shut" from "there is no door" — tagging `wasp_door` there without also enabling the room would just misfire. If a room's Hold never seems to trigger from motion alone, check both halves are actually set before assuming something's broken. (See Section 9's "the door, not the lock" tip for the other common wasp_door setup mistake.)

None of it requires a code editor or a YAML file — every feature above
turns on the same way: labels and helpers, done through the normal HA
UI. Here's how.

**Why you're doing this at all:** we can't see your house until you
tell us where things are. This is that telling — once, per room. There's
no ongoing homework: describe a room here and it's described for good.
Every feature this system ships from now on reads the same labels you
already set, so a new feature down the road costs you a label, not
another setup pass.

1. **Tag a sensor as a signal.** Settings → Areas, Labels & Zones → Labels. Create a label matching the signal type you need (`motion`, `occupied`, `engaged`, `asleep`, `bed_occupancy`, `hold`, `wasp_door`, etc.) if it doesn't already exist. Then open the sensor itself (Settings → Devices & Services → Entities, find it, click in), and add TWO labels to it: the signal type AND the room's own label (which should already match the room's Area name). Both labels, same entity, every time.
2. **Create a helper** (timer, latch, etc.). Settings → Devices & Services → Helpers → "+ Create Helper". Pick the type (Timer, Number, Toggle/Boolean, Select). Give it whatever name you like. Once created, go tag it the same way as Step 1: the class label (`room_timer`, `emergency_latch`, `asleep_minutes`, etc.) plus the room's own label.
3. **Deploy the room's state sensor** (once per room). Settings → Automations & Scenes → Blueprints tab → find "ZenOS Room Manager v3: Room State" → the ⋮ menu → Create. Fill in: Room (pick the Area), Friendly Name (e.g. "Office State"), Unique ID (e.g. "office_state"), and Trigger Entities: EVERY entity you tagged in Steps 1 and 2 for this room, listed explicitly. Miss one and it just won't react when that entity changes; nothing breaks, it's simply blind to that one signal.
4. **Save, then reload.** After creating or editing anything above, Settings → System → Restart, or use a YAML reload if you know which domain changed. When unsure, a full restart always picks everything up.

That's it: no code editor, no YAML file, just labels, helpers, and one
blueprint form per room. Everything past Step 3 (Control Burnout, TV
Sleep Timer, Vent Fan, Nightlight, Sleep Window, Autosleep Schedule,
Asleep Hold, Entertaining/Guest Hold) is pure Step 1/Step 2 labeling, no
separate blueprint needed. The dispatcher that reacts to all of it is
already running in the background and needs no per-room setup of its own.

If you have an AI assistant hooked up, you can also just describe what
you want — "turn on control burnout for the office" — and let it do
the labeling for you. Same four steps either way, it's just doing the
clicking instead of you.

---

## 7. REFLEX: TURNING A STATE INTO A SCENE

Everything in Sections 1–6 gets you a room that *knows* what it's doing —
Occupied, Asleep, Hold, whatever. Knowing isn't doing. **REFLEX is the
engine that actually reacts** — the thing that takes "the office just
became Asleep" and turns it into "the office's lights actually dim and
the TV actually turns off." It's the "Engine" named on this manual's
cover for a reason: it's a separate system from state-tracking, and it
has its own on/off switch.

**REFLEX is off by default, house-wide, even once every room's
state-tracking is fully working.** This is one switch for the whole
house, not a per-room setting — every room can know exactly what it's
doing, with zero risk of any scene firing anywhere, until you flip
REFLEX on. Ask your AI "is reflex on?" or "turn reflex on" any time.
(Per-room *wiring* — which scenes belong to which room, described
below — is separate and can absolutely be done room by room ahead of
time; it just won't actually fire anything until the house-wide switch
is on.)

### How REFLEX picks a scene

REFLEX doesn't guess which scene to fire. It looks for a scene labeled
to match the room's new state, using the exact same label-tagging
mechanism as everything else in Section 6:

1. **Label a scene with the state it belongs to.** A scene meant to fire
   when a room goes Asleep gets the label `scene_asleep`. Occupied gets
   `scene_occupied`. Same pattern for `scene_vacant`, `scene_engaged`,
   `scene_cleaning`, `scene_emergency`, `scene_checking`, `scene_nightlight`,
   and so on — one label per state, matching Section 4's state list.
2. **Label the same scene with the room.** Exactly like tagging a sensor
   in Section 6 Step 1 — the scene needs both labels: which state it's
   for, and which room it belongs to.
3. **Optionally, label it for a specific time of day.** If you want the
   office's Occupied scene to look different at 7am than at 9pm, create
   two scenes, both labeled `scene_occupied` and the room, but each also
   labeled for a daypart (morning, daytime, evening, night, etc.). A
   scene with no daypart label is the room's fallback for that state, any
   time of day. A scene WITH a daypart label wins over the fallback,
   only during that daypart.

Once a scene carries state + room (+ optionally daypart), REFLEX finds
it automatically the instant the room changes state. Nothing to wire by
hand beyond the labels — same "label it once, it just works" pattern as
everything else in this manual.

**Haven't gotten around to wiring every state for every room?** REFLEX
quietly borrows a sensible scene instead of doing nothing. A room without
its own Engaged scene borrows its Occupied one. Cleaning and Emergency
do the same. Asleep, if unwired, borrows the room's Vacant scene (lights
out is the safe default for "asleep, but nobody's built a dedicated
scene for it yet"). Wire the real one whenever you get to it — the
borrowed scene is a placeholder, not a requirement you're stuck with.

### Setting it up — what actually happens

Ask your AI: *"wire up scenes for the office"* — but here's what's
really going on behind that sentence, so you know what to expect instead
of treating it as a magic button:

1. **It scans for scenes that already exist and look like they belong
   to this room** — matching by scene name and by the room's own Area
   assignment. This is a MATCH step, not a CREATE step: it can only find
   scenes you (or whoever built your dashboard) already made in Home
   Assistant's own Scene editor. **If no scene exists for a room yet,
   nothing gets found, full stop** — REFLEX can't invent a scene, and
   "ask your AI" can't either. A brand-new or rarely-used room (a
   closet, a utility space) often genuinely has zero scenes, and that's
   fine — see Section 8, not every room needs one.
2. **For every state, it reports one of four things:** already wired
   directly, covered by borrowing another state's scene (fine, see
   above), a genuine gap (a state with no scene and nothing to borrow
   from, worth fixing), or "doesn't apply to this room" (Checking,
   Paused, Automation — these are always safe no-ops, not gaps).
3. **If a found scene's name is ambiguous about which state or time of
   day it's for** ("Late Night" could mean "this room, empty, at night"
   or "this room, occupied, at night"), it gets flagged instead of
   guessed at. That's a real decision only you can make — what should
   the room actually look like in that situation — so expect to be
   asked, not just told it's done.
4. **Nothing is written until you confirm.** Everything above is a
   preview: which scenes matched, what's covered, what's ambiguous, what
   has nothing to wire at all. Confirming is the one moment state
   actually gets written onto the scene's labels.

So the realistic outcomes of asking your AI to wire a room are: it just
does it (scenes existed, matches were unambiguous), it asks you one or
two questions first (matches existed but which-state-is-this was
genuinely unclear), or it comes back and tells you there's nothing to
wire yet because no scene exists for this room — at which point the
next step is building one in Home Assistant, not asking again.

### Testing before you trust it

Flip **dry run** on (ask your AI — one more house-wide switch, same as
REFLEX itself) and REFLEX resolves everything exactly like it would
live, for every room — same scene picked, same logic — but logs what it
WOULD do instead of actually firing it. Once you've checked the log and
you're happy, flip dry run off and REFLEX on, and it does the real
thing. If REFLEX is ever switched on for real while dry run is also on,
the real fire wins — dry run only matters while REFLEX itself is off.

> **REFLEX vs. state-tracking, in one sentence:** state-tracking always
> runs and always tells the truth about the room, REFLEX only acts on
> that truth once you've told it to, and dry run lets it rehearse
> without an audience.

---

## 8. PATTERNS & PRACTICES (playing it well)

**Trust the state, don't fight it.** If a room says Occupied and you
think it's wrong, the fix is almost always "the sensor that should be
telling this room about presence isn't wired up yet," rather than "the
system is broken." Check what's tagged for that room (Settings →
Devices & Services → Entities, filter by the room's label) — or ask
your AI to check for you.

**Use PAUSED liberally, use it locally.** If a room is misbehaving,
pausing that one room costs you nothing and doesn't touch the rest of
the house. There is no reason to disable the whole system just to quiet
down one room.

**Let CLEANING be automatic.** Never manually flip a room's control to
Cleaning. It's meant to track the real vacuum, and manually setting it
will make the room think a robot is in there when it isn't.

**Not every room needs everything.** A hallway probably just wants basic
Occupied/Vacant. A bedroom wants the full stack: Asleep, nightlight, TV
sleep timer. Match the features to how the room is actually used.

**Ensuite rooms cascade up on purpose.** Linking a bathroom to its
bedroom as a child room sets up a parent/child relationship: whatever
the child reports gets hardwired in as true for that same slot in the
parent. Net effect — your bathroom can hold the bedroom's lights on
so it doesn't go dark while you're brushing your teeth. But if you get
up in the middle of the night and the bathroom motion trips Occupied,
your partner can stay asleep in their favorite sleep scene, because
Asleep sits higher on the ladder than Occupied — the bedroom's own
direct Asleep state always wins regardless of what the child room is
reporting.

### Worked examples: setting up common room types

Here's Section 6's Steps 1/2 recipe applied to the room types people
actually set up. Each one lists what to tag by hand; the "Say" line is
the equivalent shortcut if you're handing it to an AI instead.

**🛏 A bedroom** — Motion + a bed sensor for Occupied/Asleep. Nightlight so a 2am trip to the bathroom doesn't fully wake the room's state. TV Sleep Timer if there's a TV in the room. Sleep Window is on by default, no setup needed, it just won't auto-sleep the room in broad daylight.
> Say: *"set up the master bedroom: motion, bed sensor, nightlight, and TV sleep timer."*

**🛋 A guest room** — Same as a bedroom, plus Guest Hold turned on so the room stays conservative (favors Occupied/Hold over confidently saying Vacant) for as long as guest mode is active, keeps the room from resetting itself while someone's actually staying in it.
> Say: *"set up the guest room like a bedroom, and turn on guest hold for it."*

**🚒 A bedroom for someone with a non-standard schedule** (shift worker, firefighter, anyone whose "night" isn't at night) — Set up like a bedroom, then point Autosleep Schedule at whatever actually reflects when this person sleeps: a calendar, an HA Schedule helper, or just a toggle they flip themselves before bed. This room stops caring what time the rest of the house thinks it is. If they just want to crash right now, Asleep Hold (or the manual Asleep pick, Section 5) skips the whole trigger/window question entirely — flip it, and the room is Asleep.
> Say: *"set up the bedroom like a normal bedroom, but point autosleep at my work calendar instead of the house's night hours."*

**🚪 A garage or any room with an entry door** — Motion for Occupied. Tag the actual door (not its lock) as the room's entry signal — this is what lets the room tell the difference between "someone's clearly walking through" and "motion fired but nobody actually came in." Skip TV Sleep Timer and Nightlight: a garage doesn't sleep.
> Say: *"set up the garage: motion, and the entry door as the door signal, not the lock."*

**🛁 An ensuite bathroom** — Motion + a moisture sensor if you want leak awareness. Link it to its bedroom as a child room (on by default) so a 2am bathroom trip doesn't make the bedroom look Vacant.
> Say: *"set up the master bathroom and link it to the master bedroom."*

**🛋 A living room or any room where people entertain** — Motion for Occupied, media player for Engaged. Turn on Entertaining Hold so the room stays conservative during parties instead of flickering between Occupied and Vacant as people move around. Vent Fan usually doesn't apply here.
> Say: *"set up the living room: motion, media, and turn on entertaining hold."*

**🚿 A powder room or small utility room** — Usually just motion, nothing else. These rooms don't need Asleep, Engaged, or any of the Special Moves — basic Occupied/Vacant is the whole job.
> Say: *"set up the powder room, just basic occupancy."*

---

## 9. HINTS, TIPS & TRICKS

- **Tip:** Ask "what's the state of the [room]?" any time. It's a free, instant answer. No need to guess from raw sensors.
- **Tip:** If you want to see WHY a room is in a state before you trust it, ask "why is the [room] [state] right now?"
- **Tip:** Setting a room to PAUSED is completely safe and fully reversible. When in doubt, pause first, ask questions later.
- **Tip:** A minimally configured room still works. With only basic occupancy signals tagged, it simply moves between Vacant and Occupied. There is no minimum setup required to get value.
- **Tip:** Bathrooms attached to bedrooms usually want the ensuite cascade left ON (the default). It's the behavior most people actually want, even if it surprises you the first time.
- **Secret:** REFLEX has a house-wide "dry run" mode that logs exactly what every room WOULD do without actually doing it — see Section 7. It's currently ON, house-wide, as shipped.

---

### ⭐⭐⭐ POWER USER SECRETS ⭐⭐⭐

> YOU FOUND THE HIDDEN PAGE. THE STUFF THE STRATEGY GUIDE DOESN'T PRINT. READ ON, CHAMPION.

 ► HOLD IS JUST "ONE OF THREE LINES IS HIGH." It's not its own
   signal — it's the room reporting that the wasp flag, Entertaining
   mode, or Guest mode currently reads true, whichever one it is.
   These three are hardcoded checks today, not an open template
   slot — there's no way to wire your own arbitrary condition onto
   the Hold tier yet (the generic `hold` label is a different
   mechanism — see below, it prevents the room from dropping out of
   Occupied, it doesn't put the room in Hold). Ask
   "why is the [room] hold right now" and `last_trigger` tells you
   which of the three is actually driving it — same word on the
   dashboard, three different reasons underneath, the game never
   tells you which unless you ask.

<img src="images/wasp_hold_gag.png" alt="Left: door open plus motion is confirmed. Right: door closed plus motion is Hold, not confirmed." width="500">

 ► THE DOOR, NOT THE LOCK. This is the single most common setup
   mistake and most people never find out they made it. A door's
   LOCK tells you security status. A door's CONTACT SENSOR tells
   you someone crossed the threshold. If a room feels "blind" to
   people walking in through a specific door, 9 times out of 10
   the lock got wired up instead of the door. The system will
   actually warn you about this one now if it catches the mistake
   during setup, but it's still worth knowing why.

 ► THE CLOCK ISN'T ALWAYS RUNNING. Most decaying states (Occupied,
   Engaged) run on a countdown clock that resets on activity. Hold
   from unresolved motion works differently under normal
   operation: it doesn't decay on its own, it sits there until
   something actually resolves it (the door opens again, or a
   human picks a state manually). If a room seems "stuck," check
   whether it's actually Hold before assuming something's broken.
   It's not stuck. It's waiting for real information.

 ► GRANDMA-PROOFING IS A REAL TECHNIQUE. Guest Hold exists
   specifically so a guest room doesn't "helpfully" reset itself to
   Vacant while someone's actually staying in it and just isn't
   triggering motion every five minutes. Turn it on for any room a
   guest might occupy quietly: a home office turned guest room, a
   den with a pull-out couch, whatever.

 ► THE LAUNDRY HAMPER BUG THAT NEVER HAPPENED. Bedrooms have a
   built-in defense against auto-sleeping in broad daylight: a
   pile of laundry set on the bed won't trip the room into thinking
   someone's asleep at 2pm. This runs quietly in the background on
   every bedroom, no setup required. You can turn it off per room
   if you actually want round-the-clock nap detection (a den with
   an actual napper, say).

 ► THE HOUSE'S NIGHT ISN'T EVERYONE'S NIGHT. Autosleep Schedule
   doesn't nudge the house's clock for one room — it REPLACES it
   entirely, for that room only. Point it at a calendar and this
   room stops caring what time it actually is; it only cares what
   the calendar says. Every other room keeps running on the normal
   night-to-wake window like nothing happened. This is the fix for
   "my partner works nights and the house keeps trying to put them
   to bed at 11pm."

 ► "UNTIL I SAY SO" BEATS EVERYTHING BUT AN EMERGENCY. Manually
   picking a state on the control panel (Section 5) outranks every
   automatic signal in the house except a genuine emergency. Pick
   Asleep by hand and no bed sensor, no motion, nothing short of
   you picking something else will move that room off Asleep. This
   is the correct move for "I'm napping somewhere unusual" or "keep
   this room dark for the next hour, I don't care what the sensors
   say."

 ► ASK "WHY" LIKE YOU MEAN IT. Most states your AI reports come
   with a real, traceable reason: which exact entity, and exactly
   when. This isn't a vibe check, it's a genuine audit trail (the
   one exception is Vacant, the default with nothing specific to
   point at). If something looks wrong, "why does the kitchen say
   X" almost always has a real answer, not a shrug.

 ► ROOMS TALK TO EACH OTHER, QUIETLY. The ensuite cascade (Section
   8) isn't the only place this happens: Entertaining Hold and
   Guest Hold both key off ONE shared house-wide switch, and any
   room can opt in or out independently. Flip Entertaining mode ON
   for a party and every opted-in room gets the memo instantly, no
   per-room flipping required.

 ► SCENES FADE IN BY DEFAULT, AND THAT'S ON PURPOSE. Every scene
   this system fires — automatically, not scenes you trigger by
   hand — takes 2 seconds to settle rather than slamming every
   light in the room to full brightness instantly. This also keeps
   a flickery room (a twitchy sensor, say) from hammering your
   smart home network with rapid-fire commands. Want a specific
   scene instant instead — a security scene, say, where the delay
   actually matters? Ask your AI to tag it, or do it yourself:
   label the scene `reflex_transition_0` for instant, or
   `reflex_transition_5` for a slower 5-second fade, any number you
   like. Untagged scenes just get the 2-second default.

---

## 10. TROUBLESHOOTING

| Symptom | Likely cause |
|---|---|
| Room never leaves Vacant even though someone's clearly in there | The sensor that should confirm this room isn't wired to that room yet |
| Room control keeps snapping back to Auto after I set it to Automation | Control Burnout is on for that room. That's the safety net doing its job. |
| Fan / TV sleep timer never fires | That feature isn't set up for this room yet. See Section 6, Step 2. |
| Bedroom won't go fully quiet at night because of the attached bathroom | Working as intended. See Section 8. |
| A newly added room doesn't show up yet | Settings → System → Restart, or reload templates/automations. New rooms pick themselves up automatically once that happens. |

If none of these fit, check the room's `last_trigger` attribute (Section
3) — it names the exact entity or timer currently driving the state. If
you have an AI assistant, describing what you're seeing works just as
well: "the office says occupied but nobody's in there" is a perfectly
good bug report, and it can check the same attribute for you.

---

## 11. WHAT'S NEXT

Room Manager is a living system. It grows with your house, not the other
way around. A few directions already on the table:

- Teaching Alexa-style devices to set their own wake alarms through the
  system directly (currently manual, via the device itself)
- Deeper ticket-desk integration so Room Manager issues show up
  automatically in the household's service queue
- More opt-in "special moves" as real household needs come up. The
  system was built specifically so new features cost a label, not a
  rewrite.

---

## 12. HOW TO REQUEST NEW FEATURES

Just... ask. Out loud, or in chat, to your AI. Say what room, what you
want it to do, and when. There's no form. There's no queue number.
There's no cartridge to send back.

<img src="images/mail_in_card.png" alt="Feature Request Mail-In Card: Room, What should it do, When, or just ask your AI. No stamp necessary." width="500">

*"I want the guest room's vent fan to turn off faster after someone
leaves."*

*"Can the kitchen tell the difference between someone cooking and just
walking through?"*

Every feature in this manual started as a sentence exactly like that.

---

<img src="images/game_saved.png" alt="Game Saved — Room state secure" width="300">

> Thank you for choosing Room Manager. Your house appreciates knowing
> what room it's in, even if it never says so out loud.
>
> — ZenOS-AI Home Systems &nbsp;•&nbsp; a division of nobody

---

## For the Technically Curious

This manual is deliberately non-technical. If you want the real
architecture, including the cascade logic, the event bus, the label
reference, and how to actually wire up a new room, see:

- [Component Reference: Room Manager v3 & REFLEX](../components/room_manager_v3_reflex.md)
- [Architecture Ch. 22: Room Manager v3 & REFLEX](../architecture/22_Room_Manager_v3_REFLEX.md)

---

## Addendum: For Agents Reading This Doc

Everything above is written for a human operator doing this by hand —
that's the default voice of this manual, on purpose. If you're an AI
agent reading this to help a household set something up, here's the
translation:

- **Section 6's four steps are the ground truth.** "Ask your AI" lines
  throughout this doc (worked examples' `Say:` prompts, troubleshooting
  rows, etc.) all resolve to the same underlying action: apply the
  right label(s) to the right entity, or create+label a helper, per
  Section 6 Steps 1–2. There is no separate "AI path" — it's the same
  mechanism, you're just doing the clicking.
- **Use `zen_dojotools_labels`** to create/apply labels rather than
  walking a human through Settings → Labels. Confirm the label exists
  before assuming it does — several features in this system
  (`entertaining_hold`, `guest_hold`, `autosleep_disable`,
  `asleep_window_disable`, `autosleep_schedule`, `asleep_hold`,
  `wasp_enabled`, and the per-room signal/class labels described in
  Section 6) are existence-checked, safe no-ops if missing, not
  errors — so a silently-absent label won't fail loudly, it'll just
  quietly not do anything.
- **`room_control_manager` has a real declaration now** — see the
  component reference's "Deploying a New Room" step 1. It's a template
  `select` backed by the household cabinet, not a helper you create
  through the UI like the others; the select fires an event on
  `select_option` rather than storing state itself, so don't write to
  its state directly or bypass the `room_control_request` event —
  that's what keeps validation (Hold's restricted exit list) in one
  place instead of duplicated everywhere something changes a room's
  control value.
- **After any label/helper change, a reload or restart is required**
  (Section 6 Step 4) before the room's state sensor will react. Don't
  report a setup as complete without confirming that step happened.
- **Cite `last_trigger`, not your own inference**, when explaining why
  a room is in a given state — Section 3/10 both point at this attribute
  as the authoritative answer. Guessing at a cause the attribute
  doesn't actually name is exactly the kind of confident-but-wrong
  answer this manual's plain-language framing is trying to prevent
  humans from getting stuck with; don't reintroduce it from the other
  side.
- **Run `mode=coverage_map` before guessing at a wiring gap.** It's the
  single diagnostic that combines label-discovery candidates, the
  trigger_entities gap check, dormant-feature detection, and
  `wasp_enabled` status in one call — advisory-only, never auto-applies
  anything. If you find a real gap through your own reasoning instead
  (a device that should carry a label but doesn't, say), surface it as
  a candidate for the human to confirm rather than tagging it directly
  — the "found it" and "applied it" steps must stay separated by a
  confirm gate, always.
