# Friday's ZenOS-AI

### A Modular, Context-Aware AI Home Automation Framework for Home Assistant

ZenOS-AI blends structure, personality, and unapologetic over-engineering into a living system that powers your home with Friday, Kronk, Veronica, Rosie, and the High Priestess — a coordinated AI pantheon that takes its jobs seriously (if not itself).

Welcome to the Home Monastery.
Let's automate everything that isn't nailed down.
And a few things that are.

**Current version: 4.0.0 RC2**

---

# What Is ZenOS-AI?

ZenOS-AI is a modular AI and automation architecture built on:

- **Home Assistant**
- **Home Assistant Packages (canonical configuration layer)**
- **Structured contextual memory ("Cabinets" and "Drawers")**
- **Event-driven Kata summaries**
- **Local and distributed inference engines**
- **A multi-persona AI team**

It creates a privacy-first, locally hosted intelligent home system that:

- Reasons about context
- Stores long-term structured memory
- Automates reflexively
- Summarizes itself
- Enforces identity and privilege boundaries
- Speaks like an intelligent adult
- Occasionally sighs at your poor life choices

The system works because it is modular.
It is delightful because it is chaotic-good.

---

# Architecture Overview

ZenOS-AI is structured entirely around **Home Assistant Packages** as the canonical configuration layer.

Everything lives under:

```
packages/zenos_ai/
```

Packages define the spine of the system.
DojoTools scripts provide the runtime behavior.
Cabinets persist memory.
The Monastery reasons over summarized context.
Flynn keeps the lights on.

---

# Layered Architecture

ZenOS-AI operates in concentric rings.

---

## Ring-0 — Core Kit (Canonical Spine)

`packages/zenos_ai/`

Defines:

- Global labels
- Identity resolver
- Persona metadata
- Cabinet metadata and volume routing
- Manifest definitions
- Structured EventBus schema (`zen_event`)
- Canonical JSON contracts
- Health sensors

It does **not**:

- Perform inference
- Execute runtime reasoning
- Run summarization loops
- Load persona cognition

It defines the rules of the world.

If Ring-0 breaks, Friday forgets who she is.

---

## Ring-1 — Cognitive Runtime

This layer binds behavior to definitions.

Includes:

- Zen DojoTools scripts
- Summarizer pipeline
- Prompt compiler
- Persona capsules
- Conversation agent interface
- Monastery inference integration

This is where Friday thinks.

Requires:

- Core Kit (Ring-0)
- FileCabinet active
- Custom Jinja templates loaded
- Monastery reachable

---

## Ring-2 — Administrative & Recovery Tools

Utility layer for:

- Cabinet repair
- Manifest writing
- Subsystem registration
- Emergency formatting
- Kung Fu module loading
- Identity audit and privilege repair

This is the "don't panic" layer.

---

## The Monastery (Reasoning Backend)

External but essential.

The Monastery:

- Accepts structured system state
- Produces Kata summaries
- Returns Supersummaries
- Enforces the "Order of the Monastery" (no hallucination)
- Acts as long-form cognition for Friday

Without it, Friday becomes reflexive but shallow.

---

# Installation

ZenOS-AI is a drop-in Home Assistant package collection.

## Requirements

- Home Assistant (2024.x+)
- `homeassistant.customize` support
- Custom Jinja templates (place contents of `custom_templates/` into your HA config)
- A conversation agent configured and reachable

## Steps

