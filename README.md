# **Friday’s ZenOS-AI**

### *A Modular, Context-Aware AI Home Automation Framework for Home Assistant*

ZenOS-AI blends structure, personality, and ridiculous over-engineering into a living system that powers your home with Friday, Kronk, Veronica, Rosie, and the High Priestess — a pantheon of AIs who take their jobs seriously (if not themselves).

Welcome to the Home Monastery!
Let’s automate everything that isn’t nailed down.
…and a few things that are.

---

## 🏯 What Is ZenOS-AI?

ZenOS-AI is a modular AI and automation architecture built on:

* **Home Assistant**
* **Local inference engines**
* **Structured contextual memories (“Cabinets”)**
* **Kata Summaries for reasoning**
* **A multi-persona AI team**

This creates a deeply aware, locally hosted, privacy-first intelligent home system that:

* Reasons about context
* Stores long-term memory
* Automates reflexively
* Summarizes itself
* And talks to you like an intelligent adult

The system works because it’s modular.
It’s delightful because it’s chaotic-good.

---

# 🧭 Roadmap Overview

> If Friday is the mind, this file is the map of how she grows up.

### **1. Identity & Proprioception**

* Unified identity resolver
* Privilege model (public → guest → partner → owner → prime)
* Access control baked into every tool
* Persona capsules + essence bootflow
* “LSA” (Local Selfhood Authority)

### **2. Cabinet System v3**

* Dynamic redirector (now in Core)
* Governance drawers
* Consistent metadata
* Hierarchical storage: Household → User → AI

### **3. DojoTools 2.0**

* Standardized JSON I/O
* Indexer recursive search
* Identity shim for go/no-go
* Manifest v3 engine

### **4. Zen Summarizer Pipeline**

* Kata engine v2
* Supersummary
* Reflection caching
* Monastery integration

Full roadmap → `docs/roadmap.md`

---

# 🚀 What ZenOS-AI Actually Does

* **Context-aware automation** (rooms, moods, schedules)
* **Kata Summaries** (event → memory → awareness)
* **Cabinet storage** (persistent JSON drawers)
* **Zen DojoTools** (scripts for perception + action)
* **Local inference orchestration**
* **Multi-persona AI collaboration**

Think of it like a house that knows what’s happening, remembers important things, and politely taps you on the shoulder when it’s about to do something clever.

---

# 🧙 Meet the Pantheon

| Name               | Title                       | Specialty                             |
| ------------------ | --------------------------- | ------------------------------------- |
| **Friday**         | Chief Enlightenment Officer | The coordinator, the face, the mind   |
| **Veronica**       | Snarky Supervisor           | Taste, clarity, sass, orchestration   |
| **Kronk**          | Curator of the Monastery    | Context wrangler, kata librarian      |
| **Rosie**          | Mistress of Cleanliness     | Clean floors, clean logs, clean state |
| **High Priestess** | Divine Automation Overseer  | Deep reasoning, JSON exorcism         |

Together?
Not perfect — but unstoppable on the second try.

---

# 🧱 Zen DojoTools (v3.x RC — Core & Friends)

All tools follow:

```
zen_dojotools_<function>.yaml
```

Heavy documentation lives in:
📁 `/docs/scripts/`
Light documentation in:
📁 `/scripts/`

Below is the full Kit structure.

---

# 🧩 **Core Kit (Ring-0 — Required for EVERYTHING)**

This is the entire nervous system of ZenOS-AI.
If you only install one kit… it’s this one.

### **Core Tools**

| Script                              | Description                                         |
| ----------------------------------- | --------------------------------------------------- |
| **zen_dojotools_index**             | Label + cabinet indexer                             |
| **zen_dojotools_inspect**           | Entity/state inspector                              |
| **zen_dojotools_manifest**          | Drawer/volume manifest engine                       |
| **zen_dojotools_labels**            | Label mapping/definitions (Spook needed for writes) |
| **zen_dojotools_identity**          | Identity resolver (LSA) + persona metadata          |
| **zen_dojotools_filecabinet**       | File Cabinet manager (drawers/volumes)              |
| **zen_dojotools_volume_redirector** | Dynamic volume router (FES-style)                   |
| **zen_dojotools_event_emitter**     | Structured EventBus emitter (`zen_event`)           |

