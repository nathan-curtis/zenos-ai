# Zen HyperIndex: Unified Selection, Filtering, and Composition Engine

The core structural reasoning substrate of ZenOS-AI.

The Zen HyperIndex (formal name: Zen DojoTools Index 4.x) transforms Home Assistant entities into structured hypergraphs used for contextual reasoning, planning, summarization, anomaly detection, and higher-order DojoTools.

Instead of treating entity selection as a flat list lookup, HyperIndex uses a three-stage pipeline:

1. SELECT — Library Index Join + Set Algebra


2. FILTER — ZQ-1 (ZenQuery Engine)


3. COMPOSE — Inspect + Metadata/Drawer Expansion



This architecture supports expressive targeting, deterministic outputs, and graph-aware reasoning within ZenOS-AI.


---

1. Stage One — SELECT

Library Index Join + Set Algebra (Graph-Surface Expansion)

The pipeline begins with broad, structured selection, intentionally performed before any filtering or metadata extraction.
This design allows HyperIndex to pull forward implicit relationships from the Library Index—relationships that do not yet exist as explicit graph edges until the final composition stage.

Because the hypergraph has not yet been constructed:

metadata relationships are unknown

drawer structures haven’t been read

adjacency and clusters do not yet exist

devices and entities have not been normalized


The selection stage compensates by enabling large, expressive set construction using:

AND

OR

NOT

XOR

nested expressions

drawer-based set expansion

label intersections

area intersections

multi-domain unions


This is sometimes referred to as the Index Join.

The purpose of constructing large initial sets is to give the system:

> a full conceptual surface from which the later hypergraph will emerge.



By assembling broad sets up front, the system can select based on intersections of conceptual edges—such as “all entities related to HVAC except sensors without metadata,” or “all motion-related devices on the security surface.”

The result of Stage One is the Selected Set, often intentionally large and rich with implicit structure.


---

2. Stage Two — FILTER

ZQ-1 (ZenQuery Engine): The WHERE Clause

Once Stage One has established the conceptual universe, the system applies ZQ-1, the schema-safe filtering layer.

ZQ-1 performs:

state filtering

numeric threshold filtering

regex matching

domain refinement

label refinement

area refinement

attribute validation

type inference

strict schema handling

guaranteed error-free execution


This step reduces the Selected Set to a Working Set, answering:

Which entities meet the numeric criteria?

Which entities are in the required state?

Which entities satisfy the label/domain constraints?

Which entities remain relevant based on context?


In SQL terms:

SELECT (Stage One)
WHERE  (Stage Two)

ZQ-1 guarantees deterministic, JSON-safe results regardless of malformed inputs.


---

3. Stage Three — COMPOSE

Inspect + Metadata Extraction + Drawer Traversal

After filtering, the Working Set flows into the Composition Layer, powered by zen_dojotools_inspect.yaml.

This stage extracts truth from Home Assistant and constructs the hypergraph.

Composition performs:

entity attribute extraction

device metadata extraction

registry lookups

domain and type mapping

area and label resolution

attribute normalization

deep drawer traversal

semantic relationship expansion

adjacency list generation

cluster formation

final graph assembly


This stage transforms raw entities into a structured Hypergraph containing:

nodes

edges

clusters

metadata

surface relationships


This format is consumed by:

summarizers

planners

anomaly detectors

DojoTools

room-state interpreters

kata generators

the Monastery reasoning layer


The output of Stage Three is the Composed Hypergraph, ready for downstream reasoning.


---

Pipeline Overview

┌──────────────────────────────────────────┐
          │                 STAGE 1                  │
          │                  SELECT                  │
          │  Library Index Join + Set Algebra        │
          │  (Graph-Surface Expansion)               │
          │  Large Conceptual Universe               │
          └───────────────────┬──────────────────────┘
                              ↓
          ┌──────────────────────────────────────────┐
          │                 STAGE 2                  │
          │                  FILTER                  │
          │           ZQ-1 (ZenQuery Engine)         │
          │        Working Set Refinement            │
          └───────────────────┬──────────────────────┘
                              ↓
          ┌──────────────────────────────────────────┐
          │                 STAGE 3                  │
          │                 COMPOSE                  │
          │   Inspect + Drawer Expansion + Graphing  │
          │        Final Hypergraph Output           │
          └──────────────────────────────────────────┘


---

Component Summary

Layer	File	Role

Selection Layer	Built into HyperIndex	Large-scale selection using Library Index + set algebra
Filtering Layer	custom_templates/zen_query.jinja	ZQ-1 safe filtering engine
Composition Layer	scripts/zen_dojotools_inspect.yaml	Metadata extraction and drawer-based composition
Pipeline Orchestrator	scripts/zen_dojotools_hyperIndex.yaml	Executes SELECT → FILTER → COMPOSE



---

Future Development Plan

Several enhancements are planned to increase expressive power and unify complex behaviors into simpler, declarative operations.

1. Complete the Recursion Loop

Implement full recursive evaluation through:

nested SELECT blocks

nested set operations

per-level filtering

deep drawer hierarchical expansion

recursion until no additional set growth occurs


This will allow deeply structured entity selection based on multi-layered logic.


---

2. Add Inline ZQ Filtering to the Index Block

Integrate ZQ-1 directly into the Index command structure, enabling:

index:
  and:
    - label: hvac
    - area: living_room
  zq:
    numeric_above: 72
compose: true

This integrates Stage One and Stage Two into a single declarative block.

Benefits:

fewer tool calls

more natural agent prompting

simplified automation flows

unified Select → Filter → Compose semantics



---

3. Unified Single-Command Execution Pattern

Enable fully standalone HyperIndex calls such as:

index:
  xor:
    - area: office
    - area: garage
  zq:
    state_equals: "on"
compose: true

This produces:

a complete Selected Set

refined Working Set

and the final Composed Hypergraph


…in a single operation.


---

4. Inline Composition Controls

Allow callers to tune the composition stage:

compose:
  adjacency: true
  metadata: true
  drawers: false

This offers flexible graph outputs depending on downstream requirements.


---

5. Scoped Recursion Depth

Add support for controlling recursion depth:

max_depth: 3

Useful for controlling expansion in very large installations.


---

Conclusion

The Zen HyperIndex is a multi-stage reasoning engine that begins by constructing large, expressive conceptual sets, refines them through schema-safe filtering, and composes them into a structured hypergraph.

This architecture gives ZenOS-AI the ability to:

reason across domains

understand structural relationships

generate contextual plans

detect anomalies

summarize intelligently

and operate as a true agent within Home Assistant


The future evolution of HyperIndex focuses on recursion, inline filtering, single-command composition, and configurability—pushing ZenOS-AI toward increasingly powerful and expressive graph-driven reasoning.
