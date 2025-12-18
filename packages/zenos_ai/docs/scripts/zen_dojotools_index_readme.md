# 📘 Zen DojoTools Zen Index — v3.8.0 RC1
**File:** `zen_dojotools_zen_index_readme.md`  
**Type:** Technical Documentation  

---

## Overview

The **Zen Index** is ZenOS-AI’s unified entity, label, and drawer correlation engine.  
Think of it as Friday’s *search cortex*:  
a high-level, safety-gated query system that can blend:

- entity sets  
- labels  
- drawers  
- adjacency relationships  
- set operations  
- optional deep expansion via Zen Inspect  

The Zen Index lets LLM agents ask flexible, multi-input questions like:

- *“Which entities match label X AND label Y?”*  
- *“What drawers belong to these entities?”*  
- *“What labels cluster near this selection?”*  
- *“Expand these entities through their ObjectGraph neighbors.”*  

It does all of this through a hardened, from_json-everywhere pipeline that  
**guarantees safety, no template injections, and consistent structured outputs.**

This is the module Friday and Veronica use when they need fast, safe correlation
across the ZenOS-AI entity graph.

---

## Core Capabilities

### 🔍 Entity & Label Set Logic
Supports two user-input sets (`entities_1`, `entities_2`)  
plus two label inputs (`label_1`, `label_2`).

Label inputs are resolved through Home Assistant’s `label_entities()` helper.

Each pair is combined using a selectable set operator:

- **AND** — Intersection  
- **OR** — Union  
- **NOT** — Difference  
- **XOR** — Symmetric difference  

If no inputs are provided, it returns the **system label directory**.

### 📡 Index Command Mode (Expert Mode)
The field `index_command` overrides **all** other inputs and activates the
external Zen Indexer microservice.

Flow:

1. Emit `zen_indexer_request` event  
2. Wait for matching `zen_index_response`  
3. Capture response or timeout  
4. Fall back to local logic if needed

This supports high-intelligence, LLM-originated queries using a structured query DSL.

### 🔎 Optional Entity Expansion via Zen Inspect
When `expand_entities: true`, the Zen Index:

- Invokes `script.dojotools_zen_inspect`
- Gathers relationship-expanded entities  
- Returns them alongside the base entity set  

This is how Friday walks the “adjacent graph” around selected entities.

### 🏷 Label Aggregation & Adjacency
The Zen Index automatically computes which labels appear across:

- the resolved entities  
- the expanded entities  
- drawers discovered via the Zen Indexer  

Adjacency lists give Friday an understanding of **local label neighborhoods** —
useful for predictive associations, clustering, and similarity queries.

### 🗄 Drawer Resolution
If the Zen Indexer returns drawer matches (optional), the script:

- parses drawer JSON safely  
- extracts drawer-level labels  
- flattens and deduplicates them  
- exposes “drawer adjacency” as an output field

This forms the basis of **drawer-based graph traversal** for future modules.

### 🛡 Fully Safe Parsing & Concurrency
The Zen Index uses:

- `from_json` on all inbound strings  
- strict timeout handling  
- correlation IDs for all event-based calls  
- fallback paths when the Indexer is unavailable  

The result is deterministic, drift-free, and LLM-safe.

---

## Output Structure

The Zen Index always returns a unified JSON envelope:

```

{
"result": {
"simple": [...],              # The core entity set
"expanded": [...],            # Expanded via Zen Inspect (optional)
"adjacent_labels": [...],     # Labels near the result set
"index": [...],               # [entity_id, [labels...]]
"drawers": [...],             # Drawer matches (optional)
"drawer_adjacent_labels": [...] # Aggregated drawer labels
},
"operator": "AND|OR|NOT|XOR|*",
"inputs": {
"entities_1": [...],
"label_1": "string",
"entities_2": [...],
"label_2": "string",
"expand_entities": false
},
"error": null | "Timeout …"
}

```

The operator `' * '` is returned when no inputs are supplied
and the system falls back to the global label index.

---

## Input Modes & Behavior

### ▶️ Standard Mode (no index_command)
Uses `entities_1`, `entities_2`, `label_1`, `label_2`:

1. Resolve labels → entity lists  
2. Apply set operator  
3. Optionally expand  
4. Compute adjacency  
5. Compute drawer relations (if available)  
6. Render unified output  

### ▶️ Index Command Mode
Designed for high-level, LLM-driven queries.

- Pass any DSL query string  
- Zen Indexer handles the heavy logic  
- Zen Index sanitizes and returns structured results  

Timeouts are gracefully surfaced to the agent.

---

## Example Queries

### Basic: Entities with label “kitchen”
```

entities_1: []
label_1: kitchen
operator: AND

```

### Intersection of two label groups
```

label_1: battery
label_2: critical
operator: AND

```

### Symmetric difference between two entity sets
```

entities_1:

* sensor.a
* sensor.b
  entities_2:
* sensor.b
* sensor.c
  operator: XOR

```

### Expand entity graph
```

entities_1: group.master_suite
expand_entities: true

```

### Advanced: DSL-driven query
```

index_command: entities(label:motion) AND label:critical

```

---

## Why the Zen Index Exists

Agents in ZenOS-AI need a way to:

- correlate sensors  
- cluster entities  
- reason about labels  
- navigate drawers  
- traverse local context graphs  
- answer natural-language queries about “what belongs with what”

The Zen Index is the abstraction layer that takes raw HA state  
and turns it into **correlated, reasoned, structured data** —  
the very thing LLMs are good at interpreting.

Without the Zen Index:

- label_sets would be opaque  
- entity correlations wouldn’t exist  
- adjacent context discovery would be impossible  
- Zen Inspect would have no frontend  
- the Zen Indexer would have no runtime wrapper  
- agents would mix up labels, drawers, and entity sets  

This is the *search brainstem* of ZenOS-AI.

---

## Summary

The Zen Index RC1 provides:

- A fully featured, label/entity correlation engine  
- Integrated set logic with four operators  
- Zen Indexer DSL support  
- Optional Zen Inspect expansion  
- Label adjacency & drawer adjacency  
- Strict JSON safety  
- Hardened event-driven execution  
- Deterministic and LLM-stable outputs  

If the Manifest is the MRI  
and FileCabinet is the filesystem driver  
then the Zen Index is Friday’s **graph engine** —  
the way she understands where everything is and how it connects.
