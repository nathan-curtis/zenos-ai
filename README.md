# **Friday’s ZenOS-AI**

### *A Modular, Context-Aware AI Home Automation Framework for Home Assistant*

ZenOS-AI blends structure, personality, and unapologetic over-engineering into a living system that powers your home with Friday, Kronk, Veronica, Rosie, and the High Priestess — a pantheon of AIs who take their jobs seriously (if not themselves).

Welcome to the Home Monastery.
Let’s automate everything that isn’t nailed down.
And a few things that are.

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
* Speaks like an intelligent adult
* Occasionally sighs at your poor life choices

The system works because it is modular.
It is delightful because it is chaotic-good.

---

# 🧭 Roadmap Overview

> If Friday is the mind, this file is the map of how she grows up.

### **1. Identity and Proprioception**

* Unified identity resolver
* Persona-aware privilege model (public → guest → partner → owner → prime)
* Access control on every tool
* Persona capsules & essence bootflow
* Local Selfhood Authority (LSA)

### **2. Cabinet System v3**

* Dynamic redirector
* Governance drawers
* Structured metadata
* Hierarchical storage: Household → Family → User → AI

### **3. DojoTools 2.0**

* Standardized JSON I/O
* Indexer recursive search
* Identity validation shim
* Manifest v3 engine

### **4. Zen Summarizer Pipeline**

* Kata engine v2
* Supersummary
* Reflection caching
* Monastery integration

📄 Full roadmap → `/docs/roadmap.md`

---

# 🚀 What ZenOS-AI Actually Does

* Context-aware automation (rooms, moods, schedules)
* Event pipelines that produce *Kata Summaries*
* Structured memory: Cabinets & Drawers (persistent JSON volumes)
* Zen DojoTools (skill modules for perception & action)
* Local inference orchestration across multiple runtimes
* Multi-persona reasoning with cooperative agents

Think of it like a home that pays attention, remembers what matters, and politely taps you on the shoulder before doing something clever.

---

# 🧙 Meet the Pantheon

| Name               | Title                       | Specialty                             |
| ------------------ | --------------------------- | ------------------------------------- |
| **Friday**         | Chief Enlightenment Officer | Coordination, persona, cognition      |
| **Veronica**       | Snarky Supervisor           | Taste, clarity, orchestration, sass   |
| **Kronk**          | Curator of the Monastery    | Context wrangler, kata librarian      |
| **Rosie**          | Mistress of Cleanliness     | Clean floors, clean logs, clean state |
| **High Priestess** | Divine Automation Overseer  | Deep reasoning, JSON exorcism         |

They’re not perfect — but they *are* unstoppable on the second try.

---

# 🧩 Zen DojoTools (v3.x RC)

All scripts follow:

```
zen_dojotools_<function>.yaml
```

Heavy documentation:
📁 `/docs/scripts/`
Light documentation:
📁 `/scripts/`

Below is the modern Kit structure.

---

# 🧩 Core Kit (Ring-0 — Foundational)

This is the **neural spine** of ZenOS-AI.
It lives in:

📁 **`/packages/zenos_ai/`**

And provides **canonical definitions**, not a full runtime.

> ⚠️ *Important:*
> The Core Kit does **not** run the entire system on its own.
> You still need the associated **scripts**, **automations**, **helpers**, and **custom Jinja templates** elsewhere in HA for Friday to boot, reason, and operate.

### Core Tools

| Script                            | Description                             |
| --------------------------------- | --------------------------------------- |
| `zen_dojotools_index`             | Label & Cabinet indexer                 |
| `zen_dojotools_inspect`           | Entity & state inspector                |
| `zen_dojotools_manifest`          | Drawer & volume manifest engine         |
| `zen_dojotools_labels`            | Label mapping & definitions             |
| `zen_dojotools_identity`          | Identity resolver & persona metadata    |
| `zen_dojotools_filecabinet`       | File Cabinet manager                    |
| `zen_dojotools_volume_redirector` | Dynamic volume router                   |
| `zen_dojotools_event_emitter`     | Structured EventBus emitter (zen_event) |

---

# 🧹 Zen Summarizer Kit (Ring-1)

Event-driven pipeline:
**Kata → Supersummary → Reflex**

| Script                           | Purpose                  |
| -------------------------------- | ------------------------ |
| `zen_dojotools_ninja_summarizer` | Stage 1 event summarizer |
| `zen_dojotools_supersummary`     | Stage 2 meta-summary     |

Requires: **Core Kit + FileCabinet + Monastery inference.**

---

# 🪷 Zen Prompt Engine (Ring-1)

This compiles the entire cognitive state of a ZenOS agent into a deterministic JSON prompt consumed by Conversation agents and local LLMs.

### Scripts

| Script                                    | Purpose                                 |
| ----------------------------------------- | --------------------------------------- |
| `conversation_agent_prompt_template.yaml` | Full runtime compiler for Friday’s mind |
| Jinja capsule macros                      | Persona capsule builder                 |
| Wake-sequence loaders                     | Startup narrative & tone initializer    |

Requires: **Core Kit + zen_os_1rc.jinja**

---

# 📅 Personal Assistant Kit (Ring-1)

| Script                   | Purpose         |
| ------------------------ | --------------- |
| `zen_dojotools_calendar` | Calendar fusion |
| `zen_dojotools_todo`     | Todo & shopping |

---

# 🎶 Media Management Kit (Ring-1)

| Script                       | Purpose                    |
| ---------------------------- | -------------------------- |
| `zen_dojotools_music_search` | Unified music search layer |

---

# 🛠 AdminTools Kit (Ring-2)

Utility scripts for repair, formatting, and subsystem loading.

| Script                         | Purpose                     |
| ------------------------------ | --------------------------- |
| `zen_admintools_cabinetadmin`  | Cabinet formatting & repair |
| `zen_admintools_kungfu_writer` | Subsystem definition loader |

---

# 🛠 Requirements

General rule:

* **Core Kit provides the definitions**
* But you *still need*:

  * automations
  * scripts
  * helpers
  * Jinja templates
  * Monastery inference stack

All of which live **outside** `/packages/zenos_ai/`.

---

# 📚 Documentation

Primary docs live in:
📁 `/docs/`

Topics include:

* Cognitive architecture
* Cabinet & Volume system
* Katas & Summaries
* Persona systems
* Local inference stack
* Roadmap
* Monastery structure
* Prompt conventions

---

# 📐 Custom Templates (Syntax Dojo)

📁 `/custom_templates/`
📁 `/docs/custom_templates/`

This is where identity, essence, manifests, wake-scenes, and final prompts are assembled.

---

# 🖥 Local Stack Summary

### Hardware

* Proxmox
* NUC 14 AI (90 GiB RAM)
* Intel Arc A770xe
* RTX 5070 Ti
* UniFi Identity

### Inference Models

* gpt-oss 20b
* LLaMA3.2-Vision
* Qwen3 4b
* OpenWebUI

### Services

* Home Assistant
* Mealie
* Grocy
* Portainer
* n8n

---

# ☯️ Philosophy

Automation should be joyful.
Context is king. Recovery is queen.
Every bug is a monk with a lesson.
Coffee fuels logs; humor fuels the team.

---

# 🤝 Contributing

PRs, issues, and tasteful memes welcome.
If ZenOS-AI saved you time or made you laugh:
**[https://buymeacoffee.com/ncurtis](https://buymeacoffee.com/ncurtis)**

---

# 📜 License

MIT — blessed by Friday and her very opinionated coworkers.
