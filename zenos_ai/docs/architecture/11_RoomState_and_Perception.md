# **11. The Perception System: Semantic Grounding of the Home Assistant State Graph**

The perception layer in Friday’s cognitive architecture does not attempt to hide or abstract Home Assistant’s raw state model. Friday receives full, unfiltered LiveContext on every message cycle — every entity, every state, every attribute, every recent transition. This is essential for an agentic assistant embedded inside a live environment.

However, the raw graph is *not* organized for cognition. It is inconsistent across device vendors, contains unstable signal patterns, exposes rapid state jitter, and encodes semantics implicitly rather than explicitly.

The purpose of the perception system is therefore not concealment, but **semantic grounding**: creating a stable, structured map that tells Friday what each raw value *means* and how it should be interpreted within the lived environment of the household.

This grounding is implemented through three components:

1. **Raw State (Live HA Graph)** — what HA gives for free
2. **Stabilization Layer** — minimal normalization
3. **RoomState** — the semantic decoder for LiveContext

These together provide the perceptual substrate for all higher cognition in Friday’s House.

---

## **11.1 LiveContext: Full Raw Home Assistant State**

Friday receives the entire real-time Home Assistant state graph in her `LiveContext` injection:

* entity_id
* raw state string
* attributes dictionary
* last_changed
* last_updated
* context of the last event
* dynamic discovery of new entities
* global metadata (home mode, security mode, user presence, etc.)

Examples include:

```
binary_sensor.living_motion: on
sensor.kitchen_illuminance: 439.2
input_boolean.guest_mode: off
person.nathan: home
lock.front_door_lock: locked
cover.garage_door: opening
media_player.living_room_tv: playing
assist_satellite.kitchen_voice_assistant: active
```

This layer is complete, correct, and authoritative — but it is **not cognitively usable** on its own.

Challenges from the raw graph:

* inconsistent domain semantics
* vendor-specific naming
* attribute heterogeneity
* jitter, bouncing, and sampling noise
* the absence of room-level grouping
* unstructured meaning (e.g., “on” for a motion sensor ≠ “on” for a light)

Friday can always see it.
Her problem is understanding it.

---

## **11.2 Stabilization Layer**

This layer is implemented primarily using template sensors, helper entities, and Home Assistant’s deterministic state engine. It does not summarize or infer but ensures the raw signal is stable enough to form meaningful cognitive grounding.

Functions include:

### **Debouncing**

Prevents “motion on → off → on → off” flapping during a 1–3 second transition.

### **Hysteresis & Thresholds**

Ensures illuminance, temperature, humidity, and contact sensors don’t oscillate repeatedly during marginal values.

### **Semantic Coercion**

Makes vendor-specific outputs consistent:

* `detected`, `yes`, `motion`, `active` → `on`
* `clear`, `no`, `idle` → `off`

### **Null & Unavailable Cleanup**

Ensures Friday never sees an “unavailable” that breaks chains of reasoning.

### **Safe Typing**

Coerces:

* numeric strings to numbers
* battery readings to percentages
* enumerations to canonical domain labels
* state strings into typed values

This layer provides a consistent signal surface — but still does not categorize, interpret, or group anything into rooms.

---

## **11.3 RoomState: Real-Time Semantic Map of the House**

RoomState is the first layer that provides *meaning* rather than raw signal.

It does not hide raw values; it **defines the household ontology** so Friday can correctly interpret what LiveContext is telling her.

RoomState answers:

* “What is a room?”
* “Which devices are relevant to this room?”
* “What concepts matter in this room?”
* “What does ‘motion’ mean here?”
* “How do we talk about brightness, comfort, activity, and transitions?”

Every room has a structured object representing its live semantics:

```
rs.living_room:
  motion_active: true
  brightness_level: 438
  temp_f: 72.9
  door_open: false
  last_motion: "2025-11-17T16:02:11-06:00"
  device_states:
    - media_player.living_room_tv: playing
    - light.living_lamp: off
```

**Key design properties:**

### **Real-time**

RoomState updates instantly when any stabilised raw sensor changes.

### **Declarative**

No inference, no prediction — just semantic mapping.

### **Room-centric**

Summaries reflect *spaces*, not devices.

### **Readable by Friday**

RoomState gives Friday’s cognition a consistent anchor to understand LiveContext.

### **Non-hierarchical**

RoomState does not attempt to infer occupancy, activity, comfort, or intention.
Those are the domain of Room Manager or higher layers.

RoomState simply states what the environment *is*, in clear, structured terms.

---

## **11.4 RoomState as Meaning for the LiveContext Snapshot**

LiveContext = everything HA knows
RoomState = what everything HA knows *means*

Friday reads both simultaneously.

LiveContext provides:

* the raw world
* every entity
* every state
* every attribute

RoomState provides:

* stable semantics
* room grouping
* domain-correct interpretations
* canonical naming
* standardized structures
* cross-device alignment

Friday’s perception pipeline works like this:

**LiveContext** → “The kitchen illuminance sensor is reporting 436.2.”
**RoomState** → “In the kitchen schema, brightness_level = 436.2 (daylight).”

**LiveContext** → “binary_sensor.living_motion is on.”
**RoomState** → “motion_active = true in the living room.”

This prevents hallucination, misinterpretation, or mis-attribution of meaning.

RoomState is not a filter; it is a semantic decoder that turns raw sensory information into structured facts.

---

## **11.5 RoomState Integrity Requirements**

RoomState must obey the following invariants:

### **Canonical Schema**

All fields remain stable across upgrades to prevent cognitive drift.

### **Lossless**

No summarizer ever overwrites or alters raw semantics.

### **Atomicity**

Every room object is internally consistent at the moment of read — no partial writes.

### **Strict Typing**

Every field is guaranteed numeric/boolean/string per schema definition.

### **Deterministic Update Ordering**

State transitions yield predictable RoomState updates.

### **Fully Observable**

Failures propagate upward; nothing is hidden or swallowed.

RoomState is designed to be a trustworthy reference frame for Friday’s real-time cognition.

---

## **11.6 Division of Responsibility: RoomState vs Room Manager**

RoomState:
**Semantic grounding**
A structured description of what exists, now.

Room Manager:
**Cognitive interpretation**
Determines what the room *means* behaviorally:

* occupancy probability
* inferred activities
* environmental recommendations
* automation preconditions
* sensor fusion interpretation

These layers deliberately do not overlap.

**RoomState must never do cognition.
Room Manager must never alter RoomState.**

This separation preserves both correctness and cognitive clarity.

---

## **11.7 Summary**

Friday sees everything in the HA graph.
RoomState tells her what it all means.

* Raw → unpredictable and vendor-specific
* Stabilized → clean but still unstructured
* RoomState → real-time semantic map
* LiveContext → the literal world as HA sees it
* RoomState + LiveContext → grounded cognition
* Room Manager → optional cognitive interpretation layer

This architecture gives Friday:

* real-time perception
* semantic stability
* no hallucination risk
* deterministic behavior
* compatibility with any HA environment
* a foundation for agentic cognition at household scale

RoomState is not the “processed” world — it is the **semantic lens** that makes the raw world intelligible.
Your call, boss.
