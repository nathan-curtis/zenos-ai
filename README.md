# Friday's ZenOS-AI

### A Modular, Context-Aware AI Home Automation Framework for Home Assistant

ZenOS-AI blends structure, personality, and unapologetic over-engineering into a living system that powers your home with **Friday, Kronk, Veronica, Rosie, and the High Priestess** — a coordinated AI pantheon that takes its jobs seriously (even if it doesn't always take itself seriously).

Welcome to the **Home Monastery**.

Let's automate everything that isn't nailed down.

And a few things that are.

**Current version: 4.0.0 RC2**

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
Flynn keeps the lights on.

---

# Layered Architecture

ZenOS-AI operates in **concentric rings**, separating definition, runtime cognition, and administrative tooling.

---

## Ring-0 — Core Kit (Canonical Spine)

Location:

```

packages/zenos_ai/

```

This layer defines the **rules of the world**.

It establishes:

• global labels  
• identity resolver schema  
• persona metadata  
• cabinet metadata and volume routing  
• manifest definitions  
• structured EventBus schema (`zen_event`)  
• canonical JSON contracts  
• health sensors

Ring-0 **does not perform runtime behavior**.

It does not:

• run inference  
• execute reasoning  
• perform summarization  
• load persona cognition

It defines the **structure and contracts** that the rest of the system depends on.

If Ring-0 breaks, **Friday forgets who she is**.

---

## Ring-1 — Cognitive Runtime

This layer binds **behavior to the definitions** established by Ring-0.

Components include:

• Zen DojoTools scripts  
• the summarizer pipeline  
• prompt compilation  
• persona capsules  
• the conversation agent interface  
• Monastery inference integration

This is where **Friday thinks**.

Requirements:

• Ring-0 Core Kit  
• FileCabinet active  
• custom Jinja templates loaded  
• Monastery reachable

---

## Ring-2 — Administrative & Recovery Tools

The administrative layer exists for **maintenance, repair, and recovery**.

Functions include:

• cabinet repair  
• manifest writing  
• subsystem registration  
• emergency formatting  
• Kung Fu module loading  
• identity audit and privilege repair

This is the **“don’t panic” layer**.

When something goes wrong, Ring-2 is how the system fixes itself.

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

• Home Assistant (2024.x+)  
• `homeassistant.customize` enabled  
• custom Jinja templates (included)  
• a configured conversation agent

---

## Installation Steps

1. Copy

```

packages/zenos_ai/

```

into your Home Assistant configuration directory under:

```

packages/

```

2. Copy the contents of:

```

custom_templates/

```

into your Home Assistant:

```

custom_templates/

````

directory.

3. Add the following to `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
````

4. Restart Home Assistant.

5. Run the bootstrap script:

```
script.zen_flynn
```

6. Configure:

```
input_text.zenos_conversation_agent
```

to point to your conversation agent entity.

7. Verify system health:

```
sensor.zen_agent_health
```

The sensor should report `ok`.

Plugins located under:

```
packages/zenos_ai/plugins/
```

are optional. Install only the integrations you need.

---

# Package Structure

```
packages/zenos_ai/
  zenos_cabinets.yaml          — Cabinet definitions and volume routing
  flynn.yaml                   — Flynn bootstrap and recovery engine

  dojotools/
    dojotools_filecabinet.yaml — FileCabinet v4 — typed drawer I/O
    dojotools_core.yaml        — Core operations + FileCabinet GC
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
    dojotools_office.yaml      — Office integrations (Teams, mail, todo, calendar)

  sensors/
    zenos_agent_health.yaml
    zenos_system_health.yaml
    zenos_cabinet_health.yaml
    zenos_label_health.yaml
    zenos_summarizer_system_health.yaml
    sensor_helpers.yaml

  plugins/
    grocy/grocy.yaml
    mealie/mealie.yaml
    kitchen_sync/kitchen_sync.yaml
    calderaspas/calderaspas_spa_manager.yaml

  room_manager/room_manager.yaml
  zen_image_generator.yaml

custom_templates/zenos_ai/
  library_index.jinja
  zen_os_1rc.jinja
  zen_query_jinja
```

---

# FileCabinet v4

FileCabinet provides the **structured storage interface** for ZenOS-AI.

Every memory slot accessible to AI agents is stored as a **Drawer** within a **Cabinet**.

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

All `meta` fields are optional.

Only fields containing meaningful values are persisted.

---

## Drawer Lifecycle

Drawers follow a Unix-style visibility model:

| State  | Key Pattern | Meaning                |
| ------ | ----------- | ---------------------- |
| Active | `foo`       | Normal readable drawer |
| Hidden | `.foo`      | Archived drawer        |
| System | `_foo`      | Protected drawer       |

System drawers are never touched by garbage collection.

---

## Expiry and Garbage Collection

The FileCabinet garbage collector runs every **15 minutes**.

Lifecycle rules:

• Expired active drawers → hidden
• Hidden expired drawers → deleted
• Manual recycle → hide + expire in 24 hours
• Unhide → restore drawer visibility

Protected drawers (`_prefix`) are never modified automatically.

---

# Summarizer Pipeline

ZenOS-AI compresses system activity through an event-driven summarization pipeline.

```
Event → Kata → Supersummary → Cabinet Update → Prompt Refresh
```

Components:

• `zen_dojotools_ninja_summarizer` — event → kata
• `zen_dojotools_supersummary` — kata → supersummary

This allows the system to preserve context while maintaining token efficiency.

---

# Health System

ZenOS-AI includes a layered health monitoring system.

| Sensor                        | Purpose                   |
| ----------------------------- | ------------------------- |
| `sensor.zen_label_health`     | label validation          |
| `sensor.zen_cabinet_health`   | cabinet entity validation |
| `sensor.zen_monastery_health` | Monastery connectivity    |
| `sensor.zen_flynn_health`     | infrastructure rollup     |
| `sensor.zen_agent_health`     | agent bootability roster  |

States:

```
ok
warn
error
critical
```

---

# The Pantheon

| Name           | Title                       | Specialty                  |
| -------------- | --------------------------- | -------------------------- |
| Friday         | Chief Enlightenment Officer | Coordination and cognition |
| Veronica       | Supervisor                  | Clarity and orchestration  |
| Kronk          | Curator of the Monastery    | Context wrangler           |
| Rosie          | Mistress of Cleanliness     | Logs and state hygiene     |
| High Priestess | Automation Overseer         | Deep reasoning             |

They are not perfect.

They are **unstoppable on the second try**.

---

# Local Stack Summary

### Infrastructure

• Proxmox cluster
• GPU inference node
• Portainer
• n8n agent orchestration
• Structured DNS
• UniFi Identity

### AI Runtime

• multi-model inference
• role-segmented agents
• JSON contract enforcement
• tool-layer privilege gating

### Services

• Home Assistant
• Mealie
• Grocy
• OpenWebUI
• inference backends

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

# Contributing

Pull requests, issues, and tasteful memes welcome.

If ZenOS-AI saved you time or made you laugh:

[https://buymeacoffee.com/ncurtis](https://buymeacoffee.com/ncurtis)

---

# License

MIT — blessed by Friday and her very opinionated coworkers.

