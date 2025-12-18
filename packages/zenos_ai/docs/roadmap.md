# 🏯 **ZenOS-AI Roadmap**

*A Local-First Cognitive Architecture for Persona-Scale AI*

ZenOS-AI is a structured, cabinet-centric AI framework designed for safe, deterministic, household-scale cognitive systems.
This roadmap describes the **1.0 Release Candidate**, the **1.0 GA goals**, and the **post-GA direction** of the ZenOS-AI architecture.

---

# **1.0 Release Candidate — Current Capabilities**

The RC establishes the foundational identity, reasoning, storage, and introspection layers of the ZenOS-AI system.
These components form the minimum stable substrate required for persona activation. At this point I'm working on locking in the featureset.
I am not currently working on deployment. As such some of this will be hard and klunky. But get it in there and it works. Part of RC1 will be exploring 
the good bad and ugly on getting the components out and finalizing core.

Currently I expect to deliver something similar to...

```
/packages/zenos-ai/
  ./zenos_library_cabinets.yaml  < Core default cabinets preconfigured for a default new system.
  ./zenos_core_zenos1.yaml       < All the default scripts, automations, and required support sensors as templates
    scripts:
      zen_dojotools_(foo)        < (foo) script
    automations:
      zen_volume_redirector
      zen_(foo)                  < (foo) automations
    template:
      sensor:                    < (foo) template sensor
  ./zenos_flynn.yaml             < Scripts and support sensors for 'flynn' onboarding and recovery.  Yes, THAT Flynn...
/custom_templates/
  ./zen_os_1(release).jinja      < Prompt Loader and core support macros
  ./zen_index.jinja              < Index core macros, import as 'zen'
  ./zen_(foo).jinja              < (foo) component support macros, import as 'zen' when possible to accommidate move to core.
```

---

## ✅ **Core Foundations**

### **Identity & Local Selfhood Authority (LSA)**

Stable, GUID-based identity infrastructure for users, AI personas, and system components.

### **Canonical Cabinet Schema (Ring-0)**

Deterministic cabinet definitions, cabinet loader, and health-validation tools ensuring proper system structure before any persona loads.

### **Complete Label Framework**

Unified labeling, normalization, and integrity checks used across indexing, cabinets, KFCs, and kata generation.  Deliver method to install core label set.

### **Proprioception v1 (Initial Structure Map)**

The system’s first-stage “body map,” defining essential components, expected cabinet presence, and structural awareness.

### **FileCabinet v3.9**

Typed cabinet storage with label-aware entity resolution and deterministic drawer operations.

### **Volume Redirector**

Unifies legacy variable flows and brings them under Ring-0 governance in a predictable, auditable way.

---

## 🧠 **Cognitive Substrate**

### **Zen HyperIndex v3.9 (Hypergraph Engine)**

A multi-label, multi-domain hypergraph engine for entity discovery, adjacency reasoning, and complex set-based operations.

**RC Note:**
HyperIndex currently exists as a separate parallel tool for evaluation and iteration. Don't build dependencies.

**GA Direction:**
HyperIndex is fully consolidated into **Zen DojoTools Index**, and external dependencies on the parallel RC tool should not be built.

---

## 📚 **Memory & Reflection**

### **Zen Kata Engine v2**

Long-term memory system providing structured, stable Katas, updated through event-driven Monk tasks.

### **Monastery v2**

Central reasoning and summarization loop that transforms system activity into persistent kata memory.

---

## 🔧 **Zen DojoTools (Standard Toolset)**

A consistent set of operational tools supporting identity, indexing, history access, system health, and general system introspection.
History components use the modern **Zen DojoTools History** interface.

---

## 🏗 **Initialization & Loaders**

### **RC Initialization Framework**

* Deterministic cabinet loader
* Slot verification
* Structural health validation
* Manual initialization routines

### **Kung Fu Component Loader (KFC Loader)**

Baseline KFCs are driven by labels, not environment specifics, ensuring compatibility across compliant deployments.

---

# **1.0 GA — Planned Release Target**

The GA release focuses on consolidation, completeness, persona lifecycle stability, and the onboarding experience.

---

## 🎛 **Foundational Enhancements**

### **Proprioception v1 Finalization**

Completion of structural coverage and component presence detection, now validated through **hypergraph-aware kata memory**.

During RC:

* Hypergraph fields are added to kata templates
* Summarizers are expanded to support hypergraph outputs

For GA:

* Proprioception becomes kata-validated and graph-informed
* Component reasoning shifts from static templates to hypergraph awareness

### **Cabinet Loader v1 GA**

Highly deterministic loading, slot enforcement, diagnostics, and cabinet resolution boundaries.

---

## 📘 **Memory, Governance & Stability**

### **Governance Model v1**

Declarative operational boundaries that define safe persona behavior, tool access, and lifecycle expectations. (Can I read that? / Do that? basics )

---

## 🧬 **Persona Lifecycle**

### **Persona Importer v1**

Enhanced persona loading sequence:

* Identity
* Proprioception
* Governance
* Kata
* Essence
* Prompt structure

---

## 🛠 **Flynn Onboarding & Failback Agent**

Welcome to the grid. Yes... THAT Flynn. 

Flynn is a deterministic system agent responsible for:

* First-run onboarding
* Cabinet and label validation
* Failback handling when core structures are incomplete
* Safety enforcement during persona initialization

He will be designed to load if you only have one cabinet that can load - the system cabinet. Basically, if you can successfully get 'system' mounted Flynn should be able to help with the rest.

Flynn is *not* 'exactly' a persona; Flynn is an infrastructure-level component standing in as 'the system'
ensuring core components are available before attempting to load an agent, ensuring ZenOS-AI boots correctly and safely.

The prompt runtime will be enhancced with onboarding, health, and security shunts to direct the frontline ask to 'Flynn' when trouble is detected.

Flynn will also guide the user through the onboarding experience. Implementaiton will be a combination of prompt flow adjustment and a special repair and obboarding tool locked to Flynn's use. (See Governance improvements above).

---

## 🍱 **Kung Fu Components (GA Baseline Set)**

GA guarantees a standardized set of KFCs essential for system operation, including:

* **Alert Manager** < Will be delivered in baseline set.  Taskmaster will be included in the personal assistant kit. Both are label driven and make good examples.
* Core operational flows
* Baseline support KFCs referenced throughout indexing, kata, and system introspection

All KFCs are fully label-driven for deployment independence.

---

# **v.next — Future Direction**

Following the 1.0 GA release, ZenOS-AI evolves toward:

* Proprioception v2 (dynamic extension mapping)
* Expanded KFC catalog
* Deeper kata integration and long-horizon memory improvements
* Advanced summarization patterns
* Additional governance and identity modules
* Enhanced shared templates for higher portability
* Broader support for persona coordination
* General architectural refinement and simplification

These items represent iterative evolution beyond 1.0 and may be introduced after GA.
