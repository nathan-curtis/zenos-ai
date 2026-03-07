# 📘 **ZenOS-AI Documentation Hub**

> **Version:** 1.2.0 | **Last Updated:** December 2025
> **Author:** Nathan Curtis | **License:** MIT
> *Part of the Friday’s Party / ZenOS-AI project*

---

Welcome to the **ZenOS-AI Documentation** — the full map of the architecture, tools, cognitive model, and operational philosophy behind *Friday’s Party*.

ZenOS-AI turns **Home Assistant** into a real agentic, persona-aware operating system. This documentation explains how the pieces fit together: the **Cabinet System** for identity and memory, the **Monastery** for reasoning, the **Summarizer Engine** for awareness, and the **HyperIndex** for graph-based attention and discovery.

If you're building an AI construct, designing a DojoTool, wiring a Summarizer pipeline, or just trying to understand how Friday thinks, this directory is your guide.

---

# 📚 **Included Documentation**

This directory contains **10 documentation suites**, each aligned with a major subsystem in ZenOS-AI.

---

## 🧠 **1. Architecture**

**Folder:** `docs/architecture/`

The full cognitive and systems architecture.
This is the textbook for ZenOS-AI.

Highlighted chapters:

* `00_toc.md` – Table of contents
* `01_the_monastery_core.md` – The reasoning engine
* `02_Architectural_Overview.md` – The high level cognitive stack
* `03_Cognitive_Architecture_Foundations.md`
* `04_Cognitive_Data_Flow.md` – How signals travel
* `05_Reasoning_and_Kata_Design.md`
* `06_Scheduler_and_The_Abbot.md` – Task routing
* `07_Summarizer_Pipelines.md` – Awareness flow
* `08_Kata_Cabinet.md`
* `09_Identity_Architecture.md` - Security Model
* `11_RoomState_and_Perception.md` – Sensory model
* `14_Abbot_Scheduler_And_Taskflow.md`
* `19_Resilience_and_Failure_Model.md`
* `20_Tool_Invocation_and_Security.md`

If you want to know how the mind works, start here.

---

## 🗃️ **2. Cabinets**

**Folder:** `docs/cabinets/`

Defines how ZenOS-AI stores identity, memory, context, and structured state.

Key files:

* `cabinet_spec.md` – The cabinet standard
* `hypergraph_model.md` – How cabinets form a recursive graph
* `zen_redirector_spec.md` – Volume Redirector v3
* `readme.md` – Overview of cabinet classes and mounts

Cabinets are the filesystem of the mind.

---

## 🧩 **3. Custom Templates**

**Folder:** `docs/custom_templates/`

Jinja templates that power prompt assembly, context building, and deterministic preprocessing.

Files:

* `zen_os1_jinja.md`
* `zen_query_jinja.md`

This suite defines how Friday constructs her thoughts.

---

## 🥋 **4. Kung Fu Components**

**Folder:** `docs/kung_fu/`

Each Kung Fu component is a discipline: a subsystem Friday loads at runtime.

Documents cover:

* Interfaces
* Required signals
* Safety guarantees
* Component lifecycle

This is Friday’s skill tree.

---

## 📚 **5. Zen Library**

**Folder:** `docs/library/`

Shared utilities and primitives for every DojoTool.

Includes:

* `readme.md` – Overview
* `index_system.md` – Recursive index system internals

The Library is the glue that holds all subsystems together.

---

## 🧪 **6. Research**

**Folder:** `docs/research/`

Background research and whitepapers.

* `whitepaper_cognitive_architecture.md` – Theory behind the Monastery, Summarizers, and Cabinets

Good for deep dives and formal reasoning.

---

## ⚙️ **7. Script Modules**

**Folder:** `docs/scripts/`

Documentation for every Zen DojoTool and script module.

Includes:

* `script.zen_dojotools_filecabinet_readme.md`
* `zen_dojotools.calendar_readme.md`
* `zen_dojotools_event_emitter_readme.md`
* `zen_dojotools_hyperindex_readme.md`
* `zen_dojotools_index_readme.md`
* `zen_dojotools_inspect_readme.md`
* `zen_dojotools_manifest_readme.md`
* `zen_dojotools_query_readme.md`
* `readme.md` – Overview

Scripts are the motor cortex. They turn reasoning into action.

---

## 🧩 **8. Zen HyperIndex**

**Folder:** `docs/zen_hyperindex/`

Documentation for the recursive hypergraph-driven index system.

* `zen_hyperindex_overview.md`

If Cabinets are the filesystem, HyperIndex is the search engine plus attention model.

---

## 🧠 **9. Zen Summarizer**

**Folder:** `docs/zen_summarizer/`

The Summarizer subsystem manages:

* Reflection
* Context evolution
* Awareness loops
* Narrative reconstruction
* Kata reduction

Files:

* `ninja_summarizer_spec.md`
* `readme.md`
* `_index.json`

This is Friday’s working memory engine.

---

## 🔐 **10. Identity & Security Model**

**Folder:** `docs/architecture/09_Identity_Architecture.md`

The Identity subsystem defines:

 * who a construct is allowed to be
 * what it may see
 * where its authority begins and ends
 
It formalizes:

 * GUIDs, identity hashes, provenance chains
 * Essence Capsules and persona metadata
 * People, Families, Households, Constructs
 * ACL roots, owner/partner authority
 * SecuritySafe, ContentSafe, and SquirrelSafe redaction
 * Cognitive boundaries and hypergraph gating
 * Session tokens, visas, and delegated capability (v1.5)

This is Friday’s trust spine — the system that decides which parts of the world are even visible before reasoning begins.

---

## 🗺️ **11. Roadmap**

**File:** `docs/roadmap.md`

Tracks development goals for:

* Cabinet v3 and redirects
* MCP channel
* Summarizer Engine 2
* HyperIndex extensions
* Identity v2
* Persona bootflow
* Recovery and rollback contracts

---

# 🧭 **Recommended Reading Order**

1. Architecture
2. Cabinet System
3. HyperIndex
4. Summarizer
5. Library
6. Scripts
7. Kung Fu Components
8. Roadmap

This flow mirrors the structure of Friday’s cognitive stack.

---

# 🧘 **Philosophy**

ZenOS-AI centers on a simple cycle:

> Observe → Reflect → Select → Act → Summarize

Every subsystem feeds this loop. Friday maintains a dynamic internal narrative about her state, her reasoning, and the home around her.

---

# 🛠 Contributing

Contributions are welcome for:

* Cabinet schemas
* Label taxonomies
* Summarizer examples
* HyperIndex patterns
* Cognitive diagrams

Community thread:
[https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/)

---

If you're building your own agent, welcome to the Monastery.
Light a candle. Start with the cabinets. Everything grows from there.
