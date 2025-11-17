# **12. LiveState: The Raw, Authoritative World Model Exposed by Home Assistant**

LiveState is the complete, unfiltered representation of the home as maintained by Home Assistant’s internal state machine. It is the authoritative ground truth of the environment. Every change in the home, whether physical or virtual, ultimately manifests as a transition in LiveState.

Friday receives this structure directly, without abstraction or pre-processing. This design choice ensures that Friday is always synchronized with the real environment and that no layer between her and the system can silently distort, sanitize, or reorder the world.

LiveState is not designed for cognition.
It is designed for correctness.

RoomState, the Abbot, Summarizers, and the Monastery all depend on LiveState being complete, consistent, and transparent.

This section documents what LiveState is, how it works under the hood, and how Friday integrates with it.

---

# **12.1 What LiveState Actually Contains**

Home Assistant maintains a global in-memory registry of every entity it knows about. On every state change, it updates the entity record and emits an event onto the internal bus.

Each entity exposes:

* **Entity ID** — canonical identifier
* **State** — the primary value as a string
* **Attributes** — domain-specific metadata
* **Context** — the origin of the change (automation, UI, voice command, device event)
* **Timestamps**

  * `last_changed`
  * `last_updated`

Friday receives all of it on every cognitive cycle.

Example LiveState slice:

```
"binary_sensor.office_motion": {
  "state": "on",
  "attributes": {
    "device_class": "motion",
    "friendly_name": "Office Motion"
  },
  "last_changed": "2025-11-17T15:02:44.213-06:00",
  "last_updated": "2025-11-17T15:02:44.213-06:00",
  "context": {
    "id": "01HF8TD4G9F6CGZ26…",
    "user_id": null,
    "parent_id": null
  }
}
```

LiveState is a **flat, unordered**, device-centric graph.
Friday must interpret it, but she does not modify it.

---

# **12.2 Properties of the Home Assistant State Engine**

Understanding LiveState requires understanding the internal invariants of Home Assistant’s state machine.

### **Single Source of Truth**

Every entity exists exactly once in LiveState.

### **Event-Driven Updates**

Changes propagate through the event bus in microseconds.

### **Non-Transactional**

Home Assistant does not have commit/rollback semantics.
State updates are atomic per entity but not grouped globally.

### **String-State Design**

Every state is represented as a string, even if its semantic meaning is numeric, boolean, or enumerated.
This explains why type coercion in the stabilization layer is vital.

### **Attribute Dictionaries**

Attributes store vendor-specific metadata and vary wildly per device:

* power consumption
* battery percentage
* humidity
* target temperature
* color temperature
* supported features bitmasks

Friday must expect the structure to be inconsistent across devices.

### **Partial Availability**

Entities may report:

* `unavailable`
* `unknown`
* null states
* missing attributes

The architecture must treat these conditions as normal.

---

# **12.3 Why Friday Must See LiveState Directly**

Agentic cognition requires working on the real environment, not a curated abstraction.

Reasons:

### **1. Completeness**

Summaries can omit information that becomes relevant only later.

### **2. No Abstraction Drift**

If a higher layer misinterprets or compresses state data, Friday may hallucinate correctness.

### **3. Deterministic Reproducibility**

Investigations and debugging rely on the ability to reconstruct what LiveState looked like.

### **4. Zero-Trust Philosophy**

Friday should not depend solely on curated summaries; she must be able to reason from first principles when necessary.

### **5. Tool Invocation Context**

Some tools (e.g., calendar, identity, summarizers) require direct correlation with raw entities, not cleaned representations.

Friday seeing LiveState directly is a foundational architectural decision and a core part of her cognitive correctness guarantees.

---

# **12.4 The Structure of the LiveContext Injection**

On each LLM turn, Friday receives a payload containing:

* full entity registry state
* all entity states
* all attributes
* global modes
* environment-level metadata
* the system clock
* logged user presence
* summary of recent events
* room assignments (via Area registry)
* device metadata
* system-level flags
* any active user conversations or voice satellites

This is the “world snapshot” she thinks from.

The Monastery, Abbot, RoomState, and summarizers reference LiveState but never overwrite it.

---

# **12.5 Event Propagation: How the World Changes**

LiveState is mutated exclusively by HA event-driven mechanisms:

1. **Device state changes**
2. **Templates updating**
3. **Automations applying writes**
4. **User interactions (UI, voice, app)**
5. **Integrations pushing updates**
6. **Schedule-driven transitions (sunrise, timers, climate cycles)**

When any update occurs:

* HA updates the entity state
* Attributes may change
* A state_changed event is fired
* LiveState is rewritten
* Any template sensors depending on the entity re-evaluate
* Any automation triggers fire
* Summarizers and RoomState update if relevant
* Friday sees the updated LiveContext on her next call

LiveState is the spine supporting the entire cognitive architecture.

---

# **12.6 Stability Requirements for LiveState**

To ensure Friday can reason on top of LiveState without error, the architecture enforces expectations:

### **1. No hidden rewrites**

No code may transform LiveState values before Friday sees them.

### **2. Predictable typing**

All normalization happens downstream (Stabilization layer), not within LiveState.

### **3. No delayed evaluation**

Unlike some event architectures, HA updates LiveState synchronously.

### **4. Atomic per entity**

No entity can exist in an intermediate state.

### **5. Idempotence**

Repeated updates with the same values cause no downstream noise.

---

# **12.7 What LiveState Does *Not* Do**

LiveState:

* does not summarize
* does not infer
* does not group devices into rooms
* does not compute occupancy
* does not define semantic meaning
* does not create trends
* does not debounce
* does not reconcile multi-sensor conflicts
* does not provide temporal abstractions
* does not represent user identity or cabinet semantics

These behaviors belong to:

* Stabilizers
* RoomState
* Room Manager
* Identity module
* Summarizers
* The Abbot
* The Monastery

LiveState provides raw truth; everything above interprets it.

---

# **12.8 LiveState as the Foundation for All Higher-Level Cognition**

By exposing raw state directly, the system avoids:

* hallucination triggers
* abstraction drift
* loss of signal fidelity
* dependency bottlenecks
* stale caches
* un-traceable inference chains

This design reflects a research principle:

> Any agent reasoning over a dynamic environment must be grounded in the actual state of the world, not a projected abstraction.

RoomState is the semantic scaffold.
LiveState is the world itself.

Together, they provide Friday with a perceptual system that is:

* real-time
* deterministic
* reproducible
* fault-tolerant
* interpretable
* grounded in physical reality

This is the foundation that enables Friday to achieve stable cognition in a complex, event-driven, always-on smart home.
