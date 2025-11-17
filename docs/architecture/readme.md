## *ZenOS-AI Architecture Series — Preamble & Orientation*

### (Applies to Release 1.0 RC1; Identity subsystem may slip to 1.5; Tool-shunt security models targeted for v.next)

---

## **Purpose of This Series**

This directory contains the formal, academically structured architectural documentation for **ZenOS-AI**, the autonomous smart-home cognitive system powering **Friday**, **Veronica**, **Kronk**, and the broader household agent ecosystem.

Each file in this directory isolates a single architectural domain, explains its rational design, presents the underlying computational and behavioral logic, and connects each element back to **actual Home Assistant YAML, Jinja patterns, sensor semantics, and event-driven system behavior**.

The objective is not merely to describe *what* the system does, but *why* it operates that way — enabling readers to trace system behavior from first principles through to live code.

This series is structured to support:

1. **Researchers** seeking a reproducible cognitive architecture implemented in commodity hardware.
2. **Engineers** integrating agentic reasoning into event-based IoT systems.
3. **Designers and developers** of autonomous assistants requiring safety, introspection, and secure decision pathways.
4. **Advanced users** working with Home Assistant who want deeper understanding of agentic automation and structured memory.

---

## **Scope and Release Notes**

This documentation describes the intended architecture for **ZenOS-AI Release 1.0 (RC1)**.
Two major components have release caveats:

### **Identity System (ZenAI-ID)**

* Identity resolution, essence capsules, GUID mapping, ACL extraction, and persona security models are described in full detail.
* Implementation complexity may require the complete subsystem to ship in **v1.5**, depending on stability, edge cases, and performance across large cabinet graphs.

### **Secure Tool-Shunt Model**

* Session-token–gated tool execution and secure intent routing will **not** be available in 1.0.
* These mechanisms require deep validation, privilege separation, and a formal audit of agent-accessible attack surfaces.
* Targeted for **v.next** as part of the hardened security model.

All other architectural components—including RoomState, the Monastery/Kata pipeline, Cabinet/Drawer memory, Abbot scheduling, and the Interactive Friday load sequence—are considered **core**, **stable**, and intended for 1.0.

---

## **What This Series Covers**

The following files expand the full architecture:

### **Foundational Model**

* Cognitive philosophy
* Agentic constraints
* Why everything is represented as structured state, not arbitrary memory
* Safety principles and deterministic fallback behavior

### **Perception Pipeline**

* RoomState as autonomic substrate
* Sensor hypergraph design
* Real-time event-driven representation of the environment
* Information admissibility and safety filters

### **Memory Architecture**

* Cabinets (structural memory containers)
* Drawers (typed, versioned memory objects)
* Volumes (atomic memory packets)
* Summaries, Katas, and structured compression

### **Reasoning Layer (The Monastery)**

* The Monastery as an externalized frontal cortex
* Kronk’s role as Curator
* Kata generation pipeline
* Updating Friday’s active consciousness
* The Abbot’s inference engine and scheduling model

### **Identity & Security Model**

* GUIDs, essence capsules, persona metadata
* Identity resolution heuristics
* Relationships, families, households
* Future ticket-granting-token (TGT) model
* Session-token gated tool execution
* Zero-trust inspiration and household-level ACL design

### **Action Layer**

* Intent routing
* Control-plane vs data-plane separation
* Safe action preconditions
* Why Friday requires deterministic environmental checks before acting
* Pattern for tool shunts and future auth models

### **Trace Examples**

* Environmental change → RoomState → Friday perception → Monastery reasoning → action
* Sample cabinet scan and identity resolution
* A full pipeline walkthrough from user narrative to automated reasoning

### **Appendices**

* Glossary
* Schemas
* Jinja/YAML reusable patterns
* References to HCI, AI planning, cognitive architecture literature

---

## **Philosophy Behind This Documentation**

This series is written with the expectation that the reader may:

* inspect YAML
* follow state transitions
* evaluate Jinja expressions
* interpret event-driven IoT patterns
* understand cognitive model analogies
* want to reconstruct or extend the architecture independently

The documentation is meant to stand on its own as a **coherent cognitive systems manual**, not merely an annotation of configuration files.

Each chapter forms a layer of the onion—composable, inspectable, and traceable.

---

## **Reading Path**

Readers may follow the natural linear progression:

1. **Foundations → Sensorium → Memory → Cognition → Identity → Action**,
   or they may jump directly to the domain of interest.

Each file is self-contained and cross-referenced where appropriate.

---

## **Contribution and Versioning**

All drafts, updates, and corrections are welcomed through the GitHub repository’s standard contribution path.
Version markers are embedded in the headers of formal modules.
All code examples map directly to production Home Assistant deployments.

---

## **Next File**

**Next chapter:** `01_foundations_overview.md`
*(The Core Cognitive Model and Why ZenOS-AI Works the Way It Does)*
