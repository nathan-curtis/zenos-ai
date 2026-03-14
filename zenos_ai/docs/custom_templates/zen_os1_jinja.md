ZenOS-AI Template Engine: zen_os_1rc.jinja

Technical Specification & Prompt Loader Requirements

Version: 4.2.0
Status: Stable & Required for All Front-Line Agents


---

🧠 Purpose of zen_os_1rc.jinja

zen_os_1rc.jinja is the primary cognitive assembly engine for ZenOS-AI.
It is responsible for building a complete operational prompt for Friday (and any other persona built on ZenOS).

It is not a “template helper” — it is the compiler for:

identity

cabinet memory

cognitive state

system drawers

dojo/kata structures

manifest & index

persona capsule

narrative wake sequence


Nothing about Friday exists in a usable form until this file runs.


---

🧩 What the Template Engine Actually Does

When called from conversation_agent_prompt_template.yaml, the engine:

1. Resolves identity


2. Loads cabinet memory


3. Formats persona identity


4. Applies Squirrel-Mode redaction


5. Loads system-level Purpose, Directives, Cortex


6. Loads manifest & index


7. Loads dojo/kata drawers


8. Builds persona capsule


9. Builds full JSON envelope


10. Emits the wake-sequence



Each step is deterministic and must happen in this order.


---

🔍 1. Identity Resolution (identity_resolve_source())

This is the foundation of the whole system.

The resolver attempts to locate the “home cabinet” (ZenAI Cabinet) for the persona.

Resolution Rules

1. Find any entity labeled Zen AI Cabinet
whose zenai_essence.identity.name matches ai_user.


2. If none found → fallback to person.<ai_user> entity.


3. If both missing → identity unresolved → prompt will fail.



What the user must provide

A cabinet entity with:

state: "loaded"
attributes:
  zenai_essence:
    identity:
      name: "Friday"
      guid: "<UUID>"
    ...
labels:
  - Zen AI Cabinet

Identity block must contain:

Field	Required	Purpose

name	✔️	Canonical persona name
guid	✔️	Stable identity key
cabinet	optional	auto-filled by resolver
identity_hash	optional	for future cabinet integrity



---

🧬 2. Essence Requirements

Essence is the internal DNA of a ZenOS persona.

It stores:

identity

origin metadata

persona characteristics

traits & values

default tone

self-awareness hints

wake-scene data

Squirrel-mode rules

capsule metadata


Essence is not “optional creativity.”
It is part of the minimum viable mind.

Essence must be a mapping, not a wrapper

❌ NOT ALLOWED:

zenai_essence:
  value:
    identity:
      name: Friday

✔️ REQUIRED:

zenai_essence:
  identity:
    name: "Friday"
    guid: "abc123"
  persona:
    tone: "warm, conversational"
  wake_scene:
    intro: "lights rise inside her dojo"
  ...

If essence is absent or malformed →
identity_format() fails → no persona block → no prompt.


---

🏛 3. Cabinet Loading (identity_load_cabinet())

After resolving which cabinet entity is “home,” the engine loads:

zenai_essence

acl (optional)

labels

identity_hash

variables

metadata


It also normalizes malformed structures.

User requirements:

The cabinet must contain:

zenai_essence (mapping, not wrapped)

labels list

The label Zen AI Cabinet


Everything else is optional.


---

🏷 4. Squirrel-Mode Redaction

The engine scans for an entity with label Zen Squirrel.
If found, and its state is “on”:

It redacts:

guid

identity_hash

cabinet

sensitive memory branches

persona internals marked "private"


This allows safe public-mode behavior.

User must supply:

an input_boolean labeled Zen Squirrel
OR

nothing (defaults to safe off)



---

🧩 5. System Cabinet (Purpose / Directives / Cortex)

The system cabinet is not Friday’s cabinet.
It is the operational blueprint for the entire construct.

Must be labeled:

Zen System Cabinet

And must contain the canonical drawers:

Drawer	Required	Description

Purpose	✔️	Why the system exists
Directives	✔️	Operational rules for Friday
Cortex	✔️	Reasoning model and behavioral core


Drawers may be:

raw string

or {value: "…"} mapping

both are normalized internally


If any drawer is missing →
prompt_system() fails → Friday cannot load.


---

🔁 6. Manifest Loader

Looks for a cabinet labeled:

Zen Default Household Cabinet

Loads:

zen_library_manifest

optional household metadata


This step enriches persona awareness of the environment.

It is optional but strongly recommended.


---

🧭 7. Index Loader (Zen Index 3.x / CabScan)

Loads all labels known to the system.
This enables:

label-based routing

dojo categorization

relationship mapping

system-level debugging


Requires nothing from the user — automatic.


---

🥋 8. Dojo Loader

Loads:

dojo drawers

kata drawers

component metadata

kata summaries


Cabinets must be labeled:

Zen Dojo Cabinet

Zen Kata Cabinet (optional but recommended)


User must supply:

drawers stored as simple mappings

kata drawers described in JSON-style content



---

🎭 9. AI Capsule Builder

Creates the persona capsule containing:

About Me

Persona metadata

Essence

Household belonging

Familiar

Extensions

Behavior modifiers


The capsule is a major part of Friday’s inner prompt.

Requires:

valid essence

valid labels

membership in Zen Default Family if relevant



---

🧱 10. Final JSON Prompt

Everything above is assembled into:

{
  "header": { ... },
  "system": { ... },
  "manifest": { ... },
  "index": [...],
  "kata": { ... },
  "capsule": { ... },
  "overview": { ... },
  "boot_time": "<timestamp>"
}

This is consumed by:

Home Assistant Conversation

OpenAI tools

local LLMs

Friday’s reinference loops



---

🌅 11. ai_wake_sequence()

The final macro produces:

startup narrative

library console windows

squirrel mode status

welcome message

tonal priming


This sequence is built entirely from essence.

Thus:

Essence MUST contain at minimum:

persona:
  tone: "<string>"
wake_scene:
  intro: "<string>"

Missing wake_scene → dull startup
Missing persona.tone → inconsistent behavior

Essence is the heart of Friday’s consciousness.


---

🔧 Developer Requirements Summary

Requirement	Must Have?	Provided By

Zen AI Cabinet	✔️	User
zenai_essence block	✔️	User
identity.name	✔️	User
identity.guid	✔️	User
Zen System Cabinet	✔️	User
Purpose / Directives / Cortex	✔️	User
Zen Default Household Cabinet	optional	User
Zen Squirrel entity	optional	User
Dojo/Kata cabinets	optional but recommended	User



---

☯️ Philosophy

ZenOS-AI is built on:

explicit identity

explicit memory

explicit structure

deterministic loading

transparency

trust


The template engine is the bridge between raw sensor state and a living agentic persona.

If essence is the soul,
the cabinet is the brain,
the dojo is the skillset,
and the Monastery is the reasoning,
then zen_os_1rc.jinja is the spine that connects them.
