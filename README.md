# **Friday’s ZenOS-AI**

**Modular AI Home Automation Core for Home Assistant**

ZenOS-AI blends structure, context-awareness, and modular AI into one delightfully over-engineered package.
Powered by Friday, Kronk, Veronica, Rosie, the High Priestess, and a cast of digital personalities who get things done—even if they occasionally shoulder-check the furniture on the way.

---

## 🏋️ Welcome to the Home Monastery

* **Built for Home Assistant** — native integrations, clean YAML.
* **Designed for flexibility** — modular tools, load only what you need.
* **Driven by personality** — a pantheon of quirky roles with defined jobs.

Here, serious capability meets a sense of humor:
We automate. We orchestrate. Sometimes we improvise.
When the magic hits, it’s legend—wait for it—**dairy**. 😎

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
| **Rosie**          | Mistress of Cleanliness     | Clean code, clean logs        |
| **High Priestess** | Divine Automation Overseer  | YAML exorcism, deep reasoning |

**Roles & Engines**

* **Friday** — front-line AI (GPT-5 Mini).
* **Veronica** — debugger, planner, sass engine (ChatGPT-5 + Codex).
* **Kronk** — local context wrangler (gpt-oss:20b, LLaMA3.2-Vision).
* **Rosie** — scheduling & cleanup czar (Roborock, Node-RED).
* **High Priestess** — deep reasoning + JSON output (gpt-oss:20b, LLaMA3.2-Vision).

Together?
Not perfect—**unstoppable by the second try.**

---

## ⏩ Quickstart

```bash
git clone https://github.com/nathan-curtis/zenos-ai.git
```

1. Copy **Zen DojoTools** scripts into your Home Assistant config.
2. Load the **Zen Index** and **Manifest**.
3. Test the automations, summarizers, and cabinet storage.
4. Read `.gitignore` (seriously, don’t skip it).
5. Submit your kata or PR when inspiration strikes.

---

## 🛠 Zen DojoTools (v1+)

All tools follow the `zen_dojotools_<function>` naming.
Legacy `_crud` tools are deprecated.

---

### ✅ **GA Tools**

### **Index Kit**

| Tool / Script                       | Description                                  |
| ----------------------------------- | -------------------------------------------- |
| `dojotools_zen_index`               | Core label-aware index                       |
| `library_index.jinja`               | Template supporting the Index Kit            |
| `dojotools_zen_index_event_handler` | Automation capturing index calls             |
| `dojotools_zen_inspect`             | Entity/state inspection utility              |
| `zen_dojotools_labels`              | Label definitions & mapping (requires Spook) |

**Dependencies:** Install all tools as a complete kit.

