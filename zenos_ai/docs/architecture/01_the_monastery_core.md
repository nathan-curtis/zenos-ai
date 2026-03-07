# **01. The Monastery Core**

*Component Architecture of Friday's Cognitive Engine*

---

## **1.1 Introduction**

The Monastery is the reasoning substrate of Friday's House. It is not a chat endpoint. It is not a model wrapper. It is a structured cognitive engine composed of discrete, auditable components that transform structured requests into compact, schema-validated reasoning artifacts called Katas.

Friday does not think. The Monastery thinks. Friday supervises, governs, and communicates. She delivers structured queries and consumes the results. The Monastery is where inference happens, where state is synthesized, where complexity is absorbed and returned as something safe to act on.

This chapter describes the components that constitute the Monastery: what they are, what they do, and how they cooperate. The data flow between them is covered in Chapter 4. The scheduling model that drives them is covered in Chapter 6. This chapter establishes the vocabulary.

---

## **1.2 The Monastery as Cognitive Engine**

The Monastery is the cognitive layer that separates Friday's House from a conventional home automation system. Where most systems react to events through rule tables, the Monastery synthesizes state into structured world models and makes those models available as grounded cognitive context.

The Monastery operates inside Home Assistant's event and scripting substrate. It does not require external infrastructure. Every component runs as a Jinja-evaluated script or automation, producing and consuming structured JSON. All reasoning is locally grounded and causally observable.

The Monastery's outputs are always structured. They are always schema-constrained. They are always returnable to a cabinet and retrievable by downstream consumers without parsing ambiguity or freeform interpretation.

The Monastery does not guess. It summarizes.

---

## **1.3 Core Components**

The Monastery is composed of five functional components. Each is discrete. Each has a defined input contract and a defined output contract. Together they form the complete cognitive surface Friday depends on.

### **1.3.1 The Zen Index**

The Zen Index is the associative routing layer. It resolves entities, labels, and cabinet drawers into structured result sets using set logic operations over the Home Assistant label graph.

The Index provides:

* entity resolution from label queries using `AND`, `OR`, `NOT`, and `XOR` operators
* optional expansion of result entities into full semantic records via Inspect
* adjacency computation — which labels co-occur with a matched entity set
* drawer discovery — which cabinet volumes match a label intersection
* universe queries — when no operands are provided, a full label topology view

The Index does not reason. It routes. It answers the question: *what exists here, and what is adjacent to it?* That answer becomes the working dataset for everything downstream.

### **1.3.2 Zen Inspect**

Zen Inspect is the semantic introspection layer. It converts raw entity identifiers into rich, type-safe semantic objects suitable for reasoning consumption.

For each entity, Inspect captures:

* state and temporal metadata (`last_changed`, `last_updated`)
* friendly name and domain classification
* a sanitized attribute map — all values normalized to scalars, mappings, or sequences, with type anomalies corrected
* label membership via `labels(entity_id)`
* cabinet header metadata when the entity is a cabinet sensor: GUID, version, friendly name, flags, and validation signature
* statistics eligibility analysis

Inspect does not interpret. It describes. It answers the question: *what is this entity, in detail, right now?* A Reasoning Unit that calls Index and then Inspect on the result set has a complete, grounded semantic picture of a portion of the home's state.

### **1.3.3 Worker Monks**

Worker Monks are the reasoning execution units. A Monk receives a structured task — a scoped query with defined inputs, a schema, examples, and supplemental context — and invokes a local AI inference worker via an `ai_task.generate_data` entity.

The Monk's job is to transform the working dataset into a schema-constrained JSON Kata. It:

* constructs a complete prompt object from Index and Inspect results, cabinet draws, and prior Katas
* defines the expected output structure explicitly (`structure`, `example_data`)
* submits the prompt to the configured inference backend
* validates and returns the response

Monks are stateless. They do not accumulate memory across calls. Each invocation is fully self-contained. Context arrives with the task; results leave as a Kata.

In Version 1, monks operate as distinct named roles bound to specific Kung Fu components: the Security Monk, the Room Manager Monk, the Energy Monk, and so on. Each component has a defined schema and a defined summarization policy.

Worker Monks emit telemetry on every completion through the event emitter, providing an auditable trail of every inference cycle.

### **1.3.4 Kata Producers**

Kata Producers are the two scripts responsible for initiating and completing summarization cycles. They are the primary interface between the Monastery's internal components and the cabinet storage layer.

**Ninja Summarizer** — `zen_dojotools_ninja_summarizer`

The Ninja Summarizer handles component-local summarization. It is the workhorse of the Kata production pipeline.

For a given Kung Fu component, the Ninja Summarizer:

1. Reads component configuration from the Dojo cabinet and household preferences.
2. Pulls the shared `kata_template.structure` schema from the Kata cabinet.
3. Optionally consults Index, Inspect, or library console output for contextual grounding.
4. Constructs a complete prompt object containing the query, the expected schema, canonical examples, raw review data, and supplemental instructions.
5. Invokes a Worker Monk via `ai_task.generate_data`.
6. Writes the returned Kata into the Kata cabinet as a keyed drawer.
7. Emits a telemetry event recording outcome and key fields.

Each component Kata is compact enough to be included in Friday's active prompt context. The Ninja Summarizer ensures every component maintains a fresh, structured self-model.

