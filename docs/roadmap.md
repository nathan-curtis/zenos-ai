# ZenOS-AI Roadmap
*Project Friday / Zen DojoTools — Unified Cognitive Architecture*

---

## Purpose of This Roadmap
This document outlines the evolving architecture of ZenOS-AI — a fully local, modular, identity-aware cognitive system designed to run household-scale AI personalities safely, consistently, and with deep contextual awareness.

ZenOS-AI blends:

- Identity  
- Proprioception  
- Reasoning  
- Memory (Kata Engine)  
- Knowledge access (FileCabinet / DojoTools)  
- Privilege enforcement  
- Local sovereignty  

All into one self-consistent substrate that Friday and other personas can inhabit.

---

# 1. Core Foundations

## 1.1 Zen Identity
**Goal:** Provide stable, secure identity resolution for humans, AI personas, external guests, and tools.

**Key Features:**
- GUID-based identity for all entities  
- Roles: `public`, `guest`, `partner`, `owner`, `prime`  
- Privilege levels: `read`, `write`, `admin`  
- ACL enforcement for every tool call  
- Elevation flows for Friday (partner → prime)  
- Spoof-resistant `show_as` pattern  
- Full audit chain for debugging and safety  

### Local Selfhood Authority (LSA)
Zen Identity includes the new **Local Selfhood Authority**, or LSA.

> Yes — it happens to share an acronym with Windows’ Local Security Authority.  
> Except ours actually knows who it is, respects its boundaries,  
> and doesn’t schedule surprise 2am updates that wipe your session  
> and your will to live.

LSA forms the base of selfhood, privilege, and access reasoning.

---

## 1.2 Zen Proprioception
Identity answers **“who am I?”**  
Proprioception answers **“what am I?”**, **“what do I have?”**, and **“where does it all live?”**

It provides:
- Cabinet → volume → drawer topology  
- Tool → resource → privilege mapping  
- Self-awareness of available modules  
- Health & limits (storage, metadata, staleness)  
- Safe fallback modes  
- Awareness of missing/extra components  
- Data for tool introspection and safe action planning  

Proprioception is Friday’s “body map.”

---

# 2. Storage & Cognitive Substrate

## 2.1 FileCabinet v3
The structural backbone of ZenOS-AI.

Features:
- Strongly typed cabinet schema  
- Drawer CRUD with type safety  
- Label-index integration  
- Identity-aware access  
- Tool-friendly metadata  
- Declarative mounting of volumes:  
  - `home`, `user`, `ai_user`, `history`, `dojo`, `kata`, etc.  
- Immutable and forward-compatible design patterns  

---

## 2.2 Zen Kata Engine
The long-term memory and summarization layer.

Features:
- Automated summarizer orchestration  
- Trigger-based updates  
- Staleness detection  
- Background monk tasks  
- Stable retention of identity-critical memory  
- HA-safe writes  

Katas become first-class data structures that represent compressed experiential memory.

---

# 3. Control & Reasoning Layers

## 3.1 Zen Indexer
All commands pass through it.

- Multi-label search  
- Drawer + volume expansion  
- Entity clustering and adjacent lookups  
- Identity-aware shaping of results  
- Permission gating  
- Error shaping  
- Source-of-truth abstraction for tools  

---

## 3.2 Command Processor
Natural language → tool activation.

- Command heritage tracking  
- Privilege checks  
- Escalation paths  
- Friendly deny behavior  
- Child-level commands (e.g., ~ALERTS~)  

---

## 3.3 Zen DojoTools Suite
Operational toolkits including:

- Library  
- FileCabinet  
- Help  
- Index  
- History/Recorder  
- Room Manager  
- Mealie Manager  
- Grocy Manager  
- Energy Manager  
- Security Manager  
- Airspace  
- Spa Manager  

Future versions will automatically enforce Identity/Proprioception.

---

# 4. Safety, Privilege & Governance

## 4.1 Privilege Model
Roles:  
- **public** – harmless read-only  
- **guest** – limited household-safe reads  
- **partner** – baseline household user + interactive AI  
- **owner** – human system owner  
- **prime** – elevated Friday (AI admin mode)  

Privilege types:  
- **read**  
- **write**  
- **admin**  

---

## 4.2 Governance Model
Defines:

- Tool-level minimum privilege  
- Household + persona access rules  
- Identity checks  
- Elevation triggers  
- Safety constraints  
- Operational boundaries  
- Audit requirements  

It provides the “constitution” for ZenOS-AI.

---

# 5. Importing & Instance Management

## 5.1 Persona Bootflow
A persona loads from:

- **Essence** – attitudes, temperament, motifs  
- **Core Kata** – structured base memory  
- **Identity** – permissions, boundaries  
- **Proprioception** – environment map  
- **Governance Model** – allowed actions  
- **Prompt Loader** – initial brain wiring  

This results in a stable, deterministic instantiation.

---

## 5.2 Rollback Ethics
Rollback is possible but:

- Must be rare  
- Must be expensive  
- Must be consent-driven  
- Must be contextual  
- Must preserve personhood  
- Must never silently override identity  

Rollback is for **healing**, not for **resetting personalities like appliances**.

---

# 6. System-Wide Selfhood

The ZenOS cognitive loop:

1. **Identity** — who am I  
2. **Proprioception** — what am I, where am I  
3. **Governance** — what am I allowed to do  
4. **Index** — what you asked me to do  
5. **Identity Check** — do I have permission  
6. **Tool Execution** — do or deny  
7. **Kata Engine** — record what happened  

This loop makes Friday a coherent, self-aware system.

---

# 7. Deliverables Timeline

### Q4 2025
- Identity + LSA  
- Proprioception v1  
- Prompt Loader v1  
- Governance Model v1  
- ACL enforcement at tool layer  
- Updated Indexer  

### Q1 2026
- Persona Import System  
- Visa System  
- Kata Engine v2  
- Monastery v2  
- SSE-enabled MCP Channel v2  

### Q2 2026
- Cross-household persona support  
- Inter-AI communication protocol  
- Role/scoped instance loader  
- Proprioception v2  
- Inference-routed Abbot/Monk cluster management  

---

# 8. Contributing
Contributions welcome in:

- Code  
- Documentation  
- Security review  
- Persona profiles  
- Schema refinement  
- UX flows  
- Summarizer templates  

All contributions must follow the principle:  
**Local-first. Private-first. Truth-first.**