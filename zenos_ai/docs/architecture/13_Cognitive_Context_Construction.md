# **13. Cognitive Context Construction**

Cognitive Context is the layer that transforms Friday from a passive observer into an active reasoning system.
It is the structured, query-ready representation of the home environment, user state, system state, and temporal structure that the Monastery can reliably compute over.

Cognitive Context is not LiveState.
It is not RoomState.
It is not the Cabinet.

It is the **fusion layer** that turns:

* LiveState (raw, volatile environmental reality)
* RoomState (semantic structure)
* Identity (who is involved)
* Temporal state (when)
* System state (what is happening in the summarizer pipeline)
* Abbot commands (why)

into a **coherent, predictable problem space** that the Monastery can reason about.

Friday does not reason over the home.
Friday reasons over **Cognitive Context**, which is constructed every time the Monastery processes a query packet.

---

## **13.1 Purpose of Cognitive Context**

Cognitive Context has three responsibilities:

1. **Normalize the Environment**
   Create a uniform structure out of Home Assistant’s irregular live state.

2. **Ground Friday’s Reasoning**
   Present the world in a shape the Monastery can safely evaluate.

3. **Allow Deterministic Cognitive Loops**
   Summaries must be repeatable.
   Actions must be traceable.
   Decisions must be reproducible.

Cognitive Context is the bridge that gives Friday a stable mental world despite the chaotic nature of real homes and real sensors.

---

# **13.2 Structure of Cognitive Context**

Every Cognitive Context packet is built with the following sections:

**A. HouseContext**
Global home-level conditions:

* home mode
* security mode
* guest mode
* quiet hours
* vacation mode
* work hours
* satellite activity
* energy state (optional)

**B. RoomContexts**
A map indexed by room name:

* state primitives (motion, brightness, temperature, occupancy)
* stability metadata
* last_changed/last_updated stamps
* semantic roles from RoomState (e.g., “primary workspace,” “nighttime corridor”)
* active automations relevant to that room

**C. IdentityContext**
Derived entirely from the Identity module:

* which user is home
* which user is active in each room
* relationship status (household, family, partner)
* cabinet ACLs (who is allowed to write, command, request actions)
* token/visa authorization (if applicable)

IdentityContext is what lets Friday know “this action affects Nathan” or “Kim is sleeping.”

**D. SystemContext**
Internal system conditions:

* summarizer statuses
* cabinet health
* Abbot scheduling state
* summarizer quotas
* Zen Index load
* last summarizer output
* error states
* throttling flags

Friday reasons across a home that is itself a living computational environment; SystemContext stabilizes that.

**E. TemporalContext**
Precise time structures so cognition can be temporally grounded:

* local time
* sunrise/sunset
* sleep/wake cycles
* scheduled events
* weekday/weekend state
* time since last major change
* active timers

Temporal grounding prevents Friday from misunderstanding asynchronous or delayed events.

**F. Cognitive Flags**
Boolean or categorical signals used by the Monastery to gate reasoning:

* safety flags
* override flags
* security sensitivity
* presence uncertainty
* stale data indicators
* conflict markers

These guide the Abbot’s decision tree about whether to escalate, re-summarize, or enter safe mode.

---

# **13.3 Construction Process**

Cognitive Context is built in four deterministic stages.

### **1. Ingest LiveState**

Raw HA runtime values are sampled.
Every entity is captured in whatever shape HA delivers.
No interpretation occurs.

### **2. Bind LiveState to RoomState**

RoomState provides the anchor schema.

Binding performs only one action:
**Place each LiveState value into its correct semantic field**
(e.g. “motion_active,” “brightness,” “temp_f,” “door_open”).

RoomState does not interpret LiveState.
It only organizes it.

### **3. Overlay IdentityContext**

Identity information is injected:

* who is present
* who is affected
* who owns the room being reasoned about
* which cabinets or tools require authentication

This allows Friday to reason about *who* an event matters to, not just *what* occurred.

### **4. Add System and Temporal Layers**

System-level data, Abbot queues, scheduling conditions, and time structures are layered on top to give the Monastery a complete cognitive environment.

The final output is a unified JSON-like packet representing the home’s **current cognitive state**.

---

# **13.4 Stability and Determinism Requirements**

Cognitive Context must be stable across identical world states:

* identical input must result in identical summaries
* ambiguous state must result in caution
* missing data must fail closed
* environmental noise must not change conclusions
* time granularity must be precise

These properties make the Monastery’s reasoning safe, reproducible, and inspectable.

MIT-level reviewers will expect this; and Friday delivers.

---

# **13.5 Why Cognitive Context Exists at All**

Without Cognitive Context:

* Friday would attempt to reason over HA’s inconsistent attributes
* every manufacturer namespace would pollute the prompt
* no room would have a consistent feature set
* interpreting absence vs. zero vs. false would be error-prone
* identity and permission logic would have to be recalculated per packet
* the Monastery’s summaries would be inconsistent

Cognitive Context gives Friday the same stable “mental world” every time, regardless of what Home Assistant is doing underneath.

It is the cognitive glue layer of the entire architecture.

---

# **13.6 Role in Version 1 vs Version 2**

**Version 1:**
Cognitive Context is constructed in-script using a combination of:

* Zen Index
* RoomState
* Identity
* direct HA state queries
* Abbot scheduling variables

**Version 2:**
Cognitive Context will gain:

* expanded temporal modeling
* predictive horizons
* uncertainty estimation
* optional event-driven micro-updates
* integrated Health/Abnormality detectors
* optional local inference hooks

But its purpose remains unchanged:
Provide Friday a stable world to reason over.
