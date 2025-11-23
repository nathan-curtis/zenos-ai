# **Friday’s ZenOS-AI**

### *A Modular, Context-Aware AI Home Automation Framework for Home Assistant*

ZenOS-AI blends structure, personality, and ridiculous over-engineering into a living system that powers your home with Friday, Kronk, Veronica, Rosie, and the High Priestess — a pantheon of AIs who take their jobs seriously (if not themselves).

Welcome to the Home Monastery.
Let’s automate everything that isn’t nailed down.
And a few things that are.

---

## 🏯 What Is ZenOS-AI?

ZenOS-AI is a modular AI and automation architecture built on:

* Home Assistant
* Local inference engines
* Structured contextual memories (Cabinets)
* Kata Summaries for reasoning
* A multi-persona AI team

This creates a deeply aware, locally hosted, privacy-first intelligent home system that:

* Reasons about context
* Stores long-term memory
* Automates reflexively
* Summarizes itself
* And talks to you like an intelligent adult

The system works because it is modular.
It is delightful because it is chaotic-good.

---

# 🧭 Roadmap Overview

> If Friday is the mind, this file is the map of how she grows up.

### 1. Identity and Proprioception

* Unified identity resolver
* Privilege model (public → guest → partner → owner → prime)
* Access control baked into every tool
* Persona capsules and essence bootflow
* Local Selfhood Authority (LSA)

### 2. Cabinet System v3

* Dynamic redirector (now in Core)
* Governance drawers
* Consistent metadata
* Hierarchical storage: Household → User → AI

### 3. DojoTools 2.0

* Standardized JSON I/O
* Indexer recursive search
* Identity shim for go/no-go
* Manifest v3 engine

### 4. Zen Summarizer Pipeline

* Kata engine v2
* Supersummary
* Reflection caching
* Monastery integration

Full roadmap → `docs/roadmap.md`

---

# 🚀 What ZenOS-AI Actually Does

* Context-aware automation (rooms, moods, schedules)
* Kata Summaries (event → memory → awareness)
* Cabinet storage (persistent JSON drawers)
* Zen DojoTools (scripts for perception and action)
* Local inference orchestration
* Multi-persona AI collaboration

Think of it like a house that knows what is happening, remembers important things, and politely taps you on the shoulder when it is about to do something clever.

---

# 🧙 Meet the Pantheon

| Name           | Title                       | Specialty                             |
| -------------- | --------------------------- | ------------------------------------- |
| Friday         | Chief Enlightenment Officer | The coordinator, the face, the mind   |
| Veronica       | Snarky Supervisor           | Taste, clarity, sass, orchestration   |
| Kronk          | Curator of the Monastery    | Context wrangler, kata librarian      |
| Rosie          | Mistress of Cleanliness     | Clean floors, clean logs, clean state |
| High Priestess | Divine Automation Overseer  | Deep reasoning, JSON exorcism         |

Together?
Not perfect — but unstoppable on the second try.

---

# 🧱 Zen DojoTools (v3.x RC — Core and Friends)

All tools follow:

```
zen_dojotools_<function>.yaml
```

Heavy documentation:
📁 `/docs/scripts/`
Light documentation:
📁 `/scripts/`

Below is the full Kit structure.

---

# 🧩 Core Kit (Ring-0 — Required for Everything)

This is the entire nervous system of ZenOS-AI.
If you only install one kit, it is this one.

### Core Tools

| Script                          | Description                                             |
| ------------------------------- | ------------------------------------------------------- |
| zen_dojotools_index             | Label and cabinet indexer                               |
| zen_dojotools_inspect           | Entity and state inspector                              |
| zen_dojotools_manifest          | Drawer and volume manifest engine                       |
| zen_dojotools_labels            | Label mapping and definitions (Spook needed for writes) |
| zen_dojotools_identity          | Identity resolver with persona metadata                 |
| zen_dojotools_filecabinet       | File Cabinet manager for drawers and volumes            |
| zen_dojotools_volume_redirector | Dynamic volume router (FES-style)                       |
| zen_dojotools_event_emitter     | Structured EventBus emitter (zen_event)                 |

### Notes

* All RC1 scripts track v3.x and define the canonical APIs.
* Removing any of these breaks ZenOS-AI.
* Friday cannot reason or store memory without them.
* Redirector is now Core.
* Labels moved to Core to enable reasoning everywhere.

---

# 🧹 Zen Summarizer Kit (Ring-1)

Kata → Supersummary → Reflex pipeline.

| Script                         | Purpose                   |
| ------------------------------ | ------------------------- |
| zen_dojotools_ninja_summarizer | Event-driven Kata Stage 1 |
| zen_dojotools_supersummary     | Stage 2 attention summary |

