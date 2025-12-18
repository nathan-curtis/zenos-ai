# **16. The Summarizer Engine & Kata Pipeline**

The Summarizer Engine forms the central compression layer of Friday’s cognitive architecture. It is the sole mechanism responsible for collapsing raw environmental state, temporal changes, and subsystem signals into structured, discrete, machine-actionable packets called **Katas**. These Katas provide the stable, low-entropy cognitive substrate that enables deterministic reasoning, safe planning, and cross-module coherence.

Where most LLM-based smart-home systems rely on live prompts full of volatile state, Friday’s system replaces that paradigm entirely with a **summarized substrate**. Friday reasons over Katas, not over raw sensor noise, thereby achieving stability, latency resilience, and locality of meaning.

This section details the complete Summarizer Engine pipeline as implemented in Version 1: its triggers, inputs, transforms, validation, and output guarantees.

---

## **16.1 System Role of Summarizers**

Summarizers serve three systemic purposes:

### **1. Compression**

They collapse a high-dimensional state space (thousands of sensors, binary toggles, modes, device states) into a minimal, domain-specific representation.

### **2. Stabilization**

They filter out noise, transient spikes, jitter, and partial transitions — only recording meaningful changes.

### **3. Cognitive Framing**

Each Kata embeds domain semantics (e.g., “is kitchen active,” “is security relaxed,” “is water stable,” “is house opening expected”), enabling Friday’s reasoning engine to operate on consistent, unambiguous truth primitives.

In short, they compress, stabilize, and canonicalize the house into a formal cognitive substrate.

---

## **16.2 Summarizers as Deterministic State-Transformation Engines**

A summarizer is not a predictive model and not an LLM.
It is a **deterministic, rule-driven state transformer** implemented in Home Assistant using:

* **template sensors**
* **event triggers**
* **guarded scripts**
* **health checks**
* **Zen DojoTools libraries**
* **the Cabinet write API**

This deterministic base ensures that the cognitive system maintains strict reproducibility — a requirement for any safety-critical architecture.

A change in input *must always* lead to the same Kata output.

---

## **16.3 Triggering Model**

Summarizers operate under four primary trigger classes:

### **16.3.1 State-change Triggers**

Any transition in a monitored state field initiates a summarizer run:

* motion changed
* illuminance crossed threshold
* door contact toggled
* water flow > 0
* house mode changed
* room activity inferred

### **16.3.2 Time-based Triggers**

For domains where drift matters, summarizers periodically re-assert Katas every:

* 30–90 seconds (security, presence)
* 5 minutes (water, HVAC)
* 15 minutes (energy)
* 1 hour (daily rollups)

### **16.3.3 Probe Triggers**

Triggered by:

* the Abbot
* nightly health scripts
* manual operator request

### **16.3.4 External Events**

Examples:

* guest arrival
* scheduled rituals
* security arm/disarm actions
* house mode change commands

A summarizer runs whenever the system needs a new truth packet.

---

## **16.4 Input Specification**

Summarizers take inputs from three sources:

### **16.4.1 RoomState**

The real-time, template-sensor-driven map of each room’s perceptual state.

### **16.4.2 Subsystem Sensors**

Energy, water, security, HVAC, lighting, wellness sensors, etc.

### **16.4.3 House Metadata**

* current house mode
* scheduled events
* recent interactions
* authorized users present
* contextual flags set by automations

All inputs are fully deterministic and local—never LLM-derived.

---

## **16.5 Internal Processing Pipeline**

Every summarizer adheres to a strict four-stage pipeline to derive a Kata.

---

### **16.5.1 Stage 1 — Read & Snapshot**

The summarizer script performs a **single, atomic snapshot** of all relevant sensor values.

This uses Home Assistant’s template evaluation engine, which ensures:

* reads are consistent within execution
* values do not race mid-template
* state is frozen during evaluation

This atomic freeze ensures that summarizer output is race-free.

---

### **16.5.2 Stage 2 — Normalization**

Raw values are normalized into standard primitives:

* booleans → stable domain booleans
* numeric ranges → clamped to sane domain bounds
* missing/unknown → normalized to `null`
* enumerations → domain-specific categories
* strings → canonical lowercase identifiers

Example:

| Raw           | Normalized     |
| ------------- | -------------- |
| `"On"`        | `true`         |
| `0.002 W`     | `0`            |
| `"heat_cool"` | `"hvac_mixed"` |
| `"unknown"`   | `null`         |

Normalization enforces type discipline across the cognitive pipeline.

---

### **16.5.3 Stage 3 — Semantic Derivation**

This is the heart of the summarizer.

Domain semantics are computed from raw values. Examples:

* motion + illuminance + occupancy grid → room_activity_level
* door contact + schedule flags → expected_open_state
* water flow + recirculation schedule → water anomaly risk
* entity timings → recently_active, aging buckets, etc.

These derivations convert raw observations into *meaning*.

---

### **16.5.4 Stage 4 — Validation & Guardrails**

Before a Kata is written:

* all fields are sanity-checked
* types are validated
* illegal values are replaced with sentinel warnings
* manifest version is attached
* summarizer version is attached

If the summarizer detects:

* missing critical sensors
* conflicting values
* impossible states
* cabinet GUID mismatch
* ACL conflicts

…it will fail safe:

* write nothing
* emit a zen_event summarizer_error packet
* request inspection or fallback summarizer

This is essential to maintain cognitive integrity.

---

## **16.6 Output Specification: The Kata**

The Kata is a small, immutable JSON packet with:

```
{
  "id": "kata:room.living",
  "version": "1.2.0",
  "timestamp": "2025-11-17T10:04:11-06:00",
  "state": {
      "occupied": true,
      "activity": "high",
      "lighting": "dim",
      "expected_doors": ["front"],
      "unexpected_open": false,
      "climate": {
          "temp_f": 72.1,
          "trend": "stable"
      },
      "error": null
  },
  "source": "summarizer.roomstate",
  "validation": "ok"
}
```

Characteristics:

* **immutable**
* **monotonic**
* **stable under load**
* **schema-validated**
* **linked to a Cabinet drawer**

Every Kata represents a "unit of truth" the cognitive system can rely on.

---

## **16.7 Persistence Guarantees**

Katas are written directly into Cabinets.

A write is only accepted if:

1. **ACL allows it**
2. **Cabinet GUID matches the manifest**
3. **validation signature passes**
4. **key does not violate write policy**

Writes failing these checks produce:

* no overwrites
* a zen_event:error
* an Abbot escalation if persistent

---

## **16.8 Cognitive Load Implications**

By summarizing the environment into Katas:

* Friday’s prompt stays *small and stable*
* LLM context does not explode with raw sensor data
* reasoning becomes deterministic
* the Monastery can compare Katas temporally
* drift detection and anomaly detection become trivial
* computation cost plummets
* privacy and safety are maximized

Summarizers are thus the *nervous system* of Friday’s cognition.

---

## **16.9 Summarizer Versions & Upgrades**

Each summarizer includes:

* `summarizer_version`
* `schema_version`
* `cabinet_version`

Upgrades are safe due to:

* versioned drawers
* backward compatibility
* optional migration routines

If an upgrade detects mismatched structure:

* it writes to a new drawer
* old drawers are retained
* Abbot schedules a migration Kata

This prevents catastrophic schema corruption.

---

## **16.10 Summary**

Version 1 of the Summarizer Engine provides:

* deterministic cognition
* real-time compression
* noise isolation
* safe cabinet writes
* semantic clarity
* temporal continuity

It is not an LLM.
It is a **state machine that produces meaning**, forming the bedrock on which all higher cognition in Friday’s architecture stands.