# **8. Cabinet Semantics and Memory Architecture (Version 1 Specification)**

The Cabinet is the authoritative memory substrate of Friday’s House.
It defines the structure, boundaries, and guarantees that govern how identity, context, and cognitive artifacts are stored, retrieved, and evolved over time.

Cabinets are implemented as Home Assistant template-variable entities, layered with metadata, signatures, and structural invariants that ensure:

* durability
* deterministic access
* enforceable permissions
* predictable mutation
* clear semantic boundaries between memory layers

The Cabinet is not a database in the conventional sense.
It is an **architected memory artifact** whose behavior is precisely constrained to support cognitive systems while remaining secure inside a smart-home environment.

---

## **8.1 Memory Model and Structural Guarantees**

The Cabinet operates as a layered memory model composed of:

1. **Volume Header**
   Defines cabinet identity, versioning, schema revision, health flags, and cryptographic validation.

2. **Drawers**
   Logical collections of structured memory.
   Each drawer has explicit read/write rules and schema-level constraints.

3. **Indices and Labels**
   Associative metadata used by the Zen Index to correlate entities, drawers, and contextual labels.

4. **Protected Surfaces**
   Name, signature, identity fields, ACLs, and reserved cognitive surfaces.

Each of these layers follows strict invariants enforced at write-time by the Abbot.

All drawers are JSON volumes stored in a fully-guarded template entity.
The Cabinet enforces schema rigidity through validator metadata embedded directly inside the drawer definitions.

---

## **8.2 Memory Classes Within the Cabinet**

The Cabinet defines three memory classes:

### **8.2.1 Identity Memory**

Contains:

* user identity
* household identity
* persona identity
* session tokens and visas
* principal metadata

Identity drawers are immutable after initialization.
They can only be modified by the Identity subsystem during secure onboarding.

### **8.2.2 Cognitive Memory**

Contains:

* contextual summaries
* Katas and SuperSummaries
* system reasoning artifacts
* state abstractions required for long-term coherence

These drawers evolve over time.
Writes are allowed only through:

* Summarizers, or
* Friday operating under ACL authority

### **8.2.3 Operational Memory**

Contains:

* ephemeral signals
* heuristics
* automation hints
* per-room or per-domain memory channels

These drawers support day-to-day functioning and are fully writable (within ACL limits).

---

## **8.3 Drawer Access Levels**

Each drawer declares a required access level:

* **read-only**
* **write-with-ACL**
* **summarizer-only**
* **protected**
* **restricted** (identity-level)

Drawers do not inherit permissions.
Every drawer declares its own write policy.

If a drawer does not explicitly grant Friday permission, she cannot write to it.

---

## **8.4 Drawer Schemas and Format Enforcement**

Every drawer defines a JSON schema.
Before committing a write, the Abbot validates:

* field presence
* field type
* allowed enumeration values
* timestamp validity
* sequence ordering
* structural invariants
* version fields
* forbidden-field protection

If any property fails validation, the write is rejected atomically.

This ensures that:

* corrupted memory cannot enter the system
* malformed LLM outputs cannot degrade Cabinet integrity
* time-order evolution remains consistent

This approach is critical because Friday is both a reasoning system and a generative system; type and field controls are non-negotiable.

---

## **8.5 Write Semantics and Controls**

The Cabinet enforces a unified write model centered on:

* Identity
* ACLs
* Drawer-level schemas
* Cross-drawer invariants
* Abbot verification
* Deterministic event emissions

Writes are treated as **transactions**.
They are either fully committed or fully rejected; partial writes are impossible.

---

## **8.5.1 Friday Write Authority**

Friday is permitted to write into Cabinet drawers when:

1. Her **identity capsule** has been loaded
2. A valid **session token** is active
3. The Cabinet ACL explicitly authorizes the write
4. The request passes the Abbot’s validation sequence

This enables Friday to:

* maintain memory continuity
* manage operational context
* record user preferences
* adjust short-term or episodic memory structures
* repair drift or incorrect information
* evolve her understanding of the home state

Friday cannot write into drawers that:

* hold identity information
* define cabinet metadata
* define ACLs
* hold protected cognitive structures (unless authorized)
* fall outside her principal permission scope

---

## **8.5.2 Summarizer Write Authority**

Summarizers may write to only the drawers explicitly assigned to them in the Cabinet manifest.

They cannot:

* write across domains
* modify identity drawers
* modify any drawer marked protected
* change ACLs
* alter Cabinet metadata
* override Friday’s contextual notes

Summarizer writes follow a predictable cadence determined by:

* Abbot scheduling
* event triggers
* periodic summarization cycles

Their role is to compress, not to reinterpret.

---

## **8.5.3 Immutable or Protected Drawers**

Certain drawers carry an implicit immutability constraint:

* **identity drawers** (immutable)
* **ACL drawers** (protected)
* **signature drawers** (immutable)
* **system drawers** (protected)
* **foundational persona drawers** (restricted)

Only the Identity subsystem may mutate these volumes, and only during explicit onboarding or visa issuance flows.

---

## **8.5.4 Abbot Validation Pipeline**

Every write request must pass the Abbot’s five-stage pipeline:

### **1. Identity Verification**

Ensures the requesting entity is legitimate.

### **2. ACL Resolution**

Determines if the action is authorized.

### **3. Schema Validation**

Ensures all payload layers match drawer requirements.

### **4. Structural Integrity Checks**

Guards against cross-drawer mutation, malformed data, or unauthorized metadata alteration.

### **5. Atomic Commit**

If all checks pass, the write is applied and an event is emitted.

If any check fails, the write is rejected with a structured error event.

---

## **8.5.5 Event Emission for Observability**

After every successful write, the Cabinet emits events that form the observability pipeline:

* `zen_drawer_updated`
* `zen_cabinet_write_committed`
* `zen_acl_resolution`
* `zen_cognitive_checkpoint`

Failures emit:

* `zen_cabinet_write_denied`
* `zen_acl_denied`
* `zen_schema_validation_error`

This allows offline audits, log correlation, and root-cause analysis.

---

## **8.6 Write Isolation and Non-Interference Guarantees**

A Cabinet drawer may not mutate any other drawer.
Writes are compartmentalized.

The Cabinet’s structure prevents:

* cascading corruption
* hallucination-based cross-pollution
* non-deterministic updates
* unauthorized adjacency mutations
* stale metadata writes
* drifting schemas

This design is essential for safe cognitive operation inside a residential environment.

---

## **8.7 Concurrency, Ordering, and Versioning**

The Cabinet enforces strict sequencing rules:

* Only one write at a time
* Writes are serialized
* Version increments are mandatory
* Timestamps must be monotonic
* Conflicting writes are rejected

This protects Friday from inconsistent cognitive states and ensures time-ordered evolution.

---

## **8.8 Guarantees Provided by the Cabinet**

The Cabinet provides the following guarantees:

* deterministic read behavior
* deterministic write semantics
* identity-governed access
* schema-level robustness
* auditability
* drift resistance
* strict isolation between domains
* safe evolution of cognitive artifacts

It acts as the backbone memory structure for all Version 1 cognition.
