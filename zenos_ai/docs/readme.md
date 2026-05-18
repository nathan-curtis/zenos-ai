# 📘 **ZenOS-AI Documentation Hub**

> **Version:** 2026.6.0 | **Last Updated:** May 2026 | **License:** MIT
>
> *Public releases follow Home Assistant's `YYYY.M.patch` convention — `2026.6.0` is the June release 'Clue'. A new month resets to `.0`.*

→ [Project Overview & Install](../../README.md)

---

> ### What's New in 2026.6.0 'Clue'
>
> **1. Room Manager (RoomReg) v1.42.0.** The spatial intelligence hub. Stores physical room topology — portals, adjacency, exits, safety equipment. Provides live context slices (+light/+climate/+media/+topo/+security), whole-home `home_overview` with weather + plant + home_mode snapshots, and scenario-aware emergency routing. Every room-aware tool reads from here.
>
> **2. Plant Manager v1.2.2.** New physical plant + energy manager. Electric: live watts, billing, tariff, grid carbon, panel status. Water, gas, HVAC, mechanical (water heater + sump), and circuit breakdown. Label-first discovery with `zen_plant_*` pin overrides. `mode=validate` shows every slot's resolution status.
>
> **3. Media Manager (NyxMau5) + Security Manager v1.2.0.** Media Manager handles whole-home discovery, source management, and intent routing. Security Manager replaces alarm_panel — room-aware zone inventory via RM +security slice, `cameras_by_area` cross-reference.
>
> **4. OOBE v4.2.0.** First-run room setup is now RM-native. The AI registers rooms directly into Room Manager topology (portals, adjacency, exits, rally point, safety equipment) instead of writing flat filecabinet drawers.
>
> → [Full Release Notes — Fry's Grandpa](releases/frys_grandpa.md) | [Room Manager](room_manager.md) | [Plant Manager](plant_manager.md)

---

Welcome to the **ZenOS-AI Documentation** — the full map of the architecture, tools, cognitive model, and operational philosophy behind *Friday’s Party*.

ZenOS-AI turns **Home Assistant** into a real agentic, persona-aware operating system. This documentation explains how the pieces fit together: the **Cabinet System** for identity and memory, the **Monastery** for reasoning, the **KF4 action pipeline** for awareness and action, and the **HyperIndex** for graph-based attention and discovery.

If you're building an AI construct, designing a DojoTool, wiring the action pipeline, or just trying to understand how Friday thinks, this directory is your guide.

---

# 📚 **Included Documentation**

This directory contains **12 documentation suites**, each aligned with a major subsystem in ZenOS-AI.

---

## 🚀 **0. Getting Started**

**Folder:** `docs/getting_started/`

New to ZenOS-AI? Start here.

* `install.md` — File copy, configuration.yaml setup, conversation agent prompt, set conversation agent before restart, restart, health verification
* `first_run.md` — First boot walkthrough, OOBE conversation, persona selector, editing profiles, troubleshooting
* `entity_exposure.md` — What to expose to your conversation agent: actionable vs contextable vs invisible, the three-tier model
* `cabinet_placement.md` — Where things go and why: Dojo vs Kata, drawer vs KFC, the quick-reference placement table. Read after entity_exposure.
* `oobe.md` — OOBE walkthrough: the six-step first-boot configuration protocol to your conversation agent: actionable vs contextable vs invisible, the three-tier model
* `troubleshooting.md` — Gauges → Kill Switches → Repair Tools. Health sensor quick-reads, summarizer kill switches, and a seven-step graduated repair sequence (resolver refresh → reseed → label reset → nuclear cabinet reset)
* `user_management.md` — Add/remove/move AI users and human users. Provision new identity cabinets, deprovision or swap existing ones, transfer default labels, and perform targeted identity-layer repairs or full nukes.

If you just installed ZenOS-AI and want to know what to do next, start here.

---

## 🛠️ **1. Tool Reference**

**Folder:** `docs/` (root-level)

Reference docs for every major ZenOS-AI tool. Each covers modes, discovery, parameters, and response shape.

* `room_manager.md` — Room Manager (RoomReg): spatial topology, context slices, emergency routing, home_overview, utility index
* `plant_manager.md` — Plant Manager: electric, water, gas, HVAC, mechanical, circuits, validate
* `media_manager.md` — Media Manager (NyxMau5): whole-home discovery, source management, intent routing
* `spamaster.md` — SpaMaster: spa/hot tub management, ESPHome discovery, scene/chemistry/log
* `alertmanager.md` — AlertManager: severity labels, priority inject, auto-expiry, GC sweep
* `zenlux.md` — ZenLux: lighting scenes, bleed-aware control, media awareness, sync_shades
* `zenshade.md` — ZenShade: cover management, tilt support, barrier exclusion, ZenLux sync

---

## 🧠 **2. Architecture**

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
* `09_Identity_Architecture.md` – Identity data model spec
* `11_RoomState_and_Perception.md` – Sensory model
* `14_Abbot_Scheduler_And_Task_Economy.md`
* `18_Context_Frame_Operational_Cognitive_Surface.md` – Context assembly + prompt loader
* `19_Resilience_and_Failure_Modes.md` – Highlander resolver, health sensor stack
* `20_tool_invocation_and_security.md` – Tool ACLs, safety classes, caller_token
* `security_model_ga.md` – **Operator reference:** what's active at GA vs SP1

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

* `zen_os1_jinja.md` — Core prompt assembly engine
* `zen_query_jinja.md` — ZQ-1 filter engine
* `zenos_cabinets_jinja.md` — Cabinet macro library (new in 2026.4.0): canonical safe drawer I/O, FG-38 normalization encapsulated

This suite defines how Friday constructs her thoughts.

---

## 🥋 **4. Kung Fu Components**

**Folder:** `docs/kung_fu/`

Each Kung Fu component is a discipline: a subsystem Friday loads at runtime.

Documents:

* `understanding_kf4.md` — **Start here.** Plain-language guide to the Dojo, Kung Fu Components, and the KF4 action pipeline. How to add a new component in five steps, no code required.
* `building_a_kfc.md` — Step-by-step build guide with worked example.
* `alert_manager.md` — AlertManager KFC: severity labels, fire-once dedup, TTL, priority inject wiring.
* `taskmaster.md` — Taskmaster KFC: AI conductor queue, task lifecycle, Abbot dispatch.
* `readme.md` — Technical spec: drawer schema, trigger ID reference, command strip migration notes.

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

* `whitepaper_cognitive_architectures.md` – Theory behind the Monastery, Summarizers, and Cabinets

Good for deep dives and formal reasoning.

---

## ⚙️ **7. Script Modules**

**Folder:** `docs/scripts/`

Documentation for every Zen DojoTool and script module.

Includes:

* `zen_dojotools_admintools_readme.md` — AdminTools: KungFu Writer, cabinet repair, template press, prompt loader, nuclear label reset, reset_all cabinet sequence
* `zen_dojotools_scheduler_readme.md` — Scheduler: trigger IDs, Dojo-driven dispatch, component subscription, force events, hardware trigger pattern
* `zen_dojotools_summarizers_readme.md` — Ninja Summarizer + SuperSummary: kill switches, active component selection, monk pipeline
* `zen_dojotools_library_readme.md` — Library: command interpreter dispatch, hash_md5, slugify
* `zen_home_mode_readme.md` — Home Mode: 8-state machine, schedule anchors, quiet/work hours, scheduler trigger IDs
* `zen_dojotools_filecabinet_readme.md` — Cabinet read/write controller, clone action, Highlander mode
* `zen_dojotools_manifest_readme.md`
* `zen_dojotools_inspect_readme.md`
* `zen_dojotools_index_readme.md`
* `zen_dojotools_hyperindex_readme.md`
* `zen_dojotools_query_readme.md`
* `zen_dojotools_camera_readme.md` — Camera: ai_task gate, look/scan, dynamic cabinet routing, Security Manager lens pattern
* `zen_dojotools_postman_readme.md` — Postman: ack loop, actionable notifications, image support
* `zen_dojotools_office_readme.md`
* `zen_dojotools_event_emitter_readme.md`
* `readme.md` – Overview

Scripts are the motor cortex. They turn reasoning into action.

---

## ⚕️ **8. Health Sensors**

**Folder:** `docs/sensors/`

Layered health monitoring stack — cabinet resolvers, cognition pipeline, agent bootability.

* `readme.md` — full reference: 7 always-live cabinet resolver sensors + 6 trigger-based health sensors (including `sensor.zen_prompt_health` — prompt integrity), states, conditions, attributes, troubleshooting quick-reference. Includes `zen_health_report` (full system diagnostic) and `zen_resolver_refresh` (cold-start recovery).

---

## 🧩 **9. Zen HyperIndex**

**Folder:** `docs/zen_hyperindex/`

Documentation for the recursive hypergraph-driven index system.

* `zen_hyperindex_overview.md`
* `zq1_patterns.md` — 10 real index call patterns: ghost hunt, drift detector, coverage gap analysis, power spike triage, KFC intelligence pull, access gate, full system graph, area slice, NOT query, and the history-in-inspect forward reference. Includes the full ZQ-1 filter key reference table.

If Cabinets are the filesystem, HyperIndex is the search engine plus attention model.

---

## 🧠 **10. Zen Summarizer**

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

This is Friday’s working memory engine.

---

## 🔐 **11. Identity & Security Model**

**Folder:** `docs/architecture/`

The Identity subsystem defines:

 * who a construct is allowed to be
 * what it may see
 * where its authority begins and ends

Two documents cover this:

* `09_Identity_Architecture.md` — full identity data model spec: GUIDs, identity hashes,
  provenance chains, essence capsules, ACL rules, Squirrel Safe / Content Safe filters,
  session tokens, visas, delegated capability (v1.5). The authoritative structural spec.

* `security_model_ga.md` — **start here if you’re an operator.** What is active at GA,
  what is stubbed for SP1, the `security_policy` syscab drawer, caller_token plumbing,
  prompt integrity sensor (`zen_prompt_health`), delegation and nesting hard rules, and
  the SP1 claims engine architecture. No jargon — written for someone deploying the system.

This is Friday’s trust spine — the system that decides which parts of the world are even visible before reasoning begins.

---

## 🗺️ **12. Roadmap**

**File:** `docs/roadmap.md`

**2026.6.0 'Clue' — Shipped (2026-05-17)**

* Room Manager (RoomReg) v1.42.0 — spatial topology hub, context slices, home_overview with plant/weather/home_mode, emergency routing
* Plant Manager v1.2.2 — physical plant + energy: electric, water, gas, HVAC, mechanical, circuits, validate
* Media Manager (NyxMau5) v0.7.2 — whole-home media management and intent routing
* Security Manager v1.2.0 — replaces alarm_panel; room-aware zone inventory via RM +security slice
* ZenShade v0.2.2 — cover management, tilt support, ZenLux sync
* OOBE v4.2.0 — RM-native room setup flow (portals, adjacency, exits, rally point, safety equipment)
* Office v5.0.0 — todo + calendar split into standalone dojotools
* AlertManager v1.3.0 — severity labels, 24h auto-expiry, DojoTools Core GC sweep
* calderaspas retired — SpaMaster v3.3.0 is the replacement

See: [Release Notes — Clue](releases/clue.md)

---

**2026.5.0 'Fry's Grandpa' — Shipped (2026-05-03)**

* Priority inject — error/life-safety alerts in `_zen_priority_inject`, `zen_priority_context` sensor, NOTIFICATIONS block in every AI prompt
* Alertmanager v1.2.0 — postman as primary notify target, `notification_router` deprecated, priority inject auto-wired
* Camera v1.3.0 — `set_alert_policy` mode, `sendto sensor.*` dynamic cabinet routing, `_default_ctx`/`_alert_policy` preserved across look/scan
* Identity `provision_member` — provision external family member (no HA account) into expansion slot in one call; OOBE updated
* ZQ-1 v4.6.0 — `regex` corrected to `regex_search()`, `entity_id_regex` filter (step 10b), `stats_eligible` filter
* Profile editor — fixed `mode: read` for user/family returning blank; fixed write merge and `_write_ok` key

See: [Release Notes — Fry's Grandpa](releases/frys_grandpa.md)

---

**2026.4.3 'Lights, Camera, Action' — Shipped (2026-04-24)**

* ZQ-1 exclusion suite, compound Index Command DSL, Camera v1.2.0, Postman v1.0.0

See: [Release Notes — Lights, Camera, Action](releases/lights_camera_action.md)

---

**2026.3.1 — Shipped (2026-03-27)**

* Identity graph, alertmanager, GUID correctness, user cabinet essence seeding fix (Phil's edge case)

**2026.3.0 'Ready Player Two' — Shipped (2026-03-26)**

* Identity and lifecycle release — cabinet provisioning, warmup state machine, full household/family group management
* Profile editor, zenai_essence, provisioner — all FG-38/FG-40 hardened and GA-ready
* Cortex 32 (True Voice) is now the default prompt primitive set
* SP1 — queued: `caller_token` enforcement, KFC 1.1, security architecture
* v.next — deeper memory, governance modules, cabinet import/export

See: [Release Notes — Ready Player Two](releases/ready_player_two.md)

---

# 🧭 **Recommended Reading Order**

1. Getting Started (new install? start here)
2. Architecture
3. Cabinet System
4. HyperIndex
5. Summarizer
6. Library
7. Scripts
8. Kung Fu Components
9. Roadmap

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
