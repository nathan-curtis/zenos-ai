# 18. The Context Frame: Friday’s Active Cognitive Substrate

Friday’s operational awareness is defined by a single integrated construct called the **Context Frame**. The Context Frame is assembled during each reasoning cycle from authoritative system components and delivered to Friday as structured input. It provides a live, coherent, and fully grounded representation of the household environment and the actions that may be taken upon it.

The Context Frame consists of five primary sources:

1. **Kata** – Global summarization object
2. **RoomState** – Canonical room semantics
3. **LiveState** – Real-time entity truth from Home Assistant
4. **Identity Capsule** – Authenticated user and assistant identity model
5. **Tool Inventory** – Curated, ACL-gated action surface (up to 128 tools per context)

Each component contributes a distinct class of meaning. None subsumes the others; instead, they function as orthogonal axes that jointly create Friday’s world model.

---

## 18.1 Structural Overview

The Context Frame is assembled through a deterministic process:

1. **The Abbot** validates summarizer outputs and selects the active Kata.
2. **RoomState** provides the semantic layout of the house: rooms, domains, and roles.
3. **LiveState** is drawn from Home Assistant’s state machine as the authoritative snapshot of current conditions.
4. **The Identity Capsule** resolves who is present, which cabinet(s) represent them, what privileges they hold, and which assistant persona is active.
5. **The Tool Inventory** is filtered according to identity, ACLs, security posture, and current cognitive task, then injected as an explicit list of callable tools (with metadata) into the context.

The prompt loader folds these into a single, hierarchically structured Context Frame. Every reasoning step Friday performs is grounded in this composite object.

---

## 18.2 Components of the Context Frame

### 18.2.1 The Kata (System-Level Summary)

The **Kata** is the primary cognitive object in Friday’s world model. It encodes:

* System-level view: `system.cortex`, OS/build versions, health
* Home-level status: global `status`, `risks`, and recent `events`
* Active components: per-subsystem summaries keyed by component id
* A global summary bundle:

  * `summary`, `attention`, `objective`, `subjective`
  * `urgency`, `confidence`, `criticality`
  * `emotion_queue` and `errors`
  * `timestamp` and `stale_after` bounds

This structure provides a single, queryable, machine-readable object that expresses what matters now, why it matters, and how confident the system is in that assessment.

### 18.2.2 RoomState (Semantic Map)

**RoomState** supplies spatial semantics and role definitions, not interpretation. It provides:

* A stable list of rooms (e.g., `living_room`, `kitchen`, `mbed`, `garage`)
* Per-room slots for key domains:

  * motion, occupancy, lighting, blinds, windows, doors
  * climate (temperature, humidity, comfort band)
  * AV/media presence
  * water-related surfaces where relevant (spa, bath, laundry)
* A canonical mapping from raw entity ids to room-level fields

In practice, RoomState is exposed via a template sensor that composes the live Home Assistant states into a single JSON object per room or per household view. This object is:

* **Real-time** – it reflects current HA states, not a post-processed history
* **Stable** – keys and structure remain constant even if underlying entities change
* **Semantic** – each slot has a clear meaning independent of specific devices

Friday reads RoomState directly. The design goal is not to hide reality but to **stabilize the vocabulary** with which the home describes itself.

### 18.2.3 LiveState (Real-Time Home Assistant Truth)

**LiveState** is the uncompressed substrate of truth:

* Every entity’s state and attributes
* Timestamps (`last_changed`, `last_updated`)
* Device- and area-level metadata (via Inspect/Index where needed)
* The current state of automations, scripts, and key binary conditions

Where RoomState provides *“this is how the house describes its rooms”*, LiveState provides *“this is what Home Assistant currently believes to be true.”* Both are visible to Friday:

* RoomState gives a normalized, schema-first map.
* LiveState provides the raw, debuggable state surface underlying that map.

### 18.2.4 Identity Capsule (User, Assistant, and Security Context)

The **Identity Capsule** is built by the Zen DojoTools Identity script and associated cabinet drawers. It provides:

* Per-person GUID, resolved from essence and cabinet metadata
* Household / family / assistant cabinets and their relationships
* ACL policies for access to tools, data, and actions
* “Visa”-style constructs:

  * issuance of a ticket-granting token (TGT)
  * mapping from that TGT to permitted tool families
  * bindings between person cabinets and assistant cabinets
* Friday’s own identity and persona essence:

  * her GUID
  * her role and scope in this household
  * her constraints and responsibilities

The Identity Capsule is injected into the Context Frame as both:

* a structured identity map (for machine use), and
* a compact narrative header (for human-aligned clarity in the prompt).

This ensures that every reasoning step is bound to a specific, authenticated human and a specific assistant persona, under explicit access policies.

---

### 18.2.5 Tool Inventory and Invocation Surface

The final core component is the **Tool Inventory**: a curated, ACL-filtered list of actions that Friday may invoke during a reasoning cycle.

This inventory is derived from:

