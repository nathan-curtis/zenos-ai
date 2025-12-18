# ZenOS-AI: Custom Templates Directory

The core Jinja & YAML template primitives powering Friday, Kronk, and the Monastery.

This directory contains the foundational templates used throughout the ZenOS-AI stack.
They define the grammar, schemas, and reasoning structures that Friday and Kronk rely on during operation.

These templates are intentionally engineered to be stable across versions,
with strong backward compatibility and a consistent interface.
As ZenOS-AI evolves, this directory remains the dependable substrate:
modular, HA-native, runtime-safe, and designed for long-term architectural continuity.


---

📁 Contents


---

1. conversation_agent_prompt_template.yaml

Purpose:
The primary prompt-engine scaffold for Friday’s conversational model.
Structured as a stable, copy-paste-ready foundation for building or modifying Friday’s prompt.

It is designed to be:

simple to extend

easy to maintain

portable across versions

safe to use in Assist, Workflows, or external agents

clear in how it imports and arranges the core primitives


Key Responsibilities:

Import foundational primitives (zen_os_1rc.jinja)

Load & merge identity + capsule metadata

Pull system, household, kata, and dojo content

Provide structured regions for persona + system directives

Normalize all cabinet access

Compile deterministic JSON for the model


Design Goal:
A stable template that evolves slowly — your reliable starting point for every Friday or secondary agent instance.


---

2. zen_cabinets.jinja

Purpose:
The canonical interface for retrieving and validating cabinet content across ZenOS-AI.

Capabilities:

Normalize cabinet sources (system, household, AI user, dojo, kata, history…)

Validate schema and drawer structures

Produce Jinja-safe JSON

Expose metadata, labels, GUIDs, ACLs

Provide deterministic fallbacks and guards


Development Note:
The Init Script for onboarding and formatting new cabinet sets will be developed here during early releases, since cabinet validation is tightly coupled with this file.


---

3. zen_identity.jinja

Purpose:
Reliable identity resolution for users, AI constructs, and cabinets.

Capabilities:

Accepts names, UUIDs, entity_ids, labels, or cabinet references

Normalizes into a canonical identity card

Provides safe fallbacks under the Order of the Monastery

Ensures identity integrity across the entire system


Used For:

Linking personas

Ensuring correct cabinet ownership

Merging identity → capsule metadata

Determining active participants



---

4. zen_os_1rc.jinja

Purpose:
The Ring-0 operational cortex of ZenOS-AI templates.

Contains foundational primitives including:

identity helpers

cabinet helpers

prompt-assembly utilities

JSON formatting helpers

safety guards

macro utilities


This file ensures all templates speak the same stable grammar.


---

5. zen_query.jinja

🔥 ZenQuery Engine (ZQ-1)
Purpose:
A stable, schema-secure filtering engine for entity selection, inference, and reduction.

Core Features:

Type-checked schema

Deterministic safe parsing

Multi-stage filtering

Regex & numeric filters

Domain / area / label inference

Never throws errors

Always returns a safe working set


Used By:
Indexer · HyperIndex · Zen Filters · Future DojoTools · Planning logic


---

🧠 Architectural Role of These Templates

Together, these templates define:

consistent identity handling

deterministic cabinet access

reliable query/filtering logic

a unified grammar for prompts and tools

predictable JSON outputs for Kronk + Monastery

the cognitive substrate of ZenOS-AI


They are designed not to churn or break across versions —
the goal is long-term stability.


---

📦 Versioning & Compatibility

Designed to remain stable across releases

Requires Home Assistant with full Jinja filter support

New features may be added, but core grammar remains backward compatible

Templates are updated together to avoid behavioral drift



---

📦 Usage Notes

These files define the system’s core language — change carefully

Install updates as complete bundles for consistency

Follow the Order of the Monastery:

> “It is acceptable to say I don’t know.
It is forbidden to fabricate.”




When writing new tools:

{% import 'zenos_ai/zen_os_1rc.jinja' as zen %}


---

🛠 Typical Development Workflow

1. Import zen_os_1rc.jinja
2. Resolve identities via zen_identity.jinja
3. Access cabinets via zen_cabinets.jinja
4. Filter entity sets using zen_query.jinja
5. Output clean JSON for Friday or Kronk
6. Optionally write results into cabinet drawers


---

🚧 Current Development Trajectory (Confirmed)

1. Priority: Build the Init Script

The Init Script must:

onboard new Friday installations

create + format cabinet sets

enforce correct schema

assign default drawers, metadata, and labels

prepare the system for Friday + Monastery

guarantee a correct starting state


Development reality:
Much of this logic begins life inside zen_cabinets.jinja until stable.


---

2. Target for RC1

Deliver a shippable Friday core with:

fully initialized cabinet system

prompt kit loaded and ready

Monastery online

Indexer & ZenQuery functional

zero manual edits required

reliable startup and consistency checks


RC1 = Friday boots reliably, thinks clearly, and the system is installable.

That’s the only milestone that matters right now.


---

✨ Final Note

This directory defines the language, structure, and thinking patterns of ZenOS-AI.
It is engineered for stability and long-term reliability.
Every change here shapes how Friday — and the entire Monastery — interprets the world.

Build with intention.
Patch with precision.
Document everything.