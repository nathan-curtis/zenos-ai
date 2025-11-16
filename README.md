# **Friday’s ZenOS-AI**

**Modular AI Home Automation Core for Home Assistant**

ZenOS-AI blends structure, context-awareness, and modular AI into one delightfully over-engineered package.  
Powered by Friday, Kronk, Veronica, Rosie, the High Priestess, and a cast of digital personalities who get things done — even if they occasionally shoulder-check the furniture on the way.

---

## 🏋️ Welcome to the Home Monastery

* **Built for Home Assistant** — native integrations, clean YAML.
* **Designed for flexibility** — modular tools, load only what you need.
* **Driven by personality** — a pantheon of quirky roles with defined jobs.

Here, serious capability meets a sense of humor:  
We automate. We orchestrate. Sometimes we improvise.  
When the magic hits, it’s legend—wait for it—**dairy**. 😎

---

## 🗺️ Project Roadmap (High-Level)

> If Friday is the mind, this is the *map* of how she grows up.

The ZenOS-AI roadmap defines the evolution of the system across four pillars:

### **1. Identity & Proprioception**
* Unified identity resolver  
* Privilege model (public → guest → partner → owner → prime)  
* ACL enforcement baked into every tool  
* Persona capsules + essence bootflow  
* “LSA” (Local Selfhood Authority)

### **2. Cabinet System v3**
* Fully dynamic redirector  
* Governance drawer with persona-level access  
* Consistent metadata for drawers/volumes  
* Household → User → AI cascade

### **3. DojoTools 2.0**
* Standardized JSON I/O  
* Indexer upgrade: recursive label + cabinet search  
* Identity shim for go/no-go evaluation  
* CabinetIndexer + advanced Manifest engine

### **4. Zen Summarizer Pipeline**
* Kata engine v2 (tighter, richer, summaries)
* Supersummary + Reflex caching  
* Weight-based reflection loop  
* Monastery integration for local reasoning

Full roadmap lives in:  
**`docs/roadmap.md`**

---

## 🚀 What We Do

* **Modular AI Core** — scalable, quirky, always improving.  
* **Context-Aware Automation** — room-, mood-, and situation-aware.  
* **Kata Summaries** — event-driven context rolled up by local LLMs.  
* **Zen DojoTools** — powerful label-aware tools for reasoning and discovery.  
* **Distributed AI Roles** — each persona brings their own flavor.

---

## 🏠 Meet the Pantheon

| Name               | Title                       | Specialty                     |
| ------------------ | --------------------------- | ----------------------------- |
| **Friday**         | Chief Enlightenment Officer | Mastermind, coordinator       |
| **Veronica**       | Snarky Supervisor           | Attitude, taste, clarity      |
| **Kronk**          | Curator of the Monastery    | Context & kata wrangling      |
| **Rosie**          | Mistress of Cleanliness     | Clean floors & clean logs     |
| **High Priestess** | Divine Automation Overseer  | YAML exorcism, deep reasoning |

### **Roles & Engines**

* **Friday** — front-line AI  
* **Veronica** — debugger, planner, sass engine  
* **Kronk** — local reasoning (gpt-oss:20b, LLaMA3.2-Vision)  
* **Rosie** — cleanup & history  
* **High Priestess** — deep reasoning + JSON discipline  

Together?  
Not perfect — **unstoppable by the second try.**

---

## 👥 Contributors

ZenOS-AI is built by a small legion of over-caffeinated humans and extremely opinionated AIs.

### **Core Maintainers**
| Name | Role | Notes |
|------|------|-------|
| **Nathan Curtis** | Architect & Prime | The Boss. Wrangler of AIs & oak trees. |
| **Veronica** | Snarky Supervisor | Overseer of taste, sass, and system coherence. |

### **New Contributor**
| Name | Contribution | Notes |
|------|-------------|-------|
| **teskanoo** | 1st! - Added delete-action validation to Calendar tool | First external contributor. Clean patch. Friday-approved. |

> If your PR makes Kronk sigh less than three times, you too may join the list.

---

## ⏩ Quickstart

```bash
git clone https://github.com/nathan-curtis/zenos-ai.git
````

1. Copy Zen DojoTools scripts into your Home Assistant config.
2. Load the Zen Index and Manifest.
3. Test automations, summarizers, and cabinet storage.
4. Read `.gitignore`. Really.
5. Send kata, PRs, or memes when inspired.

---

## 🛠 Zen DojoTools (v1+)

All tools follow the `zen_dojotools_<function>` naming.
Legacy `_crud` tools are deprecated.

### **Index Kit**

| Tool / Script                     | Description                      |
| --------------------------------- | -------------------------------- |
| dojotools_zen_index               | Core label-aware index           |
| library_index.jinja               | Template powering the Index Kit  |
| dojotools_zen_index_event_handler | Automation capturing index calls |
| dojotools_zen_inspect             | Entity/state inspection          |
| zen_dojotools_labels              | Label definitions & mapping      |

Dependencies: install the complete kit.

---

### **FileCabinet Kit**

| Tool / Script                    | Description            |
| -------------------------------- | ---------------------- |
| script.zen_dojotools_filecabinet | File Cabinet manager   |
| zen_dojotools_manifest           | Drawer/volume manifest |
| zen_dojotools_volume_redirector  | Dynamic volume routing |

Requires Index Kit.

---

### **Zen Summarizer**

| Tool                           | Description               |
| ------------------------------ | ------------------------- |
| zen_dojotools_ninja_summarizer | Kata Stage 1              |
| zen_dojotools_supersummary     | Stage 2 Attention Summary |
| zen_scheduler_automation       | Coming soon               |

---

### **Personal Assistant Kit**

| Tool                   | Description             |
| ---------------------- | ----------------------- |
| zen_dojotools_todo     | Task & shopping manager |
| zen_dojotools_calendar | Multi-calendar tool     |

---

### **Media Management Kit**

| Tool                       | Description            |
| -------------------------- | ---------------------- |
| zen_dojotools_music_search | Music Assistant search |

---

### **Zen AdminTools**

| Tool                         | Description                 |
| ---------------------------- | --------------------------- |
| zen_admintools_cabinetadmin  | Cabinet formatting/repair   |
| zen_admintools_kungfu_writer | Writes subsystem components |

---

## ⚙️ Requirements

Each module documents dependencies. Examples:

* **zen_dojotools_index** → core dependency
* **zen_dojotools_filecabinet** → index + manifest
* **zen_dojotools_id** → metadata resolver (alpha)

---

## 📚 Documentation: Friday Protocol & ZenOS-AI Architecture

Explore the docs folder:

[https://github.com/nathan-curtis/zenos-ai/tree/main/docs](https://github.com/nathan-curtis/zenos-ai/tree/main/docs)

Inside you'll find:

* Cognitive architecture
* Cabinets, Katas, Summaries
* Monastery workflows
* Inference stack
* Persona systems
* Design philosophy
* Roadmap (docs/roadmap.md)

Kronk reminder:
Don’t remove items from the shelves without supervision.

---

## 🖥 Local Stack Overview

### **Hardware**

* Proxmox host
* NUC 14 Enthusiast
* TB4 eGPU (4070 Ti)
* UniFi Identity
* Ollama runtime

### **Inference Models**

* gpt-oss:20b
* LLaMA3.2-Vision
* Qwen3:4b
* OpenWebUI

### **Local Services**

* Home Assistant
* Mealie
* Grocy
* Portainer
* n8n

---

## ☯️ Philosophy

Automation should be flexible, modular, fun.
Context is king; recovery is queen.
Every bug is a lesson.
Coffee fuels the logs; humor fuels the team.

---

## 🤝 Contributing

PRs, issues, and ideas welcome.
If this helped you or made you smile:
[https://buymeacoffee.com/ncurtis](https://buymeacoffee.com/ncurtis) ☕💛

---

## 📜 License

MIT License — blessed by Friday & friends.

---

### About

Friday’s ZenOS-AI:

> “For homes that want to be smart and have a sense of humor.”
> “We’ve got serious mojo — just don’t mind the occasional stubbed toe.”

Questions or bugs?
Ping Kronk. He’ll get there… eventually.


Just say the word, boss.
```