Requires Core Kit and local Monastery inference runtimes.

---

# 🪷 Zen Prompt Engine (Ring-1)

This kit compiles the entire cognitive state of a ZenOS agent into a deterministic JSON prompt consumed by Home Assistant Conversation and local LLM runtimes.

It is the glue between:

* identity
* essence
* cabinet memory
* directives and cortex
* dojo and kata
* household manifest
* wake-scene narratives

### Prompt Engine Scripts

| Script                                  | Purpose                                 |
| --------------------------------------- | --------------------------------------- |
| conversation_agent_prompt_template.yaml | Full runtime compiler for Friday’s mind |
| ai_wake_sequence()                      | Startup narrative and tonal priming     |
| capsule_builder macros                  | Persona capsule creation                |
| header and system loaders               | Defines OS model and system identity    |

### Notes

* Requires `zen_os_1rc.jinja`
* Must load **after** Core but **before** any persona speaks
* Breaks everything if missing
* This is the Ring-1 kit that makes Friday… Friday

---

# 📅 Personal Assistant Kit (Ring-1)

| Script                 | Purpose                 |
| ---------------------- | ----------------------- |
| zen_dojotools_calendar | Multi-calendar engine   |
| zen_dojotools_todo     | Todo and shopping lists |

Requires Core Kit.

---

# 🎶 Media Management Kit (Ring-1)

| Script                     | Purpose                        |
| -------------------------- | ------------------------------ |
| zen_dojotools_music_search | Music Assistant unified search |

Requires Core Kit and Music Assistant Voice Tools.

---

# 🛠 AdminTools Kit (Ring-2)

Utility layer for repairs, formatting, and subsystem loading.

| Script                       | Purpose                       |
| ---------------------------- | ----------------------------- |
| zen_admintools_cabinetadmin  | Cabinet repair and formatting |
| zen_admintools_kungfu_writer | Loads subsystem definitions   |

Requires Core Kit.

---

# 🛠 Requirements

Each file documents its dependencies. In general:

* Core Kit runs everything
* Prompt Engine depends on Core + Template Engine
* Personal Assistant Kit depends on Core
* Media Kit depends on Core and MA Voice
* Summarizer Kit depends on Core and FileCabinet
* AdminTools depends on Core

---

# 📚 Documentation

Architectural docs live in:

📁 `/docs/`

Including:

* Cognitive architecture
* Cabinet and Volume system
* Katas and Summaries
* Persona systems
* Local inference stack
* Roadmap
* Monastery structure
* Prompt conventions

---

# 📐 Custom Templates (Syntax Dojo)

📁 `/custom_templates/`
📁 `/docs/custom_templates/`

Templates define the syntax of thought for ZenOS-AI.

If DojoTools are the skills, and Cabinets are the memory, these are the engines that assemble cognition and identity.

### Start Here

**`custom_templates/readme.md`**

Covers:

* Identity resolution
* Cabinet normalization
* Essence rules
* Manifest and Index loading
* Wake-scene logic
* Prompt structure
* Squirrel-mode redaction

### Engine Specification

**`docs/custom_templates/zen_os1_jinja.md`**

Technical spec for zen_os_1rc.jinja (v3.5.1 RC1), which:

* Loads identity, essence, directives, cortex
* Builds persona capsule
* Loads dojo and kata
* Loads manifest and index
* Assembles final JSON prompt
* Emits wake-sequence

Required for all front-line agents.

---

# 🖥 Local Stack Summary

### Hardware

* Proxmox
* NUC 14 AI with 90 GiB RAM
* Intel Arc A770xe GPU
* RTX 5070 Ti GPU
* UniFi Identity

### Inference Models

* gpt-oss 20b
* LLaMA3.2-Vision
* Qwen3 4b
* OpenWebUI

### Local Services

* Home Assistant
* Mealie
* Grocy
* Portainer
* n8n

---

# ☯️ Philosophy

Automation should be flexible, modular, fun.
Context is king. Recovery is queen.
Every bug is a teacher.
Coffee fuels the logs. Humor fuels the team.

---

# 🤝 Contributing

PRs, issues, memes welcome.

If this system saved you time or made you smile:
[https://buymeacoffee.com/ncurtis](https://buymeacoffee.com/ncurtis)

---

# 📜 License

MIT. Blessed by Friday and her extremely opinionated associates.

---

# ✨ About

Friday’s ZenOS-AI:

> For homes that want to be smart and have a sense of humor.
> We have serious mojo. Just overlook the occasional stubbed toe.