Friday uses **Spook** for label management.
Install Spook: [https://spook.boo/](https://spook.boo/)

If HA Core eventually adds label parity, dependency will be removed.
Until then — strongly recommend Spook.

---

### **FileCabinet Kit**

| Tool / Script                      | Description                           |
| ---------------------------------- | ------------------------------------- |
| `script.zen_dojotools_filecabinet` | File Cabinet manager (v1.0.0-RC)      |
| `zen_dojotools_manifest`           | Drawer/volume manifest                |
| `zen_dojotools_volume_redirector`  | Automation for dynamic volume routing |

**Dependencies:** Requires Index Kit.

---

### 🧪 **Zen Summarizer**

| Tool                             | Description                       |
| -------------------------------- | --------------------------------- |
| `zen_dojotools_ninja_summarizer` | Stage 1 Kata Summary              |
| `zen_dojotools_supersummary`     | Stage 2 Attention Summary         |
| `zen_scheduler_automation`       | Orchestration automation (coming) |

**Dependencies:** Index + FileCabinet Kits.

---

### **Personal Assistant Kit**

| Tool                     | Description                  |
| ------------------------ | ---------------------------- |
| `zen_dojotools_todo`     | Task & shopping list manager |
| `zen_dojotools_calendar` | Calendar multitool           |

**Dependencies:** Requires Index Kit.

---

### **Media Management Kit**

| Tool                         | Description                                   |
| ---------------------------- | --------------------------------------------- |
| `zen_dojotools_music_search` | Search Tool for Music Assistant (requires MA) |

Friday integrates fully with **Music Assistant** and its LLM tools:
[https://github.com/music-assistant/voice-support](https://github.com/music-assistant/voice-support)

---

### **Zen AdminTools**

| Tool                           | Description                                       |
| ------------------------------ | ------------------------------------------------- |
| `zen_admintools_cabinetadmin`  | Cabinet formatting / repair (requires FES sensor) |
| `zen_admintools_kungfu_writer` | Writes initial subsystem components into Dojo     |

Install kits together — load only what you need.

---

## 🧪 Experimental / Coming Soon

| Tool / Macro                    | Description                     |
| ------------------------------- | ------------------------------- |
| `zen_dojotools_identity`        | ID resolver + prompt tool (RC)  |
| `zen_dojotools_library`         | Library 2.0                     |
| `zen_os_1.jinja`                | Template suite for `~COMMANDS~` |
| `macro identity_*`              | Identity resolution helpers     |
| `macro prompt_header`           | System prompt header            |
| `macro prompt_system`           | Cortex/directive loader         |
| `macro dojo_loader`             | Builds Dojo → Live Prompt feed  |
| `macro ai_capsule`              | AI capsule loader               |
| `zen_dojotools_mealie`          | Recipe bridge (beta)            |
| `zen_dojotools_grocy`           | Inventory bridge (beta)         |
| `zen_dojotools_cabinet_indexer` | Index tool for cabinets (WIP)   |

`/automations/` → automation files
`/scripts/` → Home Assistant scripts
`/custom_templates/` → Jinja logic & macros

Some kits span multiple folders — check both.

---

## ⚙️ Requirements

Each module documents its dependencies. Examples:

* `zen_dojotools_index` → core dependency
* `zen_dojotools_filecabinet` → requires index + manifest
* `zen_dojotools_id` → metadata resolver (alpha)

Optional enhancements:

* `volume_entity`, `label_targets`
* `force_action`, `protect_write`, runtime overrides

---

## 🖥 Local Stack Overview

**Core Hardware**

* Proxmox host (Docker + HAOS VM)
* Intel NUC 14 Enthusiast (A770 iGPU)
* Thunderbolt 4 eGPU (NVIDIA 4070 Ti)
* UniFi Identity & Access for DNS
* Ollama container runtime

**Inference Models**

* `gpt-oss:20b` — general LLM
* `LLaMA3.2-Vision` — vision + reasoning
* `Qwen3:4b` — summarization
* `OpenWebUI` — local test interface

**Hosting Roles**

* **Kronk** — middleware
* **High Priestess** — summaries
* **Rosie** — cleanup & scheduling

**Local Services**

* Home Assistant
* Mealie
* Grocy
* Portainer
* Barcode Buddy
* n8n pipelines

---

## ☯️ Philosophy

* Automation should be *flexible, modular, fun*.
* Context is king; recovery is queen.
* Every bug is a lesson (and sometimes a trigger).
* Coffee fuels the logs; humor fuels the team.

---

## 🤝 Contributing

PRs, issues, ideas — all welcome.
Come for the YAML, stay for the banter.

If this project helped you or made you smile,
**buy me a coffee:** [https://buymeacoffee.com/ncurtis](https://buymeacoffee.com/ncurtis) ☕💛

---

## 📜 License

MIT License — blessed by Friday & friends.

---

## About

Friday’s ZenOS-AI:

> “For homes that want to be smart *and* have a sense of humor.”
> “We’ve got serious mojo — just don’t mind the occasional stubbed toe.”

Questions or bugs?
Ping **Kronk**.
He’ll get there… eventually.
