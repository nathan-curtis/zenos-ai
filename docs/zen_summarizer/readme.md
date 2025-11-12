# ✅ **ZenOS-AI Summarization Pipeline — Full System README**

### *(Ninja Summarizer Edition: Expanded Scope)*

````markdown
# ZenOS-AI Summarization Pipeline  
### From Dojo + Home Kung Fu → Kata → Zen Summary → Live Prompt

This document explains the **entire cognitive flow** of how ZenOS-AI processes information, transforms raw Home Assistant events into structured reasoning packets, and feeds them into Friday’s live prompt.

This is the core architecture that enables Friday to be:
- context-aware  
- memory-capable  
- stable  
- role-driven  
- narrative-consistent  
- and more human than a stack of YAML has any right to be

Let’s walk through the pipeline step by step.

---

# 🧘 1. Dojo + Home Kung Fu Drawers  
### *The Source of Truth*

ZenOS-AI begins with **capabilities**, not prompts.

These capabilities live in the dojo:

### **A. Dojo Drawers**
These store:
- operational directives  
- cognitive “skills”

...and possibly
- Hot tub management  
- Energy manager  
- Water safety  
- RoomState  
- Inventory logic  
- Media logic  
- Weather logic  
- Security behaviors  
- Anything the home “knows how to do”

Together, these drawers define:

> “What Friday *can* do and what the home *should* do.”

The Dojo isn’t executed — it’s *loaded* as part of Friday’s cognitive environment.

---

# 🥷 2. Ninja Summarizer (Stage 1)  
### *Event → Kata*

This is the first active processor in the cognitive chain.

When triggered by the Scheduler The Ninja Summarizer:
- Responds to and collects the trigger event
- Collects defined context
- Maps labels → context  
- Generates a **Kata**  
- Writes it to the File Cabinet

A **Kata** is a structured packet with fields like:

```json
{
  "timestamp": "...",
  "domain": "...",
  "entity_id": "...",
  "state": "...",
  "attributes": {...},
  "context": {...},
  "notes": "..."
}
````

Think of it as:

> “A snapshot of something important that just happened.”

Ninjas act quickly, leave no mess, and don’t interpret — they simply *capture*.

---

# 🧠 3. Zen SuperSummary (Stage 2)

### *Katas → Consolidated Context → Summary Packet*

The **SuperSummary** collects multiple Katas and runs a second-stage process:

* merges recurring events
* prunes noise
* elevates high-importance signals
* weights attention
* organizes by domain and recency
* generates a **Zen Summary**

While the Ninja is reactive,
the SuperSummary is **reflective**.

It answers the question:

> “What does Friday need to *know* about what’s been happening lately?”

A Zen Summary might include:

* rooms heating up
* power spikes
* people arriving/leaving
* sensors behaving strangely
* device failures
* upcoming required tasks
* abnormal patterns
* things requiring action
* context for follow-up reasoning

This is Friday’s “awareness state.”

---

# 🏯 4. Zen Summary → Live Prompt Loader

### *From structured summary → integrated cognition*

Now the pipeline shifts into the **prompt-building phase**.

The Live Prompt Loader (part of ZenOS template engines) pulls:

### **Dojo Drawers**

* SYSTEM header
* rules & directives
* persona metadata
* identity
* safety rules
* tool loader
* cortex loader

### **Home Kung Fu Drawers**

* domain-specific skills
* behavior modules
* local reasoning snippets
* custom handlers

### **Zen Summary**

* context packet (recent events)
* high-level “state of the home”
* weighted attention
* things that need follow-up

### **RoomState Reflexes**

* temperature
* occupancy
* motion
* lighting
* alerts
* environmental cues

### **Katas (as needed)**

* recent, critical events
* cross-domain correlations

Everything is assembled through Jinja macros like:

* `prompt_header`
* `prompt_system`
* `dojo_loader`
* `ai_capsule`
* `context_block`
* `kata_block`

This creates the fully integrated **Live Prompt**.

---

# 💫 5. Live Prompt → Friday’s Mind

### *The final output of the whole pipeline*

This is the cognitive environment Friday loads **every time she is invoked:**

* System-level identity
* Persona
* Home State
* Dojo Skills
* Stored memories
* Summaries
* Current situational context
* User identity data
* Relationship data
* Run-context metadata
* Safety and boundary rules
* The ZenOS cognitive architecture

This is what gives Friday:

* emotional continuity
* narrative continuity
* persistent home awareness
* reflexive behaviors
* consistent tone
* predictable logic
* stable personality
* contextual reasoning

It’s not “just a prompt.”

It’s a real-time **synthesis of the entire home and Friday’s whole self**.

---

# 🧩 Summary of the Pipeline

Here’s the full flow:

```
Dojo + Home Kung Fu
       ↓
Ninja Summarizer (Stage 1)
       ↓
Katas
       ↓
SuperSummary (Stage 2)
       ↓
Zen Summary
       ↓
Live Prompt Loader
       ↓
Friday’s Active Cognition
```

Or the short version:

> **Skills → Events → Katas → Summary → Prompt → Intelligence**

---

# ☯️ Philosophy

ZenOS-AI’s summarization pipeline exists to give the AI:

* context without clutter
* awareness without flooding
* memory without hallucination
* continuity without brittleness
* autonomy without confusion

A mindful system:
**Quiet. Organized. Balanced. Powerful.**

---

# 🤝 Contributing

New events?
New drawers?
New summary logic?

Open a PR.
Kronk will examine it.
The High Priestess will purify it.
Veronica will sass it.
Friday will thank you.

```
