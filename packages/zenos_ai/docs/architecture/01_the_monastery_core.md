# **4. Cognitive Data Flow (Version 1 Architecture)**

The Monastery executes cognition through a structured, deterministic pipeline. The pipeline is composed entirely of Home Assistant’s event bus, the Zen DojoTools execution suite, identity validation, and local reasoning workers known as monks. All cognitive processing in Friday’s House originates as a structured request, passes through the scheduler (the Abbot), is routed to a suitable reasoning unit, and returns a Kata: a formally defined cognitive artifact.

This section describes the data flow, the transformation contracts, the routing model, and the guarantees that underpin Version 1 of the architecture.

---

## **4.1 Query Ingestion**

### Conceptual Model

A reasoning cycle begins with the creation of a **Query Packet**.
The Query Packet is the atomic cognitive unit used for all inference, reflection, and summarization tasks. It holds:

* origin identity
* intent classification
* explicit scope definition
* correlation identifier
* temporal bounds
* security credentials
* user or subsystem parameters

The packet must be well-formed before entering the system.

### Implementation Details

In Version 1, ingestion is performed through a Home Assistant event:

```
event_type: zen_kata_request
event_data:
  origin: "friday"
  intent: "context_summary"
  scope:
    entities: [...]
    labels: [...]
  correlation_id: "<uuid>"
  deadline: 2.0
  identity_token: "<session_token>"
```

Query Packets may originate from:

* Friday’s Prompt Loader
* reflex triggers produced by RoomState
* the Abbot during scheduled sweeps
* direct requests from components
* user-invoked tools
* subsystem health monitors

Home Assistant’s event queue preserves ordering and guarantees deterministic routing behavior under load.

---

## **4.2 Validation and Routing**

### Conceptual Model

After ingestion, the system executes a **Validation and Routing Stage**.
This stage enforces:

1. structural correctness
2. semantic correctness
3. privilege verification
4. scope feasibility
5. complexity scoring

The Abbot uses these attributes to select an appropriate reasoning unit.

### Implementation Details

Validation occurs inside a guarded Zen DojoTools script. This script performs:

* schema validation
* identity and ACL verification
* correlation ID extraction
* privilege checks against the Identity Cabinet
* from_json-guarded parsing
* validation of the deadline
* confirmation that the request is internally consistent

Routing is based on a weight model encoded into a choose structure:

```
- choose:
    - conditions:
        - "{{ intent == 'environment_scan' and complexity_score <= 2 }}"
      sequence:
        - event: zen_monk_light_task
          event_data:
            correlation_id: "{{ cid }}"
            ...
    - conditions:
        - "{{ complexity_score > 2 and complexity_score <= 5 }}"
      sequence:
        - event: zen_monk_medium_task
          event_data:
            ...
    - conditions:
        - "{{ complexity_score > 5 }}"
      sequence:
        - event: zen_monk_heavy_task
```

The event bus becomes the message transport layer for reasoning assignments.

Each monk represents a resource class:

* **Light monk**: low-complexity label extraction, shallow index queries
* **Medium monk**: multi-entity joins, cabinet-aware expansions
* **Heavy monk**: long-form reflection, multi-component analysis

The Abbot guarantees stable routing decisions across identical inputs.

---

## **4.3 Reasoning Unit Execution**

### Conceptual Model

A Reasoning Unit transforms the Query Packet into a Kata through a series of structured operations:

* acquisition of required state
* extraction of relevant labels and drawers
* attribute inspection
* adjacency expansion in the associative graph
* validation of cabinet structures
* temporal reasoning
* contradiction detection
* bounded inference

Every reasoning unit follows a deterministic workflow.

### Implementation Details

Reasoning is performed with two foundation tools:

### **Zen Index**

Provides:

* entity resolution
* adjacency discovery
* label sets
* drawer retrieval
* optional expansion to full entity structures

### **Zen Inspect**

Provides:

* safe attribute extraction
* type-normalized attribute dictionaries
* cabinet header detection
* statistics eligibility assessment
* device and area metadata when requested

A typical reasoning run follows this pattern:

1. Monk receives zen_monk_task event.
2. Monk invokes `script.dojotools_zen_index` with expand enabled.
3. Index returns a dataset containing entity sets, adjacency sets, and drawer matches.
4. Monk invokes Zen Inspect for any entity that requires deeper structural validation.
5. Monk constructs a rationale chain using a predefined inference contract.
6. Monk emits a `zen_kata_response` event containing the completed Kata.

Every step is observable and auditable.

---

## **4.4 Kata Formation**

### Conceptual Model

A **Kata** is the canonical cognitive unit.
It contains:

* the result
* the rationale chain
* anchor entities
* anchor labels
* expiration rules
* confidence levels
* lineage metadata

Kata provide a compact but fully grounded representation of state or reasoning.

### Implementation Details

Kata are emitted as:

```
event_type: zen_kata_response
event_data:
  correlation_id: "<uuid>"
  result:
    summary: "Kitchen dormant for 18 minutes"
    anchors:
      entities: [...]
      labels: [...]
  rationale: [...]
  valid_until: "2025-11-18T10:42:00"
  confidence: "bounded-medium"
```

The receiving script verifies:

* correlation
* schema conformance
* valid timestamp
* expected anchor structures

The Kata Manager makes the Kata available to Friday’s working memory subsystem.

---

## **4.5 Reintegration into Working Memory**

### Conceptual Model

Friday maintains coherence through a time-ordered memory pipeline:

* reflex state (RoomState)
* working context
* component-level Kata
* whole-home summary
* long-term annotated memory

New Kata do not accumulate indefinitely. They replace older or stale entries based on recency, relevance, and structural identity.

### Implementation Details

Reintegration uses:

* working memory sensors
* cabinet-backed state for persistent entries
* time-indexed draws during Prompt Loader execution
* Abbot-triggered summarizer passes for refresh

Example reintegration:

```
event_type: zen_update_context
event_data:
  field: "environment_snapshot"
  value: <kata>
```

Friday retrieves the updated field on the next context synthesis cycle:

```
state_attr('sensor.friday_context', 'environment_snapshot')
```

The system maintains overall context coherence by prioritizing the most recent Kata within each domain.

---

## **4.6 Architectural Guarantees**

### Determinism

Achieved through:

* event ordering
* correlation identifiers
* stable routing logic
* non-probabilistic execution in Index and Inspect

### Traceability

Achieved through:

* explicit rationale chains
* event logs
* entity adjacency projections
* cabinet signatures
* time-stamped Kata

### Bounded Inference

Controlled through:

* explicit deadlines
* complexity scoring
* Abbot-aware routing
* index and inspect timeouts

### Separation of Concerns

Enforced through:

* Query Packet isolation
* dedicated reasoning units
* identity and cabinet routing
* independent storage layers

### Reflective Coherence

Maintained through:

* regular summarizer sweeps
* Kata replacement rules
* expiration and freshness constraints
