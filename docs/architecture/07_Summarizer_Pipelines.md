# 7. Summarizer Pipelines

*The cognitive machinery that transforms raw state into structured understanding*

Friday’s intelligence does not emerge from a single monolithic model or a single context window. It arises from **a layered summarization pipeline** that progressively refines raw environmental data into structured Katas, culminating in Friday’s unified understanding of the current state of the House.

The summarizer pipeline is the cognitive engine of the Monastery. It transforms sprawling environmental chaos into stable, analyzable packets of meaning.

This chapter dives into the mechanics, invariants, safety constraints, and algorithmic structure of all summarizers.

---

# 7.1 Purpose of Summarizers

Summarizers serve three critical roles:

1. **Compression**
   Reduce large state spaces into compact, manageable packets.

2. **Stability**
   Ensure Friday has a stable grounding for reasoning despite noisy or inconsistent inputs.

3. **Boundaries**
   Define strict limits on what Friday can infer or imagine, preventing the system from drifting into hallucination.

Every summarizer produces structured data rather than prose. Every output must be machine-validated and predictable.

Summarizers are not storytellers.
They are engineers of meaning.

---

# 7.2 Three-Tier Architecture

The summarization system operates in three layers:

### Tier 1: **Component Summarizers (“Ninjas”)**

These summarizers examine **one domain** (security, water, hot tub, energy, room-state, identity).
They output structured, domain-specific Katas such as:

* `security_attention`
* `water_flow_state`
* `presence_grid`
* `energy_balance`
* `spa_health`
* `persona_state`

They are optimized for tight focus, short context windows, and domain-specific precision.

### Tier 2: **Integrative Summaries (“Monk Summaries”)**

These unify multiple Tier 1 domain Katas into cross-domain packets:

* `home_safety_unified`
* `environmental_stability`
* `operational_health`
* `occupancy_grid`

These operate with higher-level context but still avoid global interpretation.

### Tier 3: **The SuperSummary (“High Priestess”)**

This is the *single* reflective layer that produces:

* Friday’s global state
* user-facing semantic summaries
* the basis for Friday’s awareness and interaction
* the unified cognitive grounding for conversations and tasks

The SuperSummary is generated rarely, guarded by the Abbot, and always validated against structural and identity invariants.

---

# 7.3 Summarization Invariants

All summarizers must obey strict rules:

### 7.3.1 Deterministic Input

Summaries must use:

* validated, inspected entity graphs
* stable cabinet data
* timestamped RoomState objects
* precise environmental readings
* deterministic policy data

No summarizer may rely on arbitrary model inference outside explicit instructions.

### 7.3.2 Deterministic Output

Outputs must be:

* JSON-serializable
* schema-conforming
* versioned
* referentially stable
* validated before cabinet writes

Any deviation blocks cabinet updates.

### 7.3.3 Zero Hallucination Tolerance

Summarizers are forbidden to:

* invent entities
* assign motives
* speculate beyond measurable inputs
* extend states outside valid domains

Consistency is enforced through both templated guards and the Abbot’s validation layer.

### 7.3.4 Strict Isolation Between Layers

Tier 1 summarizers cannot read Tier 2 or Tier 3 output.
Tier 2 summarizers cannot read Tier 3 output.
Tier 3 is read-only upstream.

This constraint prevents backflow loops that destabilize reasoning.

---

# 7.4 Operational Flow of a Summarizer Run

The following pipeline describes exactly how any summarizer operates.

### Step 1 — Trigger

A state change or a scheduled window activates the Abbot.

### Step 2 — Component Eligibility

The Abbot determines which summarizers are eligible based on:

* component policy
* cooldown timers
* quiet hours
* dependency consistency
* trigger relevance

### Step 3 — Data Aggregation

The Abbot collects:

* entity snapshots
* room summaries
* cabinet drawers
* environmental state
* labels, tags, exceptions
* domain metadata

Each summarizer receives only the subset it needs.

### Step 4 — Template Construction

A precise, structured prompt is built:

* context header
* domain-specific schema and constraints
* explicit value ranges
* error-handling instructions
* clear JSON template for output

The template is deterministic across runs.

### Step 5 — Execution

The model performs the summarization using the domain slice.
This is run either on:

* local inference
* Abbot-assigned remote inference
* higher-tier engines (DGX, cloud) for complex domains

### Step 6 — Validation and Sanitization

The Abbot validates:

* JSON structure
* adherence to schema
* presence of mandatory fields
* absence of illegal fields
* correct typing
* value ranges
* domain logic constraints

Malformatted outputs cause immediate rejection.

### Step 7 — Cabinet Write

On successfully validated output:

* the output is versioned
* timestamps are recorded
* the new Kata overwrites the old
* an event is emitted for traceability

### Step 8 — Upstream Summarizer Eligibility

Tier 2 summarizers may now run if policy conditions are met.

The process repeats through each layer until the SuperSummary is produced.

---

# 7.5 Component Summarizers (Tier 1)

These are domain specialists. Examples:

## **7.5.1 Water Manager Summarizer**

Tracks:

* water flow events
* nominal vs anomalous usage
* leak suspicion scores
* stabilization periods

Outputs include:

```
{
  "flow_state": "normal|suspicious|leak_detected",
  "avg_flow_last_10m": <float>,
  "anomaly_confidence": <0-1>,
  "last_change": "<timestamp>"
}
```

## **7.5.2 Security Manager Summarizer**

Tracks:

* door/window activity
* attention candidates
* entry control state
* contextual noise filters

## **7.5.3 RoomState Summarizers**

Tracks:

* occupancy
* lighting
* power draw
* noise patterns
* motion patterns
* environmental stats

Outputs are used directly by presence grids and activity mapping.

## **7.5.4 Spa Manager Summarizer**

Tracks:

* pump cycles
* heater load
* water chemistry
* error codes
* service schedules

Each of these feeds into the integrative layer.

---

# 7.6 Integrative Summaries (Tier 2)

These operate over multiple component Katas.

Examples:

### 7.6.1 Environmental Stability Summary

Integrates:

* humidity
* temperature gradients
* HVAC loads
* window/door states
* occupancy

Used for energy modeling and anomaly detection.

### 7.6.2 Safety and Attention Summary

Integrates:

* water leaks
* security attention candidates
* power anomalies
* spa alerts

Used for user notifications and automated escalations.

### 7.6.3 Presence Grid Summary

Integrates:

* room-by-room occupancy
* motion patterns
* BLE or UWB presence data
* noise or light transitions

Used for Friday’s contextual awareness (“Nathan is probably in the den”).

---

# 7.7 The SuperSummary (Tier 3)

The SuperSummary is the apex summary:

* integrates all Katas
* reconciles all anomalies
* consolidates environmental, safety, presence, and identity slices
* maintains a chronological chain
* stabilizes Friday’s working memory

It produces:

```
{
  "home_state": { ... },
  "safety_state": { ... },
  "presence_map": { ... },
  "energy_state": { ... },
  "spa_state": { ... },
  "identity_constraints": { ... },
  "timestamp": "<ISO8601>",
  "version": "X.Y.Z"
}
```

This becomes the cognitive foundation for every interaction, every answer, every decision Friday makes.

The SuperSummary is never speculative.
It is only integrative.

---

# 7.8 Error Handling and Guard Rails

Summarizers must handle four failure modes:

### 7.8.1 Invalid JSON

Immediate rejection.
Abbot logs failure and freezes component.

### 7.8.2 Missing Required Fields

Rejected with detailed error output so engineers know precisely what broke.

### 7.8.3 Inference Drift

If the model invents fields, entities, or values outside a domain:

* output is quarantined
* component is locked
* Abbot emits a structured warning

### 7.8.4 Identity Violations

If a summarizer touches a protected identity capsule:

* full system lockdown
* high-priority alert
* Friday notifies the owner with context and time

The cognitive system must never self-modify identity or permissions.

---

# 7.9 Future Directions

Future versions will include:

* multi-identity SuperSummaries
* per-user cognitive slices
* per-zone reasoning modes
* memory indexing tied to identity
* deeper reflection layers if identity permits
* session-token qualified tool interaction

Version 1 builds the safest possible foundation for these future developments.

Note: Monk Tier 2 Summarization is planned for v.Next

Let me know when you want **08_Kata_Cabinet.md** and we’ll keep the directory moving.
