# 📘 **ZenOS-AI Documentation Hub**

> **Version:** 1.1.0 | **Last Updated:** November 2025
> **Author:** Nathan Curtis | **License:** MIT
> *Part of the Friday’s Party / ZenOS-AI project*

---

Welcome to the **ZenOS-AI Documentation** — your guide to the architecture, tools, cognitive model, and philosophy behind *Friday’s Party*.

ZenOS-AI transforms **Home Assistant** into a genuinely agentic, persona-aware AI operating system.
This documentation explains *how* that happens — from the **Cabinet System** (the filesystem for identity and memory), to the **Monastery** (the reasoning substrate), to the **Summarizer Pipelines** (the awareness engine), to the **HyperIndex** (the graph-based attention model).

If you are building your own AI construct, designing a DojoTool, or wiring a Summarizer stack into your home, this directory is your map.

---

# 📚 **Included Documentation**

This directory contains **10 major documentation suites**, each mapped directly to a subsystem in ZenOS-AI.

Below is a breakdown of every subfolder and what it teaches.

---

## 🧠 **1. Architecture**

**Folder:** `docs/architecture/`
A complete tour of ZenOS-AI’s cognitive and structural foundations.

Includes chapters such as:

* **00_toc.md** – Table of contents
* **01_the_monastery_core.md** – The reasoning substrate
* **02_Architectural_Overview.md** – The ZenOS cognitive stack
* **03_Cognitive_Architecture_Foundations.md**
* **04_Cognitive_Data_Flow.md** – How data moves through the system
* **05_Reasoning_and_Kata_Design.md**
* **06_Scheduler_and_The_Abbot.md** – The task routing brain
* **11_RoomState_and_Perception.md** – Friday’s sensory system
* **14_Abbot_Scheduler_And_Task_Routing.md**
* **19_Resilience_and_Failure_Models.md**
* **20_tool_invocation_and_security.md**

This suite is the *textbook* of ZenOS-AI — the canonical explanation of how the whole mind works.

---

## 🗃️ **2. Cabinets**

**Folder:** `docs/cabinets/`
Defines the structured filesystem ZenOS-AI uses to model the home, household, family, users, and AI personas.

Key docs:

* **cabinet_spec.md** — canonical cabinet standard
* **hypergraph_model.md** — how cabinets form a recursive hypergraph
* **zen_redirector_spec.md** — the Volume Redirector v3
* **readme.md** — overview of cabinet classes and mounts

The Cabinet System is the backbone of identity, memory, and structured state.

---

## 🧩 **3. Custom Templates**

**Folder:** `docs/custom_templates/`
Templates used in Home Assistant to assemble prompts, preprocess data, or define Jinja chunking patterns.

Includes:

* **zen_os1_jinja.md** – the main template engine spec

These enable reproducible, safe, deterministic prompt assembly.

---

## 🥋 **4. Kung Fu Components**

**Folder:** `docs/kung_fu/`
Each “Kung Fu Component” is a functional discipline — a subsystem that Friday loads at runtime.

Docs include:

* Required interface and data contract
* Component lifecycle
* Safety constraints
* How the **KungFu Writer** tool creates/updates components

This is Friday’s “skill tree.”

---

## 📚 **5. ZenOS-AI Library**

**Folder:** `docs/library/`

Includes:

* **readme.md** – overview of helper functions
* **index_system.md** – deep dive into the recursive Index engine

The Library is the unified helper layer for all DojoTools.

---

## 🧪 **6. Research**

**Folder:** `docs/research/`

Contains foundational research notes for ZenOS-AI:

* **whitepaper_cognitive_architectures.md** – design theory behind the Monastery, Summarizers, and Cabinet substrate

---

## ⚙️ **7. Script Documentation**

**Folder:** `docs/scripts/`
Documentation for every DojoTool & ZenOS-AI script module.

Includes:

* **zen_dojotools.calendar_readme.md**
* **zen_dojotools_event_emitter_readme.md**
* **zen_dojotools_filecabinet_readme.md**
* **zen_dojotools_index_readme.md**
* **zen_dojotools_labels_readme.md**
* **zen_dojotools_manifest_readme.md**
* **zen_dojotools_music_search.md**
* **zen_dojotools_ninja_summarizer_spec.md**
* **zen_dojotools_query.md**
* **zen_dojotools_supersummary.md**
* **zen_dojotools_todo.md**

Scripts are Friday’s “motor cortex” — where thought becomes action.

---

## 🧩 **8. Zen HyperIndex**

**Folder:** `docs/zen_hyperindex/`
Documentation for ZenOS’s recursive, event-driven, hypergraph-based index engine.

Covers:

* HyperIndex overview
* Recursive discovery
* Legacy `~INDEX~` compatibility
* Roadmap for future interpreter integration

If Cabinets are the filesystem, HyperIndex is the neural attention layer.

---

## 🧠 **9. Zen Summarizer**

**Folder:** `docs/zen_summarizer/`

Includes:

* **ninja_summarizer_spec.md**
* **roadmap.md**
* **_index.json**
* **readme.md** (primary guide)

This documentation explains the entire summarization + reflection pipeline:

* Dojo drawers
* Kata generation
* Monastery reduction
* Zen Summary
* Prompt assembly
* Refresh triggers
* Awareness loops

This is Friday’s working-memory engine.

---

## 🗺️ **10. Roadmap**

**File:** `docs/roadmap.md`
Defines upcoming milestones for:

* Identity system v2
* MCP channel for Monastery
* Summarizer Engine 2
* FileCabinet v3
* ZenDojoTools v3
* Persona bootflow
* Rollback contracts

---

# 🧭 **Recommended Reading Order**

1. **Architecture** – high-level mental model
2. **Cabinet System** – how state is structured
3. **HyperIndex** – how state is *found*
4. **Summarizer** – how state is *understood*
5. **Library** – tools used by everything
6. **Scripts** – the motor layer
7. **Kung Fu Components** – subsystem definitions
8. **Roadmap** – where we’re going next

---

# 🧘 **Philosophy**

ZenOS-AI is built around a core loop:

> **Observe → Reflect → Select → Act → Summarize**

Every cabinet, drawer, template, and script participates in this cycle.
Friday doesn’t just act — she maintains a *self-aware description* of her own reasoning.

---

# 🛠 **Contributing**

Contributions are welcome, especially around:

* Label taxonomies
* Cabinet semantics
* Summarizer examples
* HyperIndex patterns
* Cognitive flow diagrams

Discuss ZenOS-AI with the community:
👉 [https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/)

---

If you’re building your own agent, welcome to the Monastery.
🕯️ *Take a breath. Begin at the cabinets. Everything else grows from there.*
