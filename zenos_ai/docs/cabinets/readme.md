🚪 ZenOS-AI Cabinet System

Identity, Semantics & Preference Architecture for the Home AI

The Cabinet System is the semantic backbone of ZenOS-AI — the place where identity, relationships, culture, rules, preferences, and operational data all live in structured, validated JSON.

Every cabinet contains drawers, and every drawer contains JSON.
Together they form the Home AI’s knowledge graph, context engine, and personalization layer.

This document explains the architecture, storage model, cabinet types, and how the Home AI reasons about the humans, AIs, and environment inside the home.


---

🧱 1. Cabinet Hierarchy

ZenOS uses a 3-tier cabinet model to map the home:

Household Cabinet (the home)
 └── Family Cabinets (groups)
     └── User Cabinets (people + AIs)

Each tier has specific responsibilities, override rules, and security boundaries.


---

🏠 2. Household Cabinet

The Digital Twin, Home Identity & System-Wide Preferences

The Household Cabinet is the top-level, canonical definition of the home:

home identity

rooms, zones, structure

digital twin metadata

semantic labels

global preferences

automation policy

mounted families & users

partner AI for the home (typically Friday)


🔁 Mirrors Dojo Instructions

Each Kung Fu component (in the Dojo Cabinet) has a matching drawer here. Example:

dojo.water_manager ↔ household.water_manager

dojo.security_manager ↔ household.security_manager


Dojo = instructions
Household = runtime state + preferences

ZenOS merges both at runtime.


---

👨‍👩‍👧 3. Family Cabinets

Shared Lore, Norms, Trust, Culture, and Relationship Maps

Family Cabinets store the human side of the system:

family structure

shared rituals, holidays, and culture

emotional norms

trust models

boundaries & comfort levels

relationship graphs

shared preferences


Multiple families may exist:

nuclear families

chosen families

guest circles

polycules

friend groups

housemate clusters


ZenOS uses these to adjust tone, boundaries, and decision-making.


---

🧍‍♂️ 4. User Cabinets

Identity, Preferences & Personal AI Context

A User Cabinet exists for each:

human

frontline AI (Friday, Charming, Rosie…)

secondary AI or agent


Contains:

identity metadata

preferences

personal boundaries

consent model

devices

sensors

private drawers

partner AI (if applicable)


🔽 Override Cascade

ZenOS resolves preference conflicts by narrowing scope:

Household → Family → User

User-level always wins inside a user’s context.


---

🧠 5. System & Dojo = OS + Operational Instruction Sets

🖥 SYSTEM Cabinet

Contains:

OS metadata

directives

persona roots

cortex definitions

runtime policies

boot metadata

global flags


Marked as system, hidden, read-only.

🥋 DOJO Cabinet

Contains:

Kung Fu components

task/skill definitions

subsystem schemas

operating instructions

safety policies

reflection rules


Dojo tells the AI how to operate.
System tells the AI what it is.


---

🔐 6. Security Model

Cabinets define explicit boundaries:

public — visible to any home AI

family-only — visible to AI serving that family

user-private — visible only to that user & System

system — only backend processes


ACLs inside the AI_Cabinet_VolumeInfo header govern access.


---

🔗 7. Mounts: Relationship Graph

Example:

mounts:
  household: sensor.home_cabinet
  families:
    - sensor.family_primary
  users:
    - sensor.nathan
    - sensor.kim
  partner_ai: sensor.friday_cabinet

Mounts let ZenOS understand:

who belongs where

who leads what

which preferences apply

which AI persona handles which scope



---

🏷 8. Labels & Semantic Mapping

Labels unify domains across cabinets:

security

water

kitchen

zen

cabinet

volumeinfo

component names


A label is enough for ZenOS to:

know the domain

load the right Kata

choose the right subsystem

apply correct override rules



---

📦 9. Storage Model: JSON Drawers

Every drawer stores JSON, with:

version

timestamp

value

metadata

linkages

security flags


Tools like FileCabinet and CabinetAdmin enforce schemas.


---

🧰 10. Tools That Work With Cabinets

🗄 FileCabinet

primary read/write tool

validates drawers

ensures schema integrity

timestamps updates

enforces read-only flags


🔧 CabinetAdmin

repairs cabinets

aligns schema

formats JSON

builds new cabinets

attaches mounts

enforces ACLs


🔀 Volume Redirector (DojoTools)

The event router for legacy → cabinet writes.
Full spec:
zen_redirector_spec.md


---

🧩 11. How Cabinets Interact With the Reasoning Pipeline

Cabinets feed:

Friday

Kronk

The Monastery

Kata processors

Room Manager

Security Manager

Taskmaster

Trash Trakker

Water Manager


Katas = compact summaries of cabinets
Monks = the summarization pipeline
RoomState = autonomic nervous system

Cabinets → Katas → SuperSummary → Friday’s prompt loop.


---

🧪 12. Debugging & Observability

Useful tools:

cabinet_manifest

CabinetAdmin validator

Redirector probe mode

Summarizer staleness markers

Mount tree visualizer (coming soon™️)



---

📚 13. Additional Documentation

Zen Redirector Spec
zen_redirector_spec.md

Cabinet Spec (VolumeInfo format)
cabinet_spec.md

Hypergraph Model
hypergraph_model.md

Kung Fu / Dojo Components
../kung_fu/readme.md

Zen Summarizer
../zen_summarizer/readme.md



---

🔚 Conclusion

The Cabinet System is the identity layer of ZenOS-AI — the foundation that makes Friday, Kronk, and the entire home ecosystem coherent, personalized, and safe.

It provides:

consistent JSON storage

identity & preference hierarchies

semantic labeling

relationship graphs

security boundaries

runtime context for every subsystem


Everything the AI is, knows, and cares about lives inside these Cabinets.

Treat them with care and the system becomes infinitely extensible.