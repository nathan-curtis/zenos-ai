# ZenOS-AI: First Run Guide

> **Version:** 4.0.0 RC2 | **Last Updated:** March 2026

---

## Before You Start

You'll need:

- Home Assistant running with the ZenOS-AI packages installed (`packages/zenos_ai/` and `custom_templates/zenos_ai/`)
- A conversation agent configured in HA and pointed at a compatible AI model (tool-calling support required; models smaller than ~8B parameters or with short context windows are not recommended)
- The conversation agent prompt template loaded from `custom_templates/zenos_ai/conversation_agent_prompt_template.yaml`

If you haven't done those steps yet, check the install guide before continuing.

---

## What Happens on First Boot

The moment HA starts with ZenOS-AI installed, an automation called **Flynn Stepgate Sentinel** runs. You don't trigger it — it fires automatically. Flynn works through a sequence of setup gates:

1. **Labels** — Creates any missing Zen system labels in HA
2. **Assign** — Attaches those labels to the right cabinet entities
3. **Cabinets** — Initializes your four core storage cabinets (AI user, household, family, user)
4. **Bootstrap** — Writes default starting values so the system has something to work with

If everything passes you'll see a **"ZenOS-AI: System Ready"** notification. If the AI persona isn't configured yet you'll see this instead:

> *ZenOS-AI: Welcome — Let's name your AI*
> *Flynn here. Your system is ready but your AI doesn't have a name yet...*

That notification means setup completed successfully. The OOBE (onboarding) step is next.

---

## Naming Your AI — The OOBE Conversation

OOBE stands for Out-of-Box Experience. It's a one-time conversation where your AI learns your home.

**To start:** Open a conversation with your AI assistant and say something like:

> "Set up my profile"
> "Let's do first-time setup"
> "Flynn, walk me through onboarding"

Your AI will call the onboarding protocol and begin asking questions. It already knows your HA areas and entities — it'll confirm what it sees rather than asking you to describe everything from scratch.

### What it will cover

**1. Your household**
Name and address. Timezone is detected automatically.

> *"What would you like to call your home?"*

**2. Rooms**
Your AI reads HA areas and confirms the list with you. For each room it asks:
- What rooms connect to it directly?
- Anything notable? (Smart speaker, fireplace, special equipment)

It writes a room profile as it goes — not at the end.

**3. People**
Who lives in the home, with their name and role. It checks HA for matching person entities. It'll also ask about family who matter but don't live there (parents, siblings) and keep those separately.

**4. Devices**
For each category it finds in your system it'll confirm placement before tagging anything:
- Cameras → confirmed to a room
- Vacuums → confirmed cleaning coverage
- Locks → confirmed to a door
- Presence sensors → mapped to a person

It will never label a device without asking first.

**5. Components (optional)**
A quick opt-in for features relevant to your setup — security alerts, vacuum scheduling, spa manager, trash reminders, energy monitoring. If the hardware isn't there the option won't be offered.

### Rules your AI follows during OOBE

- Never asks more than two questions at once
- Writes as it goes — nothing is batched
- "Skip" or "later" always works — nothing is blocking
- If a write fails, it logs it and moves on

---

## When OOBE Finishes

Your AI writes an `_oobe_complete` flag to its cabinet. On the next HA restart, Flynn sees it and clears the welcome notification. You won't see it again.

**The persona selector** (`select.zenos_active_persona`) updates automatically once your AI has a name. If you previously only saw "friday" in the selector, completing OOBE will add your newly named AI to the list.

---

## Editing Your Profile Later

You don't need to re-run OOBE to change anything. Just ask your AI directly:

> "Change your familiar's name to Pip"
> "Update our household name to The Curtis House"
> "Add Kim as my partner"
> "What's your current profile?"

Your AI uses `zen_dojotools_profile_editor` under the hood. It reads before it writes, patches only what you specify, and never overwrites existing values unless you explicitly ask it to.

Fields you can update any time:

| Cabinet | Examples |
|---|---|
| AI persona | Name, pronouns, voice tone, selfie, familiar, room, motif |
| Household | Name, address, city, state, phone |
| User | Display name, preferred name, role, birthday, email, preferences |
| Family | Same as user |

---

## If Something Goes Wrong

**The welcome notification keeps appearing after setup**
Your AI's name may still be the default ("your AI"). Ask your AI what its name is — if it says it doesn't have one yet, run OOBE again.

**Only one persona in the selector**
The selector builds from AI user cabinets that have a named persona. Complete OOBE or ask your AI to set its name directly.

**A room or device wasn't set up correctly**
Just ask your AI to fix it. "The motion sensor in the hallway is actually in the bedroom." It can relabel entities and update room drawers without re-running the full flow.

**OOBE can be re-run**
Ask your AI: "Run OOBE again" or "Re-do first-time setup." It will walk through the protocol again. Existing values are skipped unless you ask it to overwrite.

---

## What's Next

Once your home is set up, your AI has full context to be useful:

- It knows your rooms and who's in them
- It knows your devices and what they're for
- It knows who lives in your home and how they're connected
- The label index is rebuilt and ready for queries

From here, explore what your AI can do. Start a conversation. Ask it about your home. It's ready.
