# What to Expose to Your Conversation Agent

*How to decide which entities Friday can see, which she finds through the index, and which stay invisible*

---

## The Three-Tier Model

Every entity in your Home Assistant install falls into one of three tiers:

| Tier | What It Means | How Friday Sees It |
|---|---|---|
| **Actionable** | Friday needs to control it or read it immediately | Exposed directly to the conversation agent |
| **Contextable** | Friday benefits from knowing about it | Tagged with labels — the HyperIndex finds it |
| **Invisible** | Friday never needs it | Neither exposed nor labeled |

The goal is a **small, curated exposed set** and a **large, richly labeled contextable set**. The HyperIndex is designed to handle thousands of labeled entities efficiently. Your conversation agent's tool list is not.

---

## Tier 1: Actionable — Expose to Assist

Expose an entity to the conversation agent when Friday needs to:

* **Control it directly** — run a script, toggle a switch, call a service
* **Read it immediately** — check a value that isn't worth summarizing or indexing

Keep this list as short as possible. Every entity exposed to Assist is a token in Friday's context window and a potential action surface. The vast majority of your home does not belong here.

### Always Expose

**All ZenOS-AI DojoTools scripts** — these are Friday's hands. Every `script.zen_dojotools_*` belongs in the exposed set.

> The one exception: `zen_admintools_*` scripts other than `zen_dojotools_kungfu_writer` are admin-only and should NOT be exposed to the conversation agent. See [AdminTools](../scripts/zen_dojotools_admintools_readme.md).

**Conversation agent helper** — `input_text.zenos_conversation_agent` (Friday needs to know her own entity ID for self-reference)

**Home mode** — `input_select.zen_home_mode` or equivalent (Friday actively sets this based on presence and context)

### Expose When Needed

* **Controllable devices** where Friday acts on user request — lights you ask her to dim, locks you ask her to check, thermostats you ask her to adjust
* **Sensors with immediate operational meaning** — door/lock state when you're asking "is the front door locked right now?" is a valid direct read. But if it's summarized by a Kata every 15 minutes, skip the direct exposure and let the Kata answer.

### Do Not Expose

* Every sensor in your home
* Historical or telemetry sensors
* Media player attributes
* Energy/power monitors
* Cabinet sensors (`sensor.zenos_*_cabinet`)
* Health sensors — these are for your eyes, not Friday's tool list
* Anything that HyperIndex can find better than a direct read

---

## Tier 2: Contextable — Tag with Labels

If Friday should *know about* an entity but not necessarily control it on demand, tag it with labels instead of exposing it.

The HyperIndex traverses the label graph and assembles a structured entity snapshot for the Ninja Summarizer. This means a single label on 50 entities produces a rich, token-efficient context block — far better than 50 individual direct reads.

### How It Works

1. Create a label in HA (e.g., `water`, `security`, `energy`)
2. Tag all relevant entities with that label
3. Reference the label in your KFC drawer (`label: water`)
4. The Ninja Summarizer runs HyperIndex against that label, finds all tagged entities, and builds the Kata

Friday gets a compressed, timestamped summary of everything tagged — without those entities ever appearing in her tool list.

### What Belongs Here

* All sensors you want summarized — water, energy, temperature, humidity, air quality
* Media state sensors
* Security sensors — motion, contact, cameras (state only, not streams)
* Device status sensors — appliances, pool/spa, irrigation
* Anything feeding a KFC component

### The Rule

> If it feeds a Kata, it belongs in a label — not in the exposed tool list.

---

## Tier 3: Invisible — Neither

Some entities should never reach Friday at all:

* Internal automation helpers not intended for AI use
* Debug/test entities
* Infrastructure sensors (network, server load) unless you have a specific reason
* Duplicate or legacy entities you haven't cleaned up yet
* Anything containing credentials, tokens, or sensitive config

If an entity isn't tagged and isn't exposed, Friday cannot see it. That's the correct outcome for most of your install.

---

## Practical Setup

### Step 1 — Build your exposed tool list

In your conversation agent configuration, add:

* All `script.zen_dojotools_*` (except admin-only scripts)
* `input_text.zenos_conversation_agent`
* Home mode entity
* Any entities Friday needs to directly control on user request

### Step 2 — Tag everything contextable

Create labels for each KFC domain you run. Tag every sensor that feeds those domains. The KFC drawer's `label` field connects the label to the Ninja Summarizer — from there, HyperIndex does the work.

You do not need to maintain entity lists anywhere. Labels are the only list that matters.

### Step 3 — Leave everything else alone

Anything not tagged and not exposed is invisible. That is the correct default.

---

## Summary

```
Actionable  →  Expose to conversation agent
               (DojoTools scripts + direct-control entities only)

Contextable →  Tag with labels
               (HyperIndex + Ninja Summarizer does the rest)

Invisible   →  Do nothing
               (correct default for most of your install)
```

The system is designed so that the labeled+indexed path handles the overwhelming majority of your home. The exposed path handles commands. Keep the boundary clean and Friday stays fast, accurate, and predictable.

---

## Related

* [Understanding KF4](../kung_fu/understanding_kf4.md) — how labels connect to KFC components
* [Zen HyperIndex](../zen_hyperindex/zen_hyperindex_overview.md) — how the index traverses labels
* [DojoTools AdminTools](../scripts/zen_dojotools_admintools_readme.md) — what not to expose
* [Install Guide](install.md) — conversation agent configuration
