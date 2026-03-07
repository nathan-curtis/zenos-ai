# **Cabinet Hypergraph Model — Best Practices & Usage Guide**

**Path:** `docs/cabinets/hypergraph_model.md`
**Part of the ZenOS-AI Cabinet Documentation Suite**

This document supplements the Cabinet System overview and explains **how ZenOS-AI models data using a hypergraph**, how drawers and labels create meaning, and how authors can build safe, scalable structures for AI reasoning inside Home Assistant.

---

# **1. Overview**

ZenOS-AI uses a **hypergraph-style storage model** to represent relationships between:

* cabinets
* drawers
* JSON documents
* sensors
* procedures (katas)
* automations and tools

In this system:

* **Nodes** = data containers (cabinets, drawers, JSON, sensors, katas)
* **Edges** = semantic relationships (labels, pointers, mounts, schema tags, index links)

This makes the system:

* fast
* expressive
* composable
* safe
* predictable for Friday, Kronk, the Monastery, and future AIs in your home

---

# **2. Core Concepts**

## **Nodes**

ZenOS-AI treats the following as first-class nodes:

* **Cabinets** — identity, preferences, metadata
* **Drawers** — namespaces that hold structured documents
* **Documents** — JSON blobs representing schemas
* **Entities** — sensors, booleans, stats, inputs
* **Katas** — procedural instructions
* **Automations** — external logic that writes to drawers

## **Edges**

Edges represent relationships between nodes:

* **Labels** — semantic grouping tags
* **Pointers / mounts** — a drawer referencing another drawer
* **Index relations** — label-based discovery via Zen Index
* **Live joins** — sensor state merged into drawer context at read-time

## **Metadata as Structure**

Drawers encode meaning through:

* owner
* ACL flags
* timestamp
* schema tags
* version
* urgency / criticality

## **Canonical vs Ephemeral**

**Canonical drawers**

* Household / Family / User cabinets
* Dojo subsystem definitions
* SYSTEM-level OS metadata

**Ephemeral drawers**

* scratchpad
* temporary session notes
* working memory
* todo queues

---

# **3. Why Hypergraph?**

### **Composability**

Small, focused drawers can be linked to form complex workflows without duplication.

### **Fast Discovery**

Label-based queries identify relevant nodes instantly across cabinets.

### **Safety**

Schemas, ACLs, and human-required confirmations make dangerous writes impossible without explicit opt-in.

### **Predictable AI Reasoning**

Friday, Kronk, and the Monastery all interpret structure the same way.

---

# **4. Best Practices**

## **(1) Naming & Labeling**

Use **paired labels** for clarity:

```
<domain>.<purpose>
```

Examples (public-safe):

* `climate.kata`
* `lighting.threshold`
* `inventory.recipe`

Always include:

* `owner:<role or id>`
* `version:vX.Y`

## **(2) Single-Responsibility Drawers**

Drawers should represent **one idea only**, such as:

* a threshold
* a pointer list
* a kata
* a subsystem status
* a single category of metadata

Avoid stuffing multiple subsystems into one drawer.

## **(3) Use Provided Templates**

Templates in `/docs/` (public-safe versions) include:

* `docs/kung_fu/sample_kata.yaml`
* `docs/kung_fu/kata_template.md`
* `docs/zen_summarizer/readme.md`

Predictable structure ensures Friday parses correctly.

## **(4) Pointer-First Composition**

Only store authoritative data once.
Reference everything else with:

```
pointers:
  - cabinets.<name>.drawers.<drawer_id>
```

## **(5) ACLs & Privacy Flags**

Mark sensitive drawers with ACL keys:

```
acl: restricted
visibility: squirrel
role: system-only
```

Friday respects them automatically.

## **(6) Draft → Review → Promote Workflow**

Never overwrite canonical data without:

1. Review
2. Confirmation
3. Version bump

Workflow path example:

```
draft -> scratch -> reviewed -> promoted -> canonical
```

## **(7) Routine Housekeeping**

Monthly:

* prune old drafts
* re-index labels
* clean up fragmentation

## **(8) Store Consumables in Separate Drawers**

Thresholds, filters, and replaceables should live in drawers like:

```
lighting.consumable_threshold
```

Referenced by subsystems, not embedded inside them.

## **(9) Provenance**

Include:

* timestamps
* version numbers
* author/automation info

And generate readable logs using Ninja Summaries.

---

# **5. Sample Structures (Public-Safe)**

These appear under:

```
docs/samples/
```

### **Sample Kata Drawer**

`docs/samples/sample_kata.yaml`

```yaml
title: Sample Procedure
summary: Canonical example of a ZenOS-AI procedural kata.
steps:
  - validate_sensors
  - check_interlocks
  - perform_action
preconditions:
  sensors: ["binary_sensor.example_state"]
owner: "example"
version: "1.0.0"
last_updated: "2025-01-01T00:00:00Z"
labels:
  - example.kata
  - subsystem.demo
```

### **Sample Threshold Drawer**

`docs/samples/sample_threshold.yaml`

```yaml
name: sample_threshold
target_value: 42
alert_label: example.alert
replacement_pointer: cabinets.sample.replacements.item_42
```

---

# **6. Example Workflows (Public-Safe)**

Found under:

```
docs/samples/workflows/
```

### **Lighting Workflow Example**

1. Check occupancy
2. Validate after-hours rule
3. Load brightness thresholds
4. Apply lighting profile

### **Media Workflow Example**

1. Find controllers with label `media.controller`
2. Check availability
3. Load queued playlist pointers
4. Write history event to `media.log` drawer

---

# **7. Query Patterns**

### **Label Query**

```
labels:example.kata AND labels:subsystem.demo
```

### **Pointer Expansion**

Pseudocode:

```
drawer.pointers[] -> resolve each into mounted drawer
```

Used by:

* Ninja Summarizer
* Supersummary
* Dojo Loader
* Monastery reasoning

---

# **8. Governance & Safety**

* Hardware-affecting actions must require **explicit human confirmation**.
* Canonical drawers must never be changed without a version bump.
* SYSTEM cabinet is **immutable** to frontline AIs.
* Schema mismatches must prevent writes.
* The hypergraph structure enables safe, zero-trust behavior by default.

---

# **9. Authoring Tips**

* Keep drawers small and legible
* Prefer linking over embedding
* Use templates
* Always label
* Always timestamp
* Version consistently

---

# **10. Where This Doc Lives**

Canonical GitHub location:

```
docs/cabinets/hypergraph_model.md
```

Internal (optional) in-cabinet location:

```
<household_cabinet>.documentation.cabinet_hypergraph
```

---

# **11. Related Documentation**

| Topic                            | GitHub Path                     |
| -------------------------------- | ------------------------------- |
| **Cabinet System Overview**      | `docs/cabinets/readme.md`       |
| **Kung Fu Components**           | `docs/kung_fu/readme.md`        |
| **Zen Summarizer (Stage 1 & 2)** | `docs/zen_summarizer/readme.md` |

This document is meant to complement — not replace — these.

---

# **12. External References**

**“Stuffing Elephants in Drawers — Representing Concepts Without Shoving the Whole Elephant Into Storage”**
Friday's Party post on abstraction in Home Assistant:
[https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/120?u=nathancu](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/120?u=nathancu)

A perfect metaphor for the hypergraph model.