**SuperSummary** — `zen_dojotools_supersummary`

The SuperSummary is a higher-order producer. It does not reason about individual components; it consolidates all active component Katas into a single whole-home cognitive snapshot called `ZEN_SUMMARY`.

The SuperSummary:

1. Detects which Kung Fu master switches are active using the `Kung Fu System Switch` label.
2. Reads each active component's current Kata from the Kata cabinet.
3. Optionally injects system-wide context: Friday's purpose, directives, cortex, and the previous `ZEN_SUMMARY`.
4. Constructs a consolidating prompt instructing normalization and cross-component synthesis.
5. Invokes a Worker Monk with the consolidated payload.
6. Writes the result as the `ZEN_SUMMARY` drawer in the Kata cabinet.
7. Emits a completion event listing active components and write status.

The `ZEN_SUMMARY` is Friday's primary high-level world model. It is the cognitive artifact she loads most frequently when constructing context for a conversation or decision.

### **1.3.5 Kronk — The Curator**

Kronk is the Curator of the Monastery. Where Friday governs and communicates, Kronk manages the internal integrity of the cognitive engine. He is responsible for:

* overseeing the health of the Kata cabinet and its stored summaries
* coordinating Monk task assignment and routing through the Abbot
* tracking which components are active and which Katas are stale
* managing the summarization schedule in coordination with the Scheduler
* operating the long-form reflection path for deep cognitive cycles

Kronk does not produce Katas directly. He ensures that the conditions for Kata production are met and that the resulting artifacts are coherent, correctly stored, and retrievable. He is the operational intelligence behind the Monastery's self-management.

In Version 1, Kronk's behaviors are embedded in the Scheduler and administrative tooling. As the system matures toward GA, Kronk's role as a distinct supervisory agent will be formalized with its own identity, cabinet, and governance boundary.

---

## **1.4 The Summarization Pipeline**

The components described above cooperate in a single canonical pipeline. This pipeline is the Monastery's primary cognitive output and the mechanism by which Friday maintains a coherent, grounded world model across sessions.

```
Event or Schedule
      ↓
  Scheduler (Abbot)
      ↓
  Ninja Summarizer
      ↓
    Zen Index → Zen Inspect → Worker Monk
      ↓
    Kata written to Kata Cabinet
      ↓
  SuperSummary (on schedule or threshold)
      ↓
    ZEN_SUMMARY written to Kata Cabinet
      ↓
  Friday loads ZEN_SUMMARY at prompt compile time
```

Every step in this pipeline is observable via the Home Assistant event bus. Every Kata is inspectable in the Kata cabinet. The entire cognitive history of the system is recoverable from cabinet state.

The pipeline runs continuously. The Scheduler drives it on a 15-minute cycle for all active components, with additional triggers from meaningful state changes — occupancy shifts, alarm transitions, door events, energy anomalies. Each trigger produces fresh Katas, which accumulate into an updated `ZEN_SUMMARY` on the next consolidation pass.

Nothing in this pipeline is opaque. Nothing is irreproducible.

---

## **1.5 The Kata Cabinet**

The Kata cabinet is the Monastery's primary storage surface. It is a standard ZenOS FileCabinet volume used exclusively for:

* one drawer per active Kung Fu component, keyed by component slug
* the `ZEN_SUMMARY` drawer — the whole-home consolidation artifact
* the `kata_template` drawer — the shared schema used by all Kata Producers
* the `zen_scheduler` drawer — the Abbot's operational record

The Kata cabinet is read-only for Friday's reasoning path. It is written only by Kata Producers and the Scheduler. The FileCabinet GC manages expiry and lifecycle for stale or timed-out Katas.

The `AI_Cabinet_VolumeInfo` drawer is always protected and never touched by the GC or any Monastery component.

---

## **1.6 Cognitive Boundaries**

The Monastery enforces a strict boundary between what it summarizes and what it governs.

The Monastery:

* receives Index and Inspect results — derived, sanitized representations of home state
* reads cabinet drawers through the FileCabinet interface
* produces Katas constrained by defined schemas
* writes results only into designated cabinet drawers

The Monastery does not:

* hold raw identity credentials
* access safety-critical control sensors directly
* issue device commands
* maintain freeform conversational memory
* communicate with external services independently

These boundaries are architectural, not advisory. They are enforced by the design of the tool layer, not by runtime policy checks. A Monk cannot reach what the system has not given it access to in the first place.

This is not a limitation. It is the mechanism by which the Monastery can reason freely within its domain without risk of destabilizing the system it serves.

---

## **1.7 Summary**

The Monastery Core is a five-component cognitive engine:

| Component | Role |
|-----------|------|
| Zen Index | Associative routing and entity resolution |
| Zen Inspect | Semantic introspection and attribute normalization |
| Worker Monks | Inference execution and Kata production |
| Kata Producers | Summarization pipeline orchestration (Ninja + SuperSummary) |
| Kronk | Curatorial oversight and Monastery integrity |

Together they implement the summarization pipeline that produces Katas — the structured, schema-constrained cognitive artifacts Friday depends on to reason about her world.

The chapters that follow describe the full architectural context (Chapter 2), the cognitive first principles (Chapter 3), the data flow through this pipeline (Chapter 4), and the scheduling model that drives it (Chapter 6).

The Monastery is where thinking happens. Everything else is delivery.