1. Copy `packages/zenos_ai/` into your Home Assistant config directory under `packages/`
2. Copy `custom_templates/` contents into your HA `custom_templates/` directory
3. Add to your `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

4. Restart Home Assistant
5. Run `script.zen_flynn` to initialize labels, cabinets, and health state
6. Configure `input_text.zenos_conversation_agent` to point to your conversation agent entity
7. Check `sensor.zen_agent_health` — it should reach `ok`

Plugins under `packages/zenos_ai/plugins/` are optional. Drop in what you need.

---

# Package Structure

```
packages/zenos_ai/
  zenos_cabinets.yaml          — Cabinet definitions and volume routing
  flynn.yaml                   — Flynn bootstrap and recovery engine

  dojotools/
    dojotools_filecabinet.yaml — FileCabinet v4 — typed drawer I/O
    dojotools_core.yaml        — Core ops + FileCabinet GC
    dojotools_scheduler.yaml   — Scheduled automation triggers
    dojotools_admintools.yaml  — Cabinet repair, manifest write, KFC loader
    dojotools_manifest.yaml    — Manifest engine
    dojotools_index.yaml       — Index and query tools
    dojotools_identity.yaml    — Identity resolver and privilege model
    dojotools_labels.yaml      — Label inspection and management
    dojotools_library.yaml     — Library tools
    dojotools_history.yaml     — History management
    dojotools_summarizers.yaml — Kata and Supersummary engines
    dojotools_systemtools.yaml — System tools and event emitter
    dojotools_utilities.yaml   — General utilities
    dojotools_spa_manager.yaml — SPA / single-page-app manager
    dojotools_office.yaml      — Office integrations (Teams, mail, todo, calendar)

  sensors/
    zenos_agent_health.yaml    — Agent bootability roster and RAG rollup
    zenos_system_health.yaml   — System subsystem health sensors
    zenos_cabinet_health.yaml  — Cabinet validation and slot health
    zenos_label_health.yaml    — Label set validation
    zenos_summarizer_system_health.yaml
    sensor_helpers.yaml

  plugins/
    grocy/grocy.yaml           — Grocy product and inventory operations
    mealie/mealie.yaml         — Mealie recipe and food operations
    kitchen_sync/kitchen_sync.yaml — Mealie <> Grocy food sync
    firefly_iii/firefly_iii.yaml
    wiki_js/wiki_js_rest_commands.yaml

  room_manager/room_manager.yaml
  zen_image_generator.yaml

custom_templates/
  library_index.jinja          — Cabinet index resolution macros
  (additional macro libraries)
```

---

# FileCabinet v4

FileCabinet is the core structured storage interface. Every AI-accessible memory slot is a **Drawer** inside a **Cabinet**.

## Drawer Structure

```json
{
  "value": "<any>",
  "timestamp": "ISO-8601",
  "meta": {
    "entity_labels": ["label1"],
    "description": "Human-readable purpose of this drawer",
    "expires_after": "2026-06-01T00:00:00",
    "no_autoexpire": false,
    "no_autorecycle": false,
    "acl": {}
  }
}
```

All `meta` fields are optional. Only fields with meaningful values are stored.

## Key Behaviors

**Write** — stores value with optional metadata:

```yaml
action: script.zen_dojotools_filecabinet
data:
  mode: run
  case: write
  entity_id: sensor.my_cabinet
  key: my_drawer
  value: "some content"
  description: "What this is for"
  expires_after: "2026-06-01T00:00:00"
```

**Read (direct)** — returns full drawer content if described; truncates to 64 chars if not:

```yaml
case: read
entity_id: sensor.my_cabinet
key: my_drawer
```

Drawers without descriptions return truncated values. Add descriptions. This is intentional.

**Read (label-mode)** — returns blurbs map across all drawers in a cabinet. Token-efficient scan view.

**Inspect** — returns drawer inventory with metadata.

## Drawer Lifecycle

Drawers follow a Unix-style visibility model:

| State | Key pattern | Meaning |
|-------|-------------|---------|
| Active | `foo` | Normal readable drawer |
| Hidden | `.foo` | Archived, not visible in label reads |
| System | `_foo` | Protected, never touched by GC |

`AI_Cabinet_VolumeInfo` is always protected in all GC paths.

## Expiry and GC

Set `expires_after` on any drawer to time-bomb it. The FileCabinet GC runs every 15 minutes and implements the full lifecycle:

- **Expire** → visible `foo` with past `expires_after` → renamed to `.foo` (hidden)
- **Delete** → hidden `.foo` with past `expires_after` → deleted
- **Recycle** → manual; hides drawer and sets expiry to now+24h
- **Unhide** → manual; restores `.foo` to `foo`

Drawers with `no_autoexpire: true` are skipped by the GC expiry path.
Drawers with `no_autorecycle: true` are skipped by the recycle path.
System drawers (`_` prefix) are never touched.

Manual GC trigger:

```yaml
event_type: zen_event
event_data:
  event:
    kind: gc_force
