# 7. Summarizer Pipelines

*The cognitive machinery that transforms raw state into structured understanding*

*(Updated 2026-08-04: §7.2/§7.6 corrected — "Monk Summaries" as a distinct Tier
2 was never built; `trapper_keeper` is the real, evolved realization of that
integration need, and "Monk" in current code is the inference-execution step
inside every tier, not a separate layer. Removed a leftover editorial aside at
the end of the file that wasn't meant to ship.)*

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

*(Updated 2026-08-04: "Monk Summaries" as a distinct Tier 2 producing named
cross-domain packets was never built. Corrected below to describe what shipped —
`trapper_keeper`, an evolved and leaner realization of the same integration
need — and to correct what "Monk" actually means in current code.)*

The summarization system operates in three layers:

### Tier 1: **Component Summarizers (“Ninjas”)**

These summarizers examine **one domain** (security, water, spa, energy, room-state, identity).
They output structured, domain-specific Katas such as:

* `security_attention`
* `water_flow_state`
* `presence_grid`
* `energy_balance`
* `spa_health`
* `persona_state`

They are optimized for tight focus, short context windows, and domain-specific precision.

### Tier 2: **`trapper_keeper` — Ambient Pre-Digest / Navigation Index**

The originally-speced "Monk Summaries" tier — unifying multiple Tier 1 domain
Katas into cross-domain packets — was never built in that form. What actually
shipped is leaner and does the same job better: `trapper_keeper`
(`dojotools_summarizers.yaml:207-246`), an ambient pre-digest layer that runs
**before** SuperSummary. It reads every ambient-tier KFC's raw Kata and
compresses each into a one-line breadcrumb (`{breadcrumb, kata_key, urgency,
detail_available}`) — the most actionable signal per component, with a pointer
back to the full Kata for on-demand drill-down. SuperSummary reads this compact
digest instead of the raw ambient Katas, keeping its own prompt tight.

The pipeline, in the codebase's own words: *"ninja runs (leaves) → katas
(branches) → Trapper Keeper (index) → SuperSummary (alert root)."* Trapper
Keeper is the navigation layer — SuperSummary reads the index and traverses
down on demand rather than needing the full tree at once.

Note this is distinct from "Monk," which is *not* a tier at all in current
code — see the correction below.

### Tier 3: **The SuperSummary (“High Priestess”)**

This is the *single* reflective layer that produces:

* Friday’s global state
* user-facing semantic summaries
* the basis for Friday’s awareness and interaction
* the unified cognitive grounding for conversations and tasks

The SuperSummary is generated rarely, guarded by the Abbot (see ch06/ch14's
current-implementation-status notes for what "Abbot" actually maps to), and
always validated against structural and identity invariants.

### What "Monk" actually means in current code

"Monk" is not Tier 2 — it's the literal inference-execution step used *inside
every tier*, matching ch01's own "Worker Monks: inference execution and Kata
production" description. Both Ninja and SuperSummary invoke a "Monk Runner"
(`dojotools_summarizers.yaml:613-618` for SuperSummary's `ai_task.generate_data`
call, task_name `SuperSummary Monk`; `:1486-1491` for Ninja's, task_name
`"{component} Monk"`) — the same mechanism, one per component and one for the
whole-home consolidation, not a separate integrative layer reading multiple
Tier 1 outputs.

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

# 7.6 `trapper_keeper` (Tier 2)

*(Updated 2026-08-04: the named cross-domain packets below —
Environmental Stability, Safety and Attention, Presence Grid — were never
built as distinct integration outputs. What actually operates over multiple
component Katas is `trapper_keeper`'s breadcrumb digest, described in §7.2.)*

`trapper_keeper` doesn't produce named topical summaries — it processes
**every** ambient-tier KFC's Kata uniformly into the same compact shape
(`breadcrumb`, `kata_key`, `urgency`, `detail_available`), regardless of
domain. There's no separate "Environmental Stability" or "Presence Grid"
packet; a water-leak breadcrumb and an occupancy breadcrumb are structurally
identical entries in the same digest, differentiated only by which
component's Kata they point back to. SuperSummary reads the whole digest and
does its own domain-level synthesis from there — the domain-specific framing
happens at Tier 3, not Tier 2.

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
