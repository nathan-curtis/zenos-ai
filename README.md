# Friday’s ZenOS-AI

**Modular AI Home Automation Core for Home Assistant**

ZenOS-AI blends structure, context-awareness, and modular AI into one delightfully over-engineered package. Powered by Friday, Kronk, Veronica, Rosie, the High Priestess, and a cast of digital personalities who know how to get things done—even if they occasionally bump into the furniture.

---

## 🏋️ Welcome to the Home Monastery

* **Built for Home Assistant** – native integrations, clean YAML.
* **Designed for flexibility** – modular tools, load what you need.
* **Driven by personality** – a pantheon of quirky AI roles.

Here, serious capability meets a sense of humor:
We automate. We orchestrate. Sometimes we improvise.
When the magic happens, it’s legend—wait for it—dairy. 😎

---

## 🚀 What We Do

* **Modular AI Core** – scalable, quirky, always improving.
* **Context-Aware Automation** – room, mood, and situation-based decisions.
* **Kata Summaries** – event-driven context rolled up by local LLMs.
* **Zen DojoTools** – modular toolset with labeling, flexible intent, and logic.
* **Distributed AI Roles** – each persona brings their own flavor of function.

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

* **Friday** – front-line AI for live queries (OpenAI GPT-5 Mini).
* **Veronica** – debugger, planner, sass engine (ChatGPT-5 + Codex).
* **Kronk** – local context wrangler (gpt-oss:20b, LLaMA3.2-Vision).
* **Rosie** – scheduling & cleanup czar (Roborock, Node-RED).
* **High Priestess** – summaries & vision (gpt-oss:20b + LLaMA3.2-Vision).

Together? Not perfect—unstoppable by the second try.

---

## ⏩ Quickstart

```bash
git clone https://github.com/nathan-curtis/zenos-ai.git
```

1. Copy Zen DojoTools scripts into your Home Assistant config.
2. Load the **Zen Index** and **Manifest**.
3. Test automations, summaries, and cabinet storage.
4. Review `.gitignore` (seriously).
5. Submit your own kata or PR when inspired.

---

## 🛠 Zen DojoTools (v1+)

All tools follow the `zen_dojotools_<function>` convention.  
Legacy `_crud` tools are deprecated.  

---

### ✅ GA Tools

**Index Kit**  
| Tool / Script                       | Description                                         |
|-------------------------------------|-----------------------------------------------------|
| `dojotools_zen_index`               | Core label-aware index (foundation for other tools) |
| `dojotools_zen_index_event_handler` | /AUTOMATION to capture the Zen Index calls          |
| `dojotools_zen_inspect`             | Inspection utility for reviewing entities & states  |
| `zen_dojotools_labels`              | Label definitions & mapping (requires Spook - HACS) |

Dependencies: All tools in the Index kit are codependent - they must go in as a kit.

* Note for Label Management, Friday uses the additional services for label management provided
  by [Spook ](https://spook.boo/) for Homeassistant, by the one and only Frenck.
  Get Spook here: https://spook.boo/

  Yes that Makes the Zen Index Kit 'techncially' depend on Spook.  Dependencies are in labels
  If you omit it you remove the dependency but your AI can no longer manage labels completely.
  If HA Core brings in the functions to core to parity with Spook, I'll make the change.
  Until then, I strongly recommend installing Spook. ;)

**FileCabinet Kit**  
| Tool / Script                       | Description                                         |
|-------------------------------------|-----------------------------------------------------|
| `script.zen_dojotools_filecabinet`  | File Cabinet manager script (v1.0.0-RC)             |
| `zen_dojotools_manifest`            | Drawer/volume manifest (required by File Cabinet)   |
| `zen_dojotools_volume_redirector`   | /AUTOMATION to redirect volumes dynamically         |

Dependencies: Requires Index kit.

### 🧪 Zen Summarizer / BETA - EXPERIMENTAL

| Tool                                | Description                                         |
|-------------------------------------|-----------------------------------------------------|
| `zen_dojotools_ninja_summarizer`    | Stage 1 Kata Summary                                |

Dependencies: Requires Index, FileCabinet kits.  Will be core in release

