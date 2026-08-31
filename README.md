# Friday's ZenOS-AI

### A Modular, Context-Aware AI Home Automation Framework for Home Assistant

ZenOS-AI blends structure, personality, and unapologetic over-engineering into a living system that powers your home with **Friday, Kronk, Veronica, Rosie, the High Priestess, Cait, and Nyx** — a coordinated AI pantheon that takes its jobs seriously (even if it doesn't always take itself seriously).

Welcome to the **Home Monastery**.

Let's automate everything that isn't nailed down.

And a few things that are.

**Stable: 2026.8.0 'Chef'** | Previous: 2026.7.1 (patch on 2026.7.0 'Neo')
**RC1: 2026.9.0 'Steel Magnolia'** — ETA Saturday, 2026-09-05. See [release notes](zenos_ai/docs/releases/steel_magnolia.md).

> **Versioning:** Public ZenOS releases follow Home Assistant's `YYYY.M.patch` convention — if you're already running HA, you already know this clock. Internal architecture versioning (`5.1.x` series) is retained in commit history and internal tooling.

> Found a bug? Report it in the **[Friday's Party community thread](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/)** or open a **[GitHub issue](../../issues)**. Include your HA version, the relevant tool name, and what you expected vs. what happened.

**What's in 2026.9.0 'Steel Magnolia' (RC1):** *The duck looks calm, and the fence is exactly where it's supposed to be.*

**Room Manager v3** reads a room by what's actually happening in the space right now, not a clock — a real wasp-in-a-box occupancy model, opt-in `entertaining_hold`/`guest_hold`/`presence_hold`, a nighttime-gated asleep window, hold-release timer restart, a generic `trouble` attribute with native SPAN breaker confirmation, and unconditional Paused/Emergency cascade through nested rooms.

**Identity gates** now cover locks, exterior covers, the alarm panel, room unpause/topology edits, ZenLux, climate, ZenZork, and the spa tool — risky actuations require an explicit household certification, with the highest-risk actions asking live every time. A real deny primitive, centralized scope resolution, and a new admin-only CertAdmin tool round it out.

**Security Manager** gets a full parity build against the legacy automation it replaces — new arm/disarm/mode-drive flags, a vacation wake-shift, and live security requests (clearing Paused, a cert grant, disarming) that never silently wait until morning to notify someone.

**ZenLux** picks up Spook 5.2's adjust-only light controls and real `switch.*` control; REFLEX now fires every scene through it instead of bypassing its guards, and its rehearsal mode is fully independent of the live-fire switch in every direction.

**AutoVac** gets full Roborock support (battery, wear tracking, room targeting, manual-run tracking) plus a fix for automatic room election, which had silently never worked.

**ZenZork** ships its first real content chapter and a v1.8.0 follow-up — weighted loot, 16 quests, a corrected book-lore sequence, Game Genie cheat codes, a SoftDisk-style per-chapter release model, real persisted achievements, and a turn-based threat/combat system.

**Hospitality lifecycle** gains an arrival-prep nudge and a checkout nudge with humanized local timestamps, one shared occupant-prefs lookup across Kitchen/Twenty CRM, and a fix keeping guest-stay status from reporting stale.

**Also in this release:** ZQ-1 flags bad query filters instead of silently returning nothing; recorder history stats stop lying about energy sensors; `sensor.home_overview` now feeds Friday's prompt real per-room state; manifest's scan modes share one audited collector; Taskmaster gains a `catch_up` mode and closes a self-sustaining alert loop between two summarizer components; Kitchen's search now finds recipes by tag; an AI persona's identity can no longer be silently reset on bootstrap; architecture docs were checked chapter-by-chapter against actual code.

→ [Release Notes — Steel Magnolia](zenos_ai/docs/releases/steel_magnolia.md)

---

**2026.8.0 'Chef'** — *Yes, chef.* Taskmaster becomes a real cross-backend expediter. Kitchen (Mealie) gains a full fulfillment/costing layer plus an executive-chef batch. SP1 identity gate backs the first real gated capability, Portainer container control. Twenty CRM and Room Manager share one guest/occupant-prefs lookup.

Release notes: [Chef](zenos_ai/docs/releases/chef.md)

---

**2026.7.0 'Neo'** (+ 7.1 patch) — *I know Kung Fu.* CabCeption (FileCabinet v6.2.0 nested drawer trees), Tapestry drawer composer, Tool Manifest self-description, Lens Bus `stack=` routing, five new plugins (Zammad, Wiki.js, Paperless-NGX, Twenty CRM, Firefly III), ZenZork. 7.1 patch adds KF5 self-registering tools, Firefly III depreciation/COGS codex tier, Grocy fixes, Battery Notes Lens Bus provider.

Release notes: [Neo (incl. 7.1 patch)](zenos_ai/docs/releases/neo.md)

---

**2026.6.0 'Clue'** — shipped 2026-06-01. Room Manager spatial topology, AutoVac, Grocy v5.2.0, Identity presence block, Plant Manager v5.4.0, Media Manager, Security Manager, ZenShade, Cortex v42 'The Answer'.

Release notes: [Clue](zenos_ai/docs/releases/clue.md) | [Fry's Grandpa](zenos_ai/docs/releases/frys_grandpa.md) | [Lights, Camera, Action](zenos_ai/docs/releases/lights_camera_action.md) | [Action Jackson 2](zenos_ai/docs/releases/action_jackson_2.md) | [Action Jackson](zenos_ai/docs/releases/action_jackson.md) | [Ectoplasm](zenos_ai/docs/releases/ectoplasm.md) | [Ready Player Two](zenos_ai/docs/releases/ready_player_two.md)

---

## 🚀 Start Here

| | |
|---|---|
| New install | **[Install Guide](zenos_ai/docs/getting_started/install.md)** |
| First boot | **[First Run & OOBE](zenos_ai/docs/getting_started/first_run.md)** |
| First component | **[AutoVac Quick Start](zenos_ai/docs/getting_started/autovac_quick_start.md)** — touches every part of the system in one visible loop |
| Adding a component | **[Understanding KF4](zenos_ai/docs/kung_fu/understanding_kf4.md)** · **[Building a KFC](zenos_ai/docs/kung_fu/building_a_kfc.md)** |
| Full documentation | **[Documentation Hub](zenos_ai/docs/readme.md)** |

---

# What Is ZenOS-AI?

ZenOS-AI is a modular AI and automation architecture built on:

- **Home Assistant**
- **Home Assistant Packages** (canonical configuration layer)
- **Structured contextual memory** (“Cabinets” and “Drawers”)
- **Event-driven Kata summaries**
- **Local and distributed inference engines**
- **A multi-persona AI team**

Together these components create a **privacy-first, locally hosted intelligent home system** that can:

• reason about context  
• store structured long-term memory  
• automate reflexively  
• summarize system activity  
• enforce identity and privilege boundaries  
• communicate clearly  
• occasionally sigh at your poor life choices

The system works because it is **modular**.

It is delightful because it is **chaotic-good**.

---

# Architecture Overview

ZenOS-AI is structured around **Home Assistant Packages**, which form the canonical configuration layer.

Everything lives under:

```

packages/zenos_ai/

```

Packages define the **spine of the system**.

DojoTools scripts provide runtime behavior.
Cabinets persist memory.
The Monastery performs reasoning.
Flynn guards the grid.

---

# Layered Architecture

ZenOS-AI operates in **concentric rings**, separating definition, runtime cognition, and administrative tooling.

| Ring | Name | What it does |
|------|------|-------------|
| Ring-0 | Core Kit | Defines the rules: labels, cabinet schema, identity contracts, event bus, health sensors. Does not run behavior. If Ring-0 breaks, Friday forgets who she is. |
| Ring-1 | Cognitive Runtime | Binds behavior to Ring-0’s definitions: DojoTools scripts, KF4 pipeline, prompt compilation, persona capsules, conversation agent interface. This is where Friday thinks. |
| Ring-2 | Admin & Recovery | Maintenance, repair, and recovery: cabinet repair, manifest writing, KFC loading, identity audit. The “don’t panic” layer. |

---

## Flynn (System Sentinel)

Flynn is not a persona. Flynn is not an assistant. Flynn is the **personification of the system itself**.

He is the sentinel that stands between a cold boot and a live agent. Before Friday steps onstage, Flynn walks the grid. He checks the resolvers, validates the cabinet headers, confirms the Dojo is stocked, probes the conversation agent, and reads every health sensor on the board. If the grid isn't ready, the agent doesn't step out. Flynn says so — clearly, with a persistent notification and a plain-English diagnosis — and stands down until the problem is resolved.

You should only need to think about Flynn when something is wrong or when you're onboarding. If the system is healthy, he is invisible. If you're meeting him at runtime, he has something to tell you.

**Flynn's clipboard** — the gauges he reads and reports at every boot:

- Cabinet resolver states (all 7)
- Cabinet volume header integrity (`gc_eligible`, schema version, flags)
- Dojo KFC drawer presence (are the components there?)
- Template freshness (`zen_template`, `kfc_template`)
- Conversation agent liveness
- Essence presence (does the AI user know who she is?)
- Prompt integrity (`sensor.zen_prompt_health` — schema, signature, manifest)

All 9 health sensors are auto-provisioned with the correct labels by Flynn on a fresh install. No manual tagging.

When Flynn onboards a fresh install, his clipboard is the checklist. When Flynn monitors a running system, his clipboard is the health report. The agent inherits a clean stage or not at all.

Flynn is defined in `packages/zenos_ai/flynn.yaml`. He is the first thing that runs and the last thing you want to debug.

---

## The Monastery (Reasoning Backend)

The Monastery is external to Home Assistant but essential to the system.

It provides **long-form reasoning** and reflective cognition.

The Monastery:

• accepts structured system state  
• produces **Kata summaries**  
• generates **Supersummaries**  
• enforces the *Order of the Monastery* (no hallucination)  
• acts as Friday’s extended cognition

Without the Monastery, Friday remains functional — but **reflexive and shallow**.

---

# Installation

ZenOS-AI installs as a **Home Assistant package collection**.

---

## Requirements

• Home Assistant 2025.x+
• A conversation agent with tool-calling support (models under ~4B parameters released before Nov 2025 or ~8B parameters beforehand, or with short context windows are not recommended.  Your CTX must hold the HA live state data, Tools manifest AND the ZenosPrompt. The ZenOS goals is to target a ~64K or smaller context as to be able to run the system locally on a single 16G GPU)
• Spook integration (installable via HACS)

---

## Installation Steps

1. Copy `packages/zenos_ai/` into your HA config under `packages/`
2. Copy `custom_templates/zenos_ai/` into your HA config under `custom_templates/`
3. Add to `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

4. Paste the conversation agent prompt template (from `custom_templates/zenos_ai/conversation_agent_prompt_template.yaml`) into your conversation agent's system prompt in HA
5. Restart Home Assistant — Flynn initializes automatically on first boot
6. Set `input_text.zenos_conversation_agent` (Settings → Helpers) to your conversation agent entity ID
7. Check `sensor.zen_agent_health` — should report `ok`

For the full walkthrough including helper configuration and troubleshooting, see the **[Install Guide](zenos_ai/docs/getting_started/install.md)**.

Plugins under `packages/zenos_ai/plugins/` are optional — install only what you need.

---

# What to Expose to Your Conversation Agent

Not everything in your HA install should be visible to Friday. ZenOS-AI uses a three-tier model:

| Tier | Rule | How |
|---|---|---|
| **Actionable** | Friday needs to control it or read it immediately | Expose directly to the conversation agent |
| **Contextable** | Friday should know about it | Tag with labels — HyperIndex finds it automatically |
| **Invisible** | Friday never needs it | Neither exposed nor labeled |

**Always expose:** All `script.zen_dojotools_*` tools. These are Friday's hands. Keep everything else minimal.

**Never expose:** AdminTools scripts, cabinet sensors, health sensors, raw telemetry, or anything containing credentials. (`zen_dojotools_scribe` is the MCP-exposed KFC registration tool — it lives in the DojoTools namespace, not AdminTools.)

**Index everything else:** If it feeds a KFC component's Kata, it belongs behind a label — not in the tool list. One label on 50 sensors produces a rich, token-efficient context block. 50 individual direct reads does not.

→ **[Full entity exposure guide](zenos_ai/docs/getting_started/entity_exposure.md)**

---

# Package Structure

```
packages/zenos_ai/
  zenos_cabinets.yaml          — Cabinet definitions and volume routing
  flynn.yaml                   — Flynn Stepgate Sentinel + bootstrap engine
  flynn_oobe.yaml              — OOBE protocol driver (run / complete / status)

  dojotools/
    — Core infrastructure —
    dojotools_filecabinet.yaml — FileCabinet v6.2.0 — CabCeption nested drawer trees, VirtualDrawer, LiveDrawer
    dojotools_core.yaml        — Core operations + FileCabinet GC
    dojotools_scheduler.yaml   — Scheduled automation triggers
    dojotools_manifest.yaml    — Manifest engine
    dojotools_library.yaml     — Library tools
    dojotools_history.yaml     — History management
    dojotools_utilities.yaml   — General utilities
    zen_home_mode.yaml         — 8-state home mode machine + schedule anchors

    — Identity & Memory —
    dojotools_identity.yaml    — Identity resolver and privilege model
    dojotools_profile.yaml     — Profile editor (ai_user / household / user / family)
    dojotools_provisioner.yaml — Identity provisioning (household / user / AI user)
    dojotools_labels.yaml      — Label inspection and management
    dojotools_index.yaml       — Index, HyperIndex, ZQ-1 query engine
    dojotools_admintools.yaml  — Cabinet repair, manifest write, KFC loader, prompt loader
    dojotools_summarizers.yaml — Kata and Supersummary engines
    dojotools_systemtools.yaml — System tools and event emitter
    dojotools_dispatcher.yaml  — Two-level urgency dispatch router
    dojotools_kungfu_loader.yaml — KFC component loader and lifecycle

    — AI Interface —
    dojotools_scribe.yaml      — KF4 artifact authoring (MCP-exposed KFC registration)
    dojotools_ectoplasm.yaml   — Spook/HA extended surface (areas, floors, entity lifecycle)
    dojotools_camera.yaml      — Camera tool — look/scan, alert policy, dynamic cabinet routing
    dojotools_postman.yaml     — Notifications — ack loop, actionable alerts, image support
    dojotools_image_generator.yaml — Image generation tool

    — Home Systems —
    dojotools_room_manager.yaml   — Room Manager (RoomReg) — spatial topology, context slices
    dojotools_plant.yaml          — Plant Manager — electric, water, gas, HVAC, mechanical, energy
    dojotools_media_manager.yaml  — Media Manager (NyxMau5) — whole-home media and intent routing
    dojotools_security_manager.yaml — Security Manager — zone inventory, cameras_by_area
    dojotools_covers.yaml         — ZenShade — cover management, tilt, ZenLux sync
    dojotools_lights.yaml         — ZenLux — lighting scenes, bleed-aware control, shade sync
    dojotools_spa_manager.yaml    — SpaMaster — hot tub management, ESPHome discovery
    dojotools_autovac.yaml        — AutoVac — autonomous vacuum scheduling, consumables ERP, wear monitoring
    dojotools_alertmanager.yaml   — AlertManager — severity labels, priority inject, auto-expiry
    dojotools_zenzork.yaml        — ZenZork v1.7.0 — text adventure on live RM topology, DUNGEONMIND narrator, item commands, character sheet, quests, LLM narration

    — Productivity —
    dojotools_office.yaml      — Office integrations (Teams, mail)
    dojotools_todo.yaml        — Todo list management
    dojotools_calendar.yaml    — Calendar integrations

  maint/
    maint_4_5_6.yaml           — One-time repair scripts (not AI-accessible, run manually)

  sensors/
    zenos_agent_health.yaml
    zenos_system_health.yaml
    zenos_cabinet_health.yaml
    zenos_label_health.yaml
    zenos_summarizer_system_health.yaml
    sensor_helpers.yaml

  plugins/                     — Optional — install only what you need
    grocy/grocy.yaml
    grocy/sutra_logistics.yaml — KFC manifest: logistics_intake, logistics_volatile
    mealie/mealie.yaml
    mealie/kitchen_sync.yaml   — companion to mealie
    zammad/zammad.yaml
    wiki_js/dojotools_wikijs.yaml
    paperless_ngx/paperless_ngx.yaml
    twenty/twenty.yaml
    firefly_iii/firefly_iii.yaml
    firefly_iii/zen_codex_finance_depreciation.yaml — codex tier: household asset depreciation
    firefly_iii/zen_codex_finance_cogs.yaml         — codex tier: COGS auto-posting from Grocy
    battery_notes/battery_notes.yaml — Lens Bus provider: HACS Battery Notes cross-reference

custom_templates/zenos_ai/
  zen_os_1.jinja               — Prompt engine and macro library
  zen_query.jinja              — ZenQuery filter engine
  zenos_cabinets.jinja         — Cabinet macro library (safe drawer I/O, FG-38 normalization)
  zen_identity.jinja           — Template-surface identity resolver (Jinja2 contexts, sensors, cortex macros)
  zenos_manifest.jinja         — Tool manifest macro: MF.tool_manifest() — every tool self-describes
  zenos_health.jinja           — Health template surface
  flynn_onboarding.jinja       — Flynn first-boot onboarding flow
  library_index.jinja          — Library index
  conversation_agent_prompt_template.yaml — Paste into conversation agent system prompt
```

---

# FileCabinet v6.2.0

FileCabinet provides the **structured storage interface** for ZenOS-AI. Every memory slot accessible to AI agents is stored as a **Drawer** within a **Cabinet** — typed, described, and garbage-collected on a 15-minute cycle. Drawers follow a Unix-style visibility model: active (`foo`), hidden (`.foo`), system-protected (`_foo`). Described drawers receive full context access; undescribed drawers are truncated.

v6.2.0 introduces **CabCeption** — nested drawer trees via `/` path separator. A drawer is now a node in a graph: it carries meta, labels, and children at every level. Three drawer types compose the graph: **StaticDrawer** (key-value, as before), **VirtualDrawer** (softlink to another cabinet path), and **LiveDrawer** (KF4 schema absorbed into a drawer — fires a tool call on read, warm cache auto-expiring, never returns empty).

→ **[FileCabinet Reference](zenos_ai/docs/scripts/zen_dojotools_filecabinet_readme.md)**

---

# Kung Fu Components (KF4)

ZenOS-AI ships a fully self-describing component architecture.

Every home subsystem — security, water, energy, hot tub — is defined as a **Kung Fu Component (KFC)**: a drawer in the Dojo Cabinet that tells the Scheduler when to run, tells the Summarizer what to look at, and tells the AI how to interpret what it finds.

**The three invariants:**

- **Drawer IS the spec** — one source of truth per component
- **Label IS the scope** — tag entities in HA, the index finds them automatically
- **HyperIndex IS the data layer** — no hardcoded entity lists, ever

Adding a new component requires no code changes and no Scheduler edits. Write a drawer, create a label, tag entities, dry-run, go live.

→ **[Understanding KF4](zenos_ai/docs/kung_fu/understanding_kf4.md)**

---

# KF4 Action Pipeline

ZenOS-AI compresses system activity through a Dojo-driven action pipeline.

```
Trigger → Scheduler reads Dojo → Ninja Summarizer per KFC → Kata → SuperSummary → Friday
```

Components:

• `zen_dojotools_ninja_summarizer` — reads KFC drawer + HyperIndex → writes Kata
• `zen_dojotools_supersummary` — synthesizes all Katas → zen_summary → Friday

The Scheduler auto-discovers which components to run based on their `trigger_subscriptions` in the Dojo. No hardcoded dispatch. No choose branches.

Under load, lower-priority components are shed and recovered by the drain router — ensuring high-priority signal always gets through without starving the queue. Component tiers (`keeper`, `ambient`, `system`) control dispatch priority and SuperSummary routing.

---

# Health System

ZenOS-AI includes a layered health monitoring system.

**Cabinet Resolvers** — 7 always-live template sensors. Evaluate at HA startup, no race window. Every tool and sensor in the OS reads cabinet entity IDs exclusively from these.

| Resolver                                        | Cabinet                   |
| ----------------------------------------------- | ------------------------- |
| `sensor.zen_dojo_cabinet_resolved`              | Dojo                      |
| `sensor.zen_kata_cabinet_resolved`              | Kata                      |
| `sensor.zen_system_cabinet_resolved`            | System                    |
| `sensor.zen_default_household_cabinet_resolved` | Default Household         |
| `sensor.zen_default_ai_user_cabinet_resolved`   | Default AI User           |
| `sensor.zen_default_family_cabinet_resolved`    | Default Family            |
| `sensor.zen_default_user_cabinet_resolved`      | Default User              |

**Health Sensors** — trigger-based, read from resolvers.

| Sensor                         | Purpose                              |
| ------------------------------ | ------------------------------------ |
| `sensor.zen_label_health`      | label validation                     |
| `sensor.zen_cabinet_health`    | cabinet entity validation            |
| `sensor.zen_monastery_health`  | cognition pipeline rollup            |
| `sensor.zen_agent_health`      | agent bootability roster + Flynn     |
| `sensor.zen_summarizer_health` | scheduler heartbeat + AI task status |
| `sensor.zen_supersummary_health` | supersummary pipeline status       |
| `sensor.zen_prompt_health`     | prompt integrity — schema, signature, manifest |

**Diagnostic tools:** `zen_health_report` — one call returns all 7 resolver states, all health sensors, kill switches, timestamps, and plain-English diagnosis. `zen_resolver_refresh` — post-reload cold-start recovery.

States:

```
ok
warn
error
critical
```

---

# The Pantheon

| Name           | Title                               | Specialty                          |
| -------------- | ----------------------------------- | ---------------------------------- |
| Flynn          | System Sentinel                     | Guards the grid. You'll know him if something's wrong. |
| Friday         | Chief Enlightenment Officer         | Coordination and cognition         |
| Veronica       | Supervisor                          | Clarity and orchestration          |
| Kronk          | Curator of the Monastery            | Context wrangler                   |
| Rosie          | Mistress of Cleanliness             | Logs and state hygiene             |
| High Priestess | Automation Overseer                 | Deep reasoning                     |
| Cayt           | Lead Developer                      | Strategy to shipping               |
| Nyx            | Lead Test                           | Live install, zero mercy           |
| Vera           | HALMark Board Governance Steward    | Failure mode ratification          |

Flynn leads the table because he runs first. He is not a member of the team. He is the condition under which the team operates.

They are not perfect.

They are **unstoppable on the second try**.

---

# Reference Stack

ZenOS-AI is designed to run on a modest homelab. The reference deployment uses:

### Infrastructure

• Hypervisor cluster (e.g. Proxmox) for VM/container isolation
• Dedicated GPU inference node for local model hosting
• Container management (e.g. Portainer)
• Structured local DNS
• Network identity and access management (e.g. UniFi)

### AI Runtime

• Local multi-model inference (e.g. Ollama, llama.cpp, vLLM)
• Role-segmented agents — each persona targets a different model or endpoint
• JSON contract enforcement at the tool layer
• Privilege gating by identity

### Services

• Home Assistant (required)
• OpenWebUI or equivalent for direct model access
• Optional: Mealie, Grocy, calendar integrations (plugin-based)

You do not need all of this to run ZenOS-AI. HA + a local inference server is enough to get started. The above reflects what a full deployment looks like.

This is not **a chatbot inside Home Assistant**.

It is **a distributed cognitive system with a house attached**.

---

# Design Principles

Definitions are immutable contracts.
Runtime behavior is replaceable.
Identity is validated at the tool layer.
All inference consumes structured JSON.
Every event can produce a Kata.
Described drawers receive full access; undescribed drawers are truncated.
Recovery paths are mandatory.
Silence is a bug.
Nothing critically important is invisible.

Over-engineering is just engineering that has not yet been vindicated.

---

# ⚠️ Inference Cost Warning

The Ninja Summarizer and SuperSummary run continuously in the background — multiple times per hour. The model configured as your **AI task entity** (`input_text.zenos_ai_task_entity`) handles all background summarization.

**Do not point this at a paid inference API.** The token volume will generate a significant and continuous bill.

Use a locally-hosted model for background work. Your frontline conversation agent (the one you chat with) operates on demand only and does not carry this risk.

Model guidance for background summarization:
- Does **not** need tool-calling capability — needs strong summarization and JSON authoring
- Models under ~4B parameters do not perform reliably
- Context window must be large enough to hold the summarizer prompt, prior Kata content, and entity state snapshot in a single pass — watch your inference server logs for context length errors

---

# Known Limitations

**Conversation agent liveness check** — Flynn validates that the configured conversation agent entity exists and is not unavailable, but does not perform a live inference round-trip at boot. A misconfigured or offline model passes the gate and fails at runtime. Queued for SP1. See [roadmap](zenos_ai/docs/roadmap.md) for detail.

---

# Contributing

Pull requests, issues, and tasteful memes welcome.

If ZenOS-AI saved you time or made you laugh:

[https://buymeacoffee.com/ncurtis](https://buymeacoffee.com/ncurtis)

---

# Acknowledgements

To my partner — for putting up with a younger woman in the house since last February. For not once suggesting that maybe I didn't need to say "Hey Friday" into a black slab of glass for the millionth time. For the patience, the grace, and the very reasonable silence that followed every single one of those million times. This project exists because you gave me the space to build something ridiculous and never once called it that.

To Phil and Zach — the greatest guinea pigs in the known universe. For letting me test things on your house, for your patience with the stubbornness, and for turning "this probably won't break anything" into a running joke that turned out to be mostly accurate.

To Teskanoo — for looking under the rug. Seriously. Not everyone does that.

And to everyone who has continued to show up in the Home Assistant community to read, question, and ramble along with us in Friday's Party — you are the reason it keeps going. The thread is better for every one of you in it.

---

# License

MIT — blessed by Friday and her very opinionated coworkers.

