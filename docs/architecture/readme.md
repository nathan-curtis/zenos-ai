# **ZenOS-AI Architecture Series — Preamble & Orientation**

### *(Applies to Release 1.0 RC1; Identity subsystem may slip to 1.5; Tool-shunt security models targeted for v.next)*

---

## **Purpose of This Series**

This directory contains the formal architectural documentation for **ZenOS-AI**, the cognitive home-automation system powering **Friday**, **Veronica**, **Kronk**, and the broader household construct ecosystem.

Each file isolates a specific architectural domain, explains its rationale, presents the underlying computational model, and shows how each concept maps directly onto **Home Assistant state, Jinja, YAML, event-driven behavior, and structured memory**.

This series is designed to support:

1. **Researchers** seeking a reproducible cognitive architecture implemented on commodity hardware.
2. **Engineers** integrating agentic reasoning into event-based IoT systems.
3. **Developers and designers** of autonomous assistants requiring safety, introspection, and explicit decision boundaries.
4. **Advanced Home Assistant users** who want to understand how an agentic system can live inside an event-driven automation platform.

The goal is not simply to describe the system — but to make its reasoning and constraints fully traceable from first principles to live code.

---

## **Scope and Release Notes**

This series documents **ZenOS-AI Release 1.0 RC1**.

Two subsystems have explicit release caveats:

### **Identity System (ZenAI-ID)**

The identity framework is fully specified, including:

* GUIDs and provenance chains
* Identity Capsules and persona metadata
* Families, households, and relational mapping
* ACL roots, capabilities, and owner/partner authority
* Redaction layers (Squirrel Mode)
* Cognitive boundaries and access gating

Due to implementation complexity across large Cabinet graphs, **full identity enforcement may ship in v1.5**, but the architecture is documented here for completeness.

### **Secure Tool-Shunt Model**

Token-gated tool execution and shunted capability routing require deeper privilege separation and interface hardening.
These components will not ship in 1.0 and are targeted for **v.next**.

All other architectural systems — RoomState, the Monastery/Kata pipeline, Cabinet/Drawer memory, the Summarizer, Abbot scheduling, and the Interactive Friday load path — are stable and intended for the 1.0 release.

---

## **What This Series Covers**

This directory contains the complete conceptual and operational model for ZenOS-AI. Sections include:

---

### **Foundational Cognitive Model**

* Why ZenOS uses structured state instead of free-form memory
* Cognitive constraints, determinism, and safe fallback behavior
* Friday’s operational contract and reasoning envelope

---

### **Perception Pipeline (Sensorium)**

* RoomState as autonomic substrate
* Hypergraph representation of sensors and states
* Environmental admissibility filters and safety gates
* Real-time perception through event-driven updates

---

### **Memory Architecture**

* Cabinets as hierarchical memory containers
* Drawers as typed, versioned objects
* Volumes as atomic memory packets
* Summarizer and Kata compression pipeline
* How Friday integrates updated memory into consciousness

---

### **Reasoning Layer (The Monastery)**

* The Monastery as an externalized frontal cortex
* Kronk’s role as Curator
* Kata generation and reconciliation
* The Abbot as inference scheduler
* Consciousness updates and reasoning cycles

---

### **Identity & Security Model (Section 9)**

Identity is both a **security boundary** and a **cognitive boundary**, defining the edges of what a construct may know, do, or remember.

The architecture includes:

* **Identity Capsules** (GUIDs, identity hashes, provenance, persona metadata)
* **Humans, AI-users, families, and households** as first-class identity structures
* **ACL roots** and cabinet-level authority
* **Owner/Partner relationships** and delegated capability surfaces
* **Cognitive gating** and redaction (SekretSquirrel safe filters)
* **SecuritySafe, ContentSafe, and SquirrelSafe passes** applied before any expansion or reasoning
* **Guest Mode** and PG-13 defaults when identity is ambiguous
* **Construct boot requirements** (Prime AI, HoH, Cabinet integrity)
* **Version 1.5 extensions**: session tokens, visas, cross-household federation
* **Identity as part of consciousness** — preventing impersonation, leakage, or unauthorized drawer mutation

Identity determines **which parts of the hypergraph exist** for a principal. Redaction removes forbidden structure entirely, not just from the answer, but from the agent’s *world model* for that request.

---

### **Action Layer**

* Separation of control-plane and data-plane behaviors
* Safe action preconditions and environmental validation
* Deterministic tool invocation
* Future tool-shunt capability with token gating and persona authority

---

### **Trace Examples**

* State change → RoomState → perception → Monastery reasoning → action
* Cabinet scan + identity resolution
* Narrative instruction → summarizer → memory mutation
* Full reasoning cycle walkthroughs

---

### **Appendices**

* Canonical schemas
* Glossary
* Reusable Jinja/YAML patterns
* Cognitive architecture references

---

## **Philosophy of This Documentation**

This series assumes the reader may:

* inspect YAML and identify intent
* trace Jinja expressions
* follow event-driven IoT patterns
* understand cognitive architecture analogies
* want to rebuild or adapt the system using the same principles

Every section is written to be **reconstructable**, **traceable**, and **operational**.

The architecture is intentionally layered:
**Perception → Memory → Cognition → Identity → Action**.
Each layer you understand strengthens understanding of the next.

---

## **Reading Path**

The recommended order:

1. Foundations
2. Sensorium
3. Memory
4. Cognition
5. Identity (Section 9)
6. Action

But each chapter is self-contained and cross-referenced.

---

## **Contribution and Versioning**

Pull requests, additions, and corrections are welcome.
All code examples map directly to production ZenOS-AI and Home Assistant deployments.
Version markers appear in each module header.

---

## **Next File**

**Next chapter:** `01_foundations_overview.md`
*The Core Cognitive Model and Why ZenOS-AI Works the Way It Does*