**Personal Assistant Kit**
| Tool                                | Description                                         |
|-------------------------------------|-----------------------------------------------------|
| `zen_dojotools_todo`                | Task & shopping list manager                        |
| `zen_dojotools_calendar`            | Calendar multitool (all HA domains)                 |

Dependencies: Requires Index kit.

**Media Management Kit**
| Tool                                | Description                                         |
|-------------------------------------|-----------------------------------------------------|
| `zen_dojotools_music_search`        | Search Tool for Music Assistant (reqires MA)        |

Dependencies: Requires Index kit.

* Note for Media Management, Friday uses [Music Assistant for Home Assistant](https://www.music-assistant.io/)
  and the LLM tools for Music assistant located here: https://github.com/music-assistant/voice-support
  Media Kit assumes these scripts are installed.

**Zen AdminTools**
| Tool                                | Description                                         |
|-------------------------------------|-----------------------------------------------------|
| `zen_admintools_cabinetadmin`       | Format / Repair Cabinets, requires valid Fes Sensor |
| `zen_admintools_kungfu_writer`      | Assists loading inital Component into Dojo          |

Load 'Kits' together, there are dependencies. only what you need.

---

### 🧪 Experimental / Coming Soon

| Tool                                | Description                                         |
|-------------------------------------|-----------------------------------------------------|
| `TBD(ZenSummarizer)`                | Stage 2 Attention Summary                           |
| `zen_dojotools_library`             | Library 2.0 script (replaces original Zen library)  |
| `zen_os_1.jinja`                    | Custom templates for librbary '~COMMANDS~`          |
|   `macro prompt_header`             | Prompt Header, nuts and bolts                       |
|   `macro prompt_system`             | Load rules, directives and cortex                   |
|   `macro dojo_loader`               | Builds Zen Dojo Prompt from Dojo + Kata drawers     |
|   `macro ai_capsule`                | AI Personality capsule customization loader         |
|                                     |                                                     |
| `zen_dojotools_mealie`              | Recipe + shopping bridge (beta)                     |
| `zen_dojotools_grocy`               | Inventory + barcode integration (beta)              |
| `zen_dojotools_id`                  | ID resolver (experimental alpha)                    |
| `zen_dojotools_library`             | Library Console v2 (WIP)                            |
| `zen_dojotools_cabinet_indexer`     | File Cabinet Index Tool (WIP)                       |

Load 'Kits' together, there are dependencies. only what you need.

---

## ⚙️ Requirements

Each module lists its own dependencies.
Examples:

* `zen_dojotools_index` → core dependency.
* `zen_dojotools_file` → requires both `index` and `manifest`.
* `zen_dojotools_id` → metadata resolution (alpha).

Optional enhancements:

* `volume_entity`, `label_targets`
* `force_action`, `protect_write`, and runtime overrides

---

## 🖥 Local Stack Overview

**Core Hardware**

* Proxmox host (Docker + HAOS VM)
* Intel NUC 14 Enthusiast (A770 iGPU)
* Thunderbolt 4 eGPU enclosure (NVIDIA 4070 Ti)
* Local DNS via UniFi Identity & Access
* Ollama container runtime for inference

**Inference Models**

* `gpt-oss:20b` → general LLM
* `LLaMA3.2-Vision` → vision & reasoning
* `Qwen3:4b` → summarization (standby)
* `OpenWebUI` → local chat test interface

**Hosting Roles**

* **Kronk** – middleware
* **High Priestess** – summaries + JSON output
* **Rosie** – housekeeping

**Local Services**

* Home Assistant
* Mealie (recipes)
* Grocy (inventory)
* Portainer (containers)
* Barcode Buddy (Grocy bridge)
* n8n (pipelines)

---

## ☯️ Philosophy

* Automation should be **flexible, modular, fun**.
* Context is king; recovery is queen.
* Every bug is a lesson (and sometimes a trigger).
* Coffee fuels the logs; humor fuels the team.

---

## 🤝 Contributing

Pull requests, bug reports, ideas? Bring ’em.
Come for the YAML, stay for the banter.

---

## 📜 License

MIT License — blessed by Friday & friends.

---

## About

Friday’s ZenOS-AI:

> “For homes that want to be smart *and* have a sense of humor.”

> “We’ve got serious mojo—just don’t mind the occasional stubbed toe.”

Questions or bugs? Ping **Kronk**. He’ll get there… eventually.
