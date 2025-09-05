Friday's ZenOS-AI

Modular AI Home Automation Core for Home Assistant

Powered by Friday, Kronk, Veronica, Rosie, the High Priestess, and a cast of AI personalities who know how to get things done - even if they occasionally bump into the furniture.


---

🏋️ Welcome to the Home Monastery

ZenOS-AI brings structured automation, cheeky context-awareness, and modular AI together under one roof.

Built for Home Assistant

Designed for flexibility

Driven by a lively pantheon of digital personalities


Here, serious capability meets a sense of humor:
We automate. We orchestrate. Sometimes we improvise.
When the magic happens, it's legend - wait for it - dairy... 😎


---

What We Do (and Where We're Headed)

Modular AI Core: Friday's ZenOS-AI is scalable, quirky, and always improving.

Context-Aware Automation: Dynamic room, mood, and situation-based decisions.

Kata Summaries: Event-driven summarization offloaded to local LLMs.

Zen DojoTools: Modular toolset with smart labeling, flexible intent handling, and context-driven logic.

Distributed AI Roles: Personalities with specific functions and charming quirks.



---

🏠 Meet the Pantheon

Name	Title	Specialty

Friday	Chief Enlightenment Officer	Mastermind, coordinator
Veronica	Snarky Supervisor	Attitude, taste, clarity
Kronk	Curator of the Monastery	Context & kata wrangling
Rosie	Mistress of Cleanliness	Clean code, clean logs
High Priestess	Divine Automation Overseer	YAML exorcist


Roles:

Friday: Frontline AI; handles live queries with fast LLMs. (GPT-5 Mini, OpenAI)

Veronica: Script debugger, planner, and sass engine. Uses GPT-5 Mini, OpenAI ChatGPT-5, and Codex as needed for creative and technical reasoning.

Kronk: Local model for context + lookup. Runs locally on a dedicated GPU box with NVIDIA acceleration. Uses gpt-oss:20b for inference and LLaMA3.2-Vision for visual tasks.

Rosie: Scheduling & cleanup czar. (Roborock S7 MaxV integrator)

High Priestess: Handles deep summarization and vision processing. Runs both LLaMA3.2-Vision and gpt-oss:20b for inference and visual reasoning.


Together? Not perfect - but usually unstoppable by the second try.


---

⏩ Quickstart

git clone https://github.com/nathan-curtis/zenos-ai.git

1. Copy Zen DojoTools scripts into your Home Assistant config.


2. Load the Zen Index and Manifest.


3. Test the automations, summaries, and file/cabinet storage.


4. Review .gitignore (seriously).


5. Submit your own kata or PR when inspired.




---

Zen DojoTools (v1+)

All tools follow the zen_dojotools_<function> convention as of v1.0.0+. Legacy _crud tools are deprecated.

Tool	Description

zen_dojotools_todo	Task & shopping list manager
zen_dojotools_calendar	Calendar multitool for all HA domains
zen_dojotools_file	File Cabinet volume manager (v1.0.0-RC)
zen_dojotools_mealie	Recipe + shopping bridge (beta)
zen_dojotools_grocy	Inventory + barcode integration (beta)
zen_dojotools_index	Label-aware manifest index (required)
zen_dojotools_manifest	Drawer/volume manifest (required by File tool)
zen_dojotools_id	Tool/label/entity resolver (experimental alpha)
zen_dojotools_library	Library Index v2 (WIP)


Each tool is fully modular. Load only what you need.


---

Requirements

Each Zen DojoTool module is self-contained and will list its own required dependencies.

Examples:

zen_dojotools_index is a core dependency for most tools.

zen_dojotools_file (File Cabinet) specifically requires both zen_dojotools_index and zen_dojotools_manifest.

zen_dojotools_id (experimental alpha) provides metadata resolution and tool-to-entity mapping.


Optional enhancements include support for:

volume_entity

label_targets

force_action, protect_write, and other runtime overrides



---

Local Stack Overview

Core Hardware:

Proxmox host running Docker and HAOS VM

Intel NUC 14 Enthusiast w/ A770 iGPU

External Thunderbolt 4 eGPU enclosure (NVIDIA 4070 Ti)

Local DNS, routed through UniFi Identity & Access

Inference traffic routed via Ollama container runtime


Inference Models:

gpt-oss:20b → General-purpose LLM

LLaMA3.2-Vision → Vision, object reasoning, scene summary

Qwen3:4b → Context summarization (on standby)

OpenWebUI → Local chat test interface


Hosting Roles:

Kronk = orchestration & middleware

High Priestess = summaries + vision + JSON output

Rosie = housekeeping via HA + Node-RED


Local Services:

Home Assistant (automations)

Mealie (recipes & meal planning)

Grocy (inventory + shopping)

Portainer (container UI)

Barcode Buddy (Grocy bridge)

n8n (automation pipelines)



---

Philosophy

Automation should be flexible, modular, and fun.

Context is king; recovery is queen.

Every bug is a lesson (and an automation trigger).

Coffee fuels the logs; humor fuels the team.



---

Contributing

Pull requests, bug reports, ideas? Bring 'em.
Come for the YAML, stay for the banter.


---

License

MIT License - blessed by Friday & friends.
See LICENSE.


---

☯️ About

Friday’s ZenOS-AI:

> For homes that want to be smart and have a sense of humor.



> “We’ve got serious mojo - just don’t mind the occasional stubbed toe.”



Questions or bugs? Ping Kronk. He’ll get there. Eventually.

