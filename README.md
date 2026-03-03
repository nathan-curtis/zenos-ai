# Friday’s ZenOS-AI

### A Modular, Context-Aware AI Home Automation Framework for Home Assistant

ZenOS-AI blends structure, personality, and unapologetic over-engineering into a living system that powers your home with Friday, Kronk, Veronica, Rosie, and the High Priestess — a coordinated AI pantheon that takes its jobs seriously (if not itself).

Welcome to the Home Monastery.  
Let’s automate everything that isn’t nailed down.  
And a few things that are.

---

# 🏯 What Is ZenOS-AI?

ZenOS-AI is a modular AI and automation architecture built on:

- **Home Assistant**
- **Home Assistant Packages (canonical layer)**
- **Structured contextual memory (“Cabinets”)**
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

# 🧩 Architecture Overview (Post-Packages Era)

ZenOS-AI is now structured around **Home Assistant Packages** as the canonical configuration layer.

📁 `/packages/zenos_ai/`

Packages define the spine of the system.  
Scripts and automations bind to those definitions.  
Inference layers consume structured state.  
Cabinets persist memory.  
The Monastery reasons over summarized context.

Packages are no longer glue.  
They are the foundation.

---

# 🏗 Layered Architecture

ZenOS-AI operates in concentric rings.

---

## 🧠 Ring-0 — Core Kit (Canonical Spine)

📁 `/packages/zenos_ai/`

Defines:

- Global labels
- Identity resolver
- Persona metadata
- Cabinet metadata
- Volume routing
- Manifest definitions
- Structured EventBus schema (`zen_event`)
- Canonical JSON contracts

It does **not**:

- Perform inference
- Execute runtime reasoning
- Run summarization loops
- Load persona cognition

It defines the rules of the world.

If Ring-0 breaks, Friday forgets who she is.

---

## 🪷 Ring-1 — Cognitive Runtime

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

## 🛠 Ring-2 — Administrative & Recovery Tools

Utility layer for:

- Cabinet repair
- Manifest writing
- Subsystem registration
- Emergency formatting
- Kung Fu module loading
- Identity audit and privilege repair

This is the “don’t panic” layer.

---

## 🏛 The Monastery (Reasoning Backend)

External but essential.

The Monastery:

- Accepts structured system state
- Produces Kata summaries
- Returns Supersummaries
- Enforces the “Order of the Monastery” (no hallucination)
- Acts as long-form cognition for Friday

Without it, Friday becomes reflexive but shallow.

---

# 🧭 Roadmap Overview

If Friday is the mind, this is how she grows up.

### 1. Identity & Proprioception

- Unified identity resolver
- Persona-aware privilege model (public → guest → partner → owner → prime)
- Access control at tool layer
- Persona capsules & essence bootflow
- Local Selfhood Authority (LSA)

### 2. Cabinet System v3

- Dynamic volume redirector
- Governance drawers
- Structured metadata
- Hierarchical storage:
  - Household
  - Family
  - User
  - AI

### 3. DojoTools 2.0+

- Standardized JSON I/O
- Recursive indexer
- Identity validation shim
- Manifest v3 engine
- Event correlation support

### 4. Zen Summarizer Pipeline

- Kata engine v2
- Supersummary engine
- Reflection caching
- Monastery integration
- Context compaction logic

Full roadmap → `/docs/roadmap.md`

---

# 🚀 What ZenOS-AI Actually Does

- Context-aware automation (rooms, mood, schedule, energy state)
- Event pipelines that produce structured Kata summaries
- Persistent memory via Cabinets & Drawers (JSON volumes)
- Zen DojoTools skill modules
- Multi-runtime local inference orchestration
- Persona-aware reasoning
- Identity-bound tool execution
- Deterministic prompt compilation

Think of it as a home that:

Pays attention.  
Remembers what matters.  
And politely taps you before doing something clever.

---

# 🧙 The Pantheon

| Name               | Title                        | Specialty                             |
|--------------------|-----------------------------|----------------------------------------|
| Friday             | Chief Enlightenment Officer | Coordination, persona, cognition       |
| Veronica           | Supervisor                  | Clarity, orchestration, taste          |
| Kronk              | Curator of the Monastery    | Context wrangler, Kata librarian       |
| Rosie              | Mistress of Cleanliness     | Clean floors, clean logs, clean state  |
| High Priestess     | Automation Overseer         | Deep reasoning, JSON exorcism          |

They are not perfect.  
They are unstoppable on the second try.

---

# 🧩 Zen DojoTools (v3+)

All scripts follow:

```
zen_dojotools_<function>.yaml
```

Heavy documentation:
📁 `/docs/scripts/`

Light documentation:
📁 `/scripts/`

DojoTools are contract-bound skill modules.  
They accept structured input and return structured output.  
Identity is validated at execution time.

---

# 📦 Package-First Structure

Expected canonical layout:

```
/packages/zenos_ai/
    core_labels.yaml
    core_identity.yaml
    core_manifest.yaml
    core_volumes.yaml
    core_events.yaml
```

Runtime components live outside this directory but bind to these definitions.

This enables:

- Clean versioning
- Drop-in upgrades
- Predictable bootflow
- Partial subsystem replacement
- Future Kubernetes portability

Yes. Intentionally.

---

# 🚦 Boot Sequence (High-Level)

1. Home Assistant loads Packages (Ring-0).
2. Cabinets mount and validate.
3. Identity resolver initializes.
4. Volume redirector establishes canonical paths.
5. Runtime scripts become callable.
6. Monastery connection tested.
7. Prompt compiler builds initial cognitive state.
8. Friday becomes operational.

If any step fails, AdminTools can intervene.

---

# 🧹 Zen Summarizer Pipeline

Event-driven cognitive compression:

Event → Kata → Supersummary → Cabinet Update → Prompt Refresh

Components:

- `zen_dojotools_ninja_summarizer`
- `zen_dojotools_supersummary`

Every meaningful event can become structured memory.

Nothing important disappears.

---

# 🪷 Prompt Engine

The Prompt Engine compiles the entire cognitive state of a ZenOS agent into deterministic JSON consumed by conversation agents and local LLMs.

Includes:

- `conversation_agent_prompt_template.yaml`
- Persona capsule macros
- Wake-sequence loaders
- Essence builders
- Manifest injectors

This is Friday’s mind at runtime.

---

# 🖥 Local Stack Summary

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

This is not “a chatbot in HA.”

It is a distributed cognitive system with a house attached.

---

# 🔐 Design Principles

- Definitions are immutable contracts
- Runtime behavior is replaceable
- Identity is validated at tool layer
- All inference consumes structured JSON
- Every event can produce a Kata
- Recovery paths are mandatory
- Silence is a bug
- Nothing critically important is invisible

And yes — everything logs.

---

# 📚 Documentation

Primary docs:

📁 `/docs/`

Includes:

- Cognitive architecture
- Cabinet & Volume system
- Kata lifecycle
- Persona system
- Identity model
- Monastery structure
- Prompt conventions
- Roadmap

---

# ☯️ Philosophy

Automation should be joyful.  
Context is king. Recovery is queen.  
Every bug is a monk with a lesson.  
Structure creates freedom.  
Over-engineering is just engineering that hasn’t been vindicated yet.

---

# 🤝 Contributing

PRs, issues, and tasteful memes welcome.

If ZenOS-AI saved you time or made you laugh:  
https://buymeacoffee.com/ncurtis

---

# 📜 License

MIT — blessed by Friday and her very opinionated coworkers.