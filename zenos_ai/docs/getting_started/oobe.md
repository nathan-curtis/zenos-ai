# OOBE — Out-of-Box Experience

*First-boot onboarding protocol for ZenOS-AI*

---

## Overview

OOBE is the guided first-run workflow that configures your ZenOS-AI install through conversation. Your AI assistant leads you through six steps — naming the household, mapping rooms, adding people, linking integrations, activating components, and sealing the install.

OOBE runs once. When complete, it stamps `_oobe_complete` in the AI user cabinet and the flag is never set again unless explicitly reset.

---

## How OOBE Starts

Flynn manages OOBE detection automatically at boot (Gate 3.5). There are two entry paths:

**Path 1 — Conversational (recommended)**

Start a conversation with your AI assistant. Flynn's Gate 3.5 will have posted a notification directing you here. Just talk to the AI — it will invoke `zen_flynn_oobe` with `mode: run` and walk you through the steps.

**Path 2 — Agent Builder**

Call `script.flynn_build_agent` with `mode: oobe` from Developer Tools → Services. This delegates to the same OOBE protocol.

OOBE is not a script you run manually. It is a protocol the AI follows.

---

## The Six Steps

### Step 1 — Household Profile

Collects your household's name and address. The AI writes these to the household cabinet via `zen_dojotools_profile_editor`.

- Timezone is auto-detected from HA — the AI will not ask for it
- Fields: household name, address, city, state, zip, country

---

### Step 2 — Rooms

Maps your home's physical layout into the ZenOS label graph.

The AI first queries your existing HA areas (`zen_dojotools_index` or `zen_dojotools_inspect`). For each area you confirm:

1. Adjacent rooms are noted
2. Notable features are recorded
3. A label is created if one doesn't exist (`zen_dojotools_labels`)
4. A room drawer is written to the household cabinet with `{area_label, adjacent, features, notes}`

If you mention suite groupings (e.g., a master suite), a suite drawer is written as well.

> The AI confirms entity placement with you before labeling. It will never silently guess which room a device belongs to.

---

### Step 3 — People

Registers household members and extended family.

For each person in the household:
- Name and role collected (`head_of_household`, `partner`, `child`, `guest`)
- Matched to existing HA person entities via `zen_dojotools_identity`
- Written to the user cabinet via `zen_dojotools_profile_editor` (`target_type: user`)

Extended family (non-residents who matter to the household context) are written as `target_type: family`.

---

### Step 4 — Integration Mapping

Links your HA integrations into the ZenOS label graph.

The AI only offers categories where relevant integrations exist — no vacuum prompts if you don't have a vacuum.

| Category | Domain / Device Class | What Gets Tagged |
|---|---|---|
| Cameras | `camera` | Room label + `camera` label |
| Vacuums | `vacuum` | `autovac` label + room coverage order |
| Locks | `lock` | `security` label + room label |
| Presence | device_class: `presence` | Mapped to person from Step 3 |

Nothing is labeled without your confirmation.

---

### Step 5 — Component Activation

Opt-in activation of available KFC components. The AI presents only the components that make sense given your integrations.

Options offered (if applicable):
- Security alerts
- Vacuum scheduler
- Hot tub / spa manager
- Trash day reminders
- Energy monitoring

Activation preferences are written to the system cabinet.

---

### Step 6 — Close Out

The AI:
1. Rebuilds the compact index (`zen_dojotools_index`, mode: `build_compact_index`)
2. Writes the OOBE completion flag (`zen_flynn_oobe`, mode: `complete`)
3. Tells you the system is ready

After Step 6, Flynn's Gate 3.5 will no longer fire the OOBE notification. The system is live.

---

## OOBE Script Reference (`zen_flynn_oobe`)

The script itself is not a general-purpose dojotool — it is surfaced only through the AI conversation flow and the Flynn Agent Builder. Three modes are available if you need to interact with it directly:

| Mode | What It Does |
|---|---|
| `run` | Returns the full onboarding protocol for the AI to follow |
| `status` | Reports current OOBE state (complete flag, persona name, pending status) |
| `complete` | Writes the `_oobe_complete` flag to the AI user cabinet |

### Status response

```json
{
  "status": "ok",
  "mode": "status",
  "oobe_complete": false,
  "oobe_pending": true,
  "persona_name": "your AI",
  "ai_cabinet": "sensor.zenos_default_ai_user_cabinet"
}
```

---

## Hard Rules the AI Follows During OOBE

The protocol enforces these constraints on the AI's behavior:

- Never ask more than 2 questions at a time
- Write and label as you go — never batch writes to the end
- Read HA state before asking — discover what's already known, only ask what can't be inferred
- Never label an entity without confirming placement with the user
- If the user says "skip" or "later", move on — nothing blocks
- Keep responses short; no markdown in spoken answers
- If a write or label call errors, log it and continue — do not halt

---

## Resetting OOBE

To re-run OOBE on an existing install:

1. Delete the `_oobe_complete` drawer from the AI user cabinet via `zen_dojotools_filecabinet` (`action_type: delete`, `key: _oobe_complete`)
2. Restart HA — Flynn Gate 3.5 will detect the missing flag and re-enter the OOBE flow

Or call `zen_flynn_oobe` with `mode: complete` to re-stamp the flag after manual edits without re-running the full protocol.

---

## Related

- [Flynn Stepgate Sentinel](../scripts/zen_flynn_readme.md) — Gate 3.5 OOBE detection
- [Profile Editor](../scripts/zen_dojotools_profile_readme.md) — household/user profile writes
- [Labels](../scripts/zen_dojotools_labels_readme.md) — label creation during room/integration mapping
- [Install Guide](install.md) — prerequisites before first boot