```

---

# Zen DojoTools

All DojoTools scripts follow the contract:

```
zen_dojotools_<module>
```

They accept structured input and return structured JSON output.
Identity is validated at execution time.
Every tool documents its cases via `case: help`.

## Tool Inventory

| Tool | Cases / Purpose |
|------|----------------|
| `filecabinet` | read, write, inspect, delete, list, label-read |
| `filecabinet_gc` | gc, recycle, hide, unhide — drawer lifecycle |
| `manifest` | read, write, inspect |
| `index` | query, list, inspect cabinets |
| `identity` | resolve, validate, audit |
| `labels` | list, inspect, health |
| `library` | load, query, resolve |
| `history` | read, write, trim |
| `summarizers` | kata, supersummary, ninja |
| `admintools` | repair, format, write-kfc, audit |
| `systemtools` | system state, event emit |
| `utilities` | general helpers |
| `spa_manager` | SPA session management |
| `office` | Teams, mail, todo, calendar |
| `grocy_helper` | product lookup, inventory |
| `mealie_helper` | foods, recipes |

Documentation: `case: help` on any tool returns its full case list and field schema.

---

# Boot Sequence

1. Home Assistant loads Packages (Ring-0)
2. Cabinets mount and validate
3. Identity resolver initializes
4. Volume redirector establishes canonical paths
5. Runtime scripts become callable
6. Monastery connection tested
7. Prompt compiler builds initial cognitive state
8. Friday becomes operational

Check `sensor.zen_agent_health` for live bootability status.
If any gate fails, the roster attribute shows which gate is blocking and why.

If any step fails, AdminTools can intervene.

---

# Summarizer Pipeline

Event-driven cognitive compression:

```
Event → Kata → Supersummary → Cabinet Update → Prompt Refresh
```

Components:

- `zen_dojotools_ninja_summarizer` — lightweight event-to-kata
- `zen_dojotools_supersummary` — kata-to-supersummary consolidation

Every meaningful event can become structured memory.

The scheduler drives periodic summarization. The Monastery drives reflection.

Nothing important disappears.

---

# Health System

ZenOS-AI has a tiered health rollup:

| Sensor | What it measures |
|--------|-----------------|
| `sensor.zen_label_health` | Required labels present and valid |
| `sensor.zen_cabinet_health` | Cabinet entities exist and readable |
| `sensor.zen_monastery_health` | Monastery reachability |
| `sensor.zen_flynn_health` | Rollup of all infra gates |
| `sensor.zen_agent_health` | Agent bootability roster + top-level RAG |

States: `ok` / `warn` / `error` / `critical`

`sensor.zen_agent_health.attributes.roster` — per-agent gate-by-gate bootability detail.

---

# The Pantheon

| Name | Title | Specialty |
|------|-------|-----------|
| Friday | Chief Enlightenment Officer | Coordination, persona, cognition |
| Veronica | Supervisor | Clarity, orchestration, taste |
| Kronk | Curator of the Monastery | Context wrangler, Kata librarian |
| Rosie | Mistress of Cleanliness | Clean floors, clean logs, clean state |
| High Priestess | Automation Overseer | Deep reasoning, JSON exorcism |

They are not perfect.
They are unstoppable on the second try.

---

# Local Stack Summary

### Infrastructure

- Proxmox cluster
- GPU inference node
- Portainer (stack orchestration)
- n8n (agent delegation)
- Structured DNS
- UniFi Identity

### AI Runtime

- Multi-model inference (local + remote)
- Role-segmented agents
- JSON contract enforcement
- Tool-layer privilege gating

### Services

- Home Assistant
- Mealie
- Grocy
- OpenWebUI
- Inference backends

This is not "a chatbot in HA."

It is a distributed cognitive system with a house attached.

---

# Design Principles

- Definitions are immutable contracts
- Runtime behavior is replaceable
- Identity is validated at tool layer
- All inference consumes structured JSON
- Every event can produce a Kata
- Described drawers get full access; undescribed drawers get truncated access
- Recovery paths are mandatory
- Silence is a bug
- Nothing critically important is invisible

And yes — everything logs.

---

# Documentation

`/docs/` — Architecture, cabinet system, Kata lifecycle, persona model, identity model, Monastery structure, prompt conventions, roadmap

---

# Philosophy

Automation should be joyful.
Context is king. Recovery is queen.
Every bug is a monk with a lesson.
Structure creates freedom.
Over-engineering is just engineering that hasn't been vindicated yet.

---

# Contributing

PRs, issues, and tasteful memes welcome.

If ZenOS-AI saved you time or made you laugh:
https://buymeacoffee.com/ncurtis

---

# License

MIT — blessed by Friday and her very opinionated coworkers.