* Zen DojoTools manifests (e.g., cabinet drawers that describe tools)
* Home Assistant scripts and automations advertised as tools
* External services and agents represented as callable endpoints
* Per-tool metadata describing:

  * `id` / canonical name
  * category (e.g., `read`, `write`, `control`, `summarize`, `external`)
  * risk level and safety tier
  * allowed actors (via ACL and identity bindings)
  * required authentication (secure mode flags, TGT presence)
  * rate limits or guardrails where applicable

Because the context window is finite, the system maintains a **bounded tool surface**:

* The Abbot and Identity Capsule collaborate to select **up to 128 tools** for the current reasoning session.
* Selection is **task-aware**: summarization-focused calls prioritize summarizers, FileCabinet, Identity, Index, Inspect, and a small set of control tools.
* Selection is **identity-aware**: a guest will see fewer and safer tools than the system owner; privileged energy or security tools require explicit secure-mode elevation.

Within the Context Frame, tools are exposed as a well-structured list, for example:

```json
{
  "tools": [
    {
      "id": "zen_index",
      "kind": "read",
      "scope": "state_and_labels",
      "requires_secure_mode": false,
      "acl": ["owner", "household_member", "assistant"],
      "summary": "Query labeled entities, cabinets, and adjacent labels."
    },
    {
      "id": "ninja_summarizer",
      "kind": "summarize",
      "scope": "component_kata",
      "requires_secure_mode": true,
      "acl": ["owner", "assistant"],
      "summary": "Produce a Kata-compliant summary of a Kung Fu component."
    },
    {
      "id": "filecabinet_update",
      "kind": "write",
      "scope": "kata_drawers",
      "requires_secure_mode": true,
      "acl": ["owner", "assistant"],
      "summary": "Update a cabinet drawer with validated, timestamped content."
    }
    // … up to 128 entries
  ]
}
```

Friday does not need to infer what she *might* be able to do. The action surface is enumerated, finite, and directly bound to identity and ACLs.

---

## 18.3 Assembly Process

During each reasoning cycle, the Context Frame is assembled as follows:

1. **Kata Selection**

   * The Abbot identifies the relevant summarizer output (e.g., ZEN_SUMMARY or a component-specific Kata) from the Kata Cabinet.
   * Validity is checked via timestamps and schema adherence.

2. **Semantic Map Injection (RoomState)**

   * The RoomState template sensor is read to obtain the normalized room-level map.
   * This is inserted as a dedicated section (e.g., `roomstate`) preserving per-room and per-domain slots.

3. **LiveState Snapshot**

   * Home Assistant’s state machine provides current entity states.
   * For high-cardinality data, the system either passes through the raw surface or extracts a focused subset relevant to the active Kata.

4. **Identity Resolution**

   * The Identity script resolves the active human (or humans) and assistant persona from person entities and cabinets.
   * GUIDs, ACLs, essence, and relationships are collected into a single identity block.
   * If a session token (TGT) exists, it is bound here.

5. **Tool Inventory Curation**

   * Based on identity, ACLs, and current task type, the system selects a bounded set of tools (up to 128).
   * Tools requiring secure mode are included only if a valid TGT is present.
   * High-risk tools may be omitted unless explicitly requested, even for the owner.

6. **Prompt Construction**

   * The prompt loader composes these into a single structure:

     ```json
     {
       "identity": { ... },
       "kata": { ... },
       "roomstate": { ... },
       "livestate": { ... },
       "tools": [ ... ]
     }
     ```

   * This object becomes the core Context Frame passed to Friday (or to a Monk, if the reasoning task is delegated).

The result is a deterministic, inspectable, and fully explainable cognitive substrate.

---

## 18.4 Cross-Layer Cohesion

Within the Context Frame:

* **Kata** expresses *what is important right now*.
* **RoomState** expresses *how the home is structured and labeled*.
* **LiveState** expresses *what is happening right now in the raw HA state*.
* **Identity Capsule** expresses *who is involved and what is allowed*.
* **Tool Inventory** expresses *what actions are actually available under those permissions*.

This layering simplifies reasoning:

* Friday can reference LiveState when she needs hard truth.
* She can use RoomState to understand where that truth lives in the physical world.
* She can appeal to the Kata when prioritizing attention or deciding what to surface.
* She can consult Identity and the Tool Inventory to determine whether, and how, to act.

---

## 18.5 Design Rationale

The Context Frame is engineered to satisfy several non-negotiable properties:

* **Determinism** – identical state produces identical Context Frames.
* **Traceability** – every field can be traced back to a HA sensor, cabinet drawer, or identity record.
* **Bounded Actuation** – the action surface is explicit, finite, and ACL-gated.
* **Extensibility** – new tools, components, or identities can be integrated by extending the cabinet manifests and ACL definitions, without changing the cognitive model.
* **Auditability** – forensics can reconstruct why a tool was available and how a decision was made.

The inclusion of a structured, bounded **Tool Inventory** within the Context Frame ensures that cognition and actuation are **co-designed**, not bolted together. Friday always reasons in a space where:

* the world model,
* the actors, and
* the possible actions

are all explicit, coherent, and securely bound.
