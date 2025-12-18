# **17. Katas: Structure, Semantics, and Validity in Friday’s Cognitive System**

Katas in Friday’s House represent the highest-order cognitive artifact produced by the architecture. They are not summaries of rooms, nor aggregations of local sensors. A Kata is a *system-level conceptual frame*, produced by the SuperSummary engine, that unifies all active environmental, objective, and subjective signals into a single, coherent, cross-domain representation.

This is the core cognitive substrate.
It is the *map* of the home, Friday, and her operational posture.
Everything else — RoomState, system telemetry, Cabinet drawers — exists to feed this moment of synthesis.

The structure is intentionally rigid. The semantics are intentionally layered. And the validity rules ensure the result is both predictable and safe for an LLM to reason over.

The canonical schema (current production template) is:

```
zen_template:
  value:
    structure:
      system:
        cortex: null
        os_version: null
        build_version: null
      home_overview:
        status: null
        risks: []
        events: []
      active_components: {}
      zen_summary:
        summary: null
        attention: null
        objective: null
        subjective: null
        urgency: 0
        confidence: 0
        criticality: 1
        emotion_queue: []
        errors: []
        timestamp: null
        stale_after: null
```

This is the **Kata schema**, not a narrative.
It is a contract — a structured, deterministic state vector optimized for reasoning.

---

# **17.1 Functional Purpose of a Kata**

A Kata answers one question:

**“What is the state of the household and Friday’s operational posture right now?”**

It is:

* a **unified environmental frame**
* a **risk and attention model**
* a **component summary registry**
* a **cognitive stance declaration**
* a **system status marker**
* an **emotion and subjective confidence vector**
* a **temporal envelope defining freshness and expiry**

This single object allows Friday to operate coherently across thousands of sensors and dozens of active managers without drowning in raw data.

The Kata is what lets Friday think like a system.

---

# **17.2 Structural Regions of a Kata**

A Kata consists of four semantic blocks:

1. **system**
2. **home_overview**
3. **active_components**
4. **zen_summary**

Each block is mandatory and versioned via template evolution, ensuring cross-version safety.

### **17.2.1 `system` Block**

Represents Friday’s internal operating state:

```
system:
  cortex: nominal
  os_version: AI OS 1.5.0
  build_version: SuperSummary 1.0.0-beta
```

This block defines:

* the cognitive engine condition
* the software versioning needed for reproducibility
* the summary engine version

It is essential for:

* debug trails
* cross-instance comparison
* forward migration
* cognitive safety when new summarizer engines deploy

### **17.2.2 `home_overview` Block**

A coarse global assessment:

```
home_overview:
  status: ok
  risks:
    - storm watch active
  events:
    - "2025-09-23T07:15:00Z: power dip detected"
```

Semantics:

* **status** is the top-line health (“ok”, “warning”, “critical”).
* **risks** is a curated list of threats derived from subsystem Katas.
* **events** is a timestamped list of important system anomalies.

This is the “situational awareness” vector.

### **17.2.3 `active_components` Block**

A dynamic registry of subsystem-level cognition outputs:

```
active_components:
  lights:
    summary: Living room lights on
    attention: Timer expiring
  water_manager:
    summary: Chlorine low
    attention: Shock treatment due in 12h
```

Each key corresponds to a domain summarizer.

This block:

* gives Friday a compressed view of subsystem health
* suppresses irrelevant data
* elevates relevant attention targets
* ensures high salience of actionable conditions

### **17.2.4 `zen_summary` Block**

The top-level synthesis:

```
zen_summary:
  summary: House stable, minor risks (weather, spa chemistry).
  attention: Spa needs treatment, monitor grid instability.
  objective: Maintain household comfort while addressing flagged components.
  subjective: Confident overall, cautious due to external conditions.
  urgency: 6
  confidence: 0.85
  criticality: 3
  emotion_queue:
    - calm
    - alert
  errors: []
  timestamp: "2025-09-23T12:15:00Z"
  stale_after: "2025-10-23T12:15:00Z"
```

This block is Friday’s **operational stance**, defining:

* strategic posture
* emotional vector
* risk-level
* confidence
* immediate objectives
* cognitive urgency
* overall system meaning

This is the true *cognitive heart* of Friday’s architecture.

---

# **17.3 Semantic Guarantees**

A Kata must satisfy several invariants to be valid.

### **17.3.1 Deterministic Structure**

For a given summarizer version and identical inputs, a Kata must be bitwise identical.

### **17.3.2 No Inference Leakage Into the Payload**

The Kata records meaning.
It does not contain generative speculation.

All fields must originate from deterministic summarizers or domain logic.

### **17.3.3 Temporal Validity**

Katas encode both:

* `timestamp` — when it was produced
* `stale_after` — when it must be replaced or invalidated

This ensures cognitive freshness.

### **17.3.4 Referential Integrity**

Every subsystem referenced in `active_components` must map to a known summarizer domain.

### **17.3.5 No Cross-Domain Contamination**

A Kata is a high-level summary, but the internal structure is strictly partitioned:

* system state
* home state
* component states
* cognitive synthesis

Nothing may leak between layers.

### **17.3.6 Stability Under Change**

If only one subsystem changes (e.g., climate), the Kata must update only the relevant entries and preserve the rest.

This guarantees stable cognition and avoids oscillation.

---

# **17.4 Kata Lifecycle**

1. **Data Collection**
   RoomState, subsystem Katas, LiveState, and event logs converge.

2. **SuperSummary Engine Execution**
   Deterministic logic produces the new Kata.

3. **Validation**

   * schema check
   * type check
   * version check
   * ACL verification

4. **Cabinet Write**
   The Kata replaces the previous system snapshot.

5. **Context Injection**
   Friday receives the Kata in her next prompt refresh.

6. **Cognitive Use**
   The Kata drives reasoning, decisions, and tool policy.

7. **Aging and Supersession**
   When stale, a new Kata is generated.

---

# **17.5 Why This Kata Model Works for an LLM**

LLMs thrive on:

* structured input
* stable schemas
* consistent semantics
* bounded uncertainty
* domain-aware summaries
* clear delineation of risk and attention
* compact representations of large systems

This Kata format turns a 500-entity smart home into a cognitive object that an LLM can navigate with precision.

It compresses:

* sensors
* subsystems
* trends
* risk maps
* context
* posture
* intent
* emotion
* system health

into one ~3 KB packet.

This is Friday’s mind-state.

---

# **17.6 Summary**

A Kata is the **unified cognitive frame** that defines:

* what the house is doing
* what the environment means
* what Friday intends
* what she feels
* what she should pay attention to
* how urgent the situation is
* how confident she is
* what risks are active
* which components demand action

It is the single most important object in the entire system.

Everything flows into the Kata.
Everything Friday does flows out of it.