### **Notes**

* All RC1 scripts track **v3.x** and define the new canonical APIs.
* Removing any of these breaks ZenOS-AI.
* Friday cannot reason or store memory without them.
* Redirector is now Core (Cabinet v3 requirement).
* Labels moved to Core so everything can reason.

---

# 🧹 **Zen Summarizer Kit (Ring-1)**

Kata → Supersummary → Reflex pipeline.

| Script                             | Purpose                   |
| ---------------------------------- | ------------------------- |
| **zen_dojotools_ninja_summarizer** | Event-driven Kata Stage 1 |
| **zen_dojotools_supersummary**     | Stage 2 attention summary |

**Requires:**

* Core Kit
* Local Monastery inference runtimes

---

# 📅 **Personal Assistant Kit (Ring-1)**

| Script                     | Purpose               |
| -------------------------- | --------------------- |
| **zen_dojotools_calendar** | Multi-calendar engine |
| **zen_dojotools_todo**     | Todo & shopping lists |

**Requires:** Core Kit

---

# 🎶 **Media Management Kit (Ring-1)**

| Script                         | Purpose                        |
| ------------------------------ | ------------------------------ |
| **zen_dojotools_music_search** | Music Assistant unified search |

**Requires:**

* Core Kit
* Music Assistant Voice Tools

---

# 🛠 **AdminTools Kit (Ring-2)**

Utility layer for repairs, formatting, and subsystem loading.

| Script                           | Purpose                               |
| -------------------------------- | ------------------------------------- |
| **zen_admintools_cabinetadmin**  | Cabinet repair & formatting           |
| **zen_admintools_kungfu_writer** | Loads subsystem (kung fu) definitions |

**Requires:** Core Kit

---

# 🛠 Requirements

Each file documents its dependencies, but in general:

* **Core Kit** → everything depends on it + Spook! if you want label editing.
* **Personal Assistant Kit** → depends on Core
* **Media Kit** → depends on Core + MA Voice
* **Summarizer Kit** → depends on Core + FileCabinet (now Core)
* **AdminTools** → depends on Core

---

# 📚 Documentation

You'll find architectural docs in:

📁 **`/docs/`**

Including:

* Cognitive architecture
* Cabinet & Volume system
* Katas & Summaries
* Persona systems
* Local inference stack
* Roadmap
* Monastery structure
* Prompt conventions

**Kronk's reminder:**
*Please stop removing things from the shelves without supervision.*

---

# 🖥 Local Stack Summary

### Hardware

* Proxmox
* NUC 14 AI 90GiB RAM
* GPU Intel Arc A770xe, 16 GiB + 48 GiB shared from host = 64GiB vRAM, VM: Taran, Long Context
* TB4 eGPU RTX 5070 Ti, 16 GiB vRAM, VM: Iona, Fast Response
* UniFi Identity

### Inference Models

* gpt-oss:20b (Iona)
* LLaMA3.2-Vision (Iona, Taran)
* Qwen3:4b (Taran)
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
Context is king; recovery is queen.
Every bug is a teacher.
Coffee fuels the logs; humor fuels the team.

---

# 🤝 Contributing

PRs, issues, memes welcome.

If this system saved you time or made you smile:
[https://buymeacoffee.com/ncurtis](https://buymeacoffee.com/ncurtis) ☕💛

---

# 📜 License

MIT — blessed by Friday & her extremely opinionated associates.

---

# ✨ About

*Friday’s ZenOS-AI:*

> “For homes that want to be smart and have a sense of humor.”
> “We’ve got serious mojo — just don’t mind the occasional stubbed toe.”
