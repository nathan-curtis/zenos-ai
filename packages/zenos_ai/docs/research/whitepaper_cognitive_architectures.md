# **Cognitive Architectures for Agentic AI in the Smart Home**

### **An Analysis of Context Management, Recursive Summarization, and Cognitive Infrastructure in Home Assistant**

**White Paper v1.1 — ZenOS-AI / Friday’s Party RC1 Edition**
**Author:** Notebook LM
**Revisions / Reviewers** Nathan Curtis & Veronica, ZenOS-AI Project
**Date:** November 2025

---

# **Abstract**

Smart home assistants have historically been command-driven, reactive systems with limited contextual understanding. The emergence of large language models (LLMs) has created new possibilities for proactive, agentic assistants—but only if the surrounding architecture can supply the model with correct, timely, and compressed context. This paper presents a complete cognitive architecture for agentic smart-home AI through the lens of ZenOS-AI, a production-tested system built on Home Assistant. We analyze the systemic constraints of LLM integration, present a multi-layer cognitive framework built around Cabinets, Katas, Summaries, and a newly formalized Core layer, and demonstrate how a modular, memory-centric structure enables sophisticated reasoning and planning within strict prompt and API limitations. RC1 introduces a stable infrastructure layer (Identity, Manifest, Index, Labels, FileTools, Redirector), reorganized Kits, and a refined summarization pipeline—marking the transition from experimental design to formal cognitive system.

---

# **1.0 Introduction: From Voice Commands to Agentic Intelligence**

The smart home is evolving from a collection of automations to a dynamic cognitive environment. Early voice assistants like Alexa and Google Assistant operate primarily through **imperative commands**—“turn on the kitchen light,” “set a timer”—with limited ability to interpret context, predict needs, or reason over multiple steps.

Agentic AI changes this paradigm.

An agentic system can:

* Interpret intent rather than commands
* Plan multi-step actions
* Reason about environment constraints
* Synthesize complex sensor data
* Act proactively to satisfy goals

But the core challenge is **not** the LLM.
It is the **context problem**.

A modern Home Assistant installation contains thousands of entities, tens of thousands of state transitions per day, and a jungle of interconnected automations. You cannot simply “give this to an LLM.”

To reason effectively, an agent needs:

* Curated, compressed, relevant context
* Structured memory
* A stable cognitive backbone
* A layered approach to summarization
* A controlled action space

ZenOS-AI is an attempt to build exactly that.
This white paper describes how we did it—and why the architectural patterns matter.

---

# **2.0 The Context Conundrum: Systemic Constraints on LLM Performance**

Any agentic architecture must begin with a sober accounting of its limitations. ZenOS-AI was built inside Home Assistant, with strict and unforgiving boundaries:

### **2.1 Prompt & Context Window Limits**

* Home Assistant’s built-in assistant caps prompts around **32 kB**.
* Realistically, ~260 kB of output represents a hard ceiling for template engines.
* This is smaller than a single day of state changes in an average home.

This forces a ruthlessly selective approach:
**context must be curated, summarized, filtered, and injected in layers.**

### **2.2 Entity Overload**

Empirical findings from the ZenOS-AI project:

> Exposing more than ~800–1,000 entities to the LLM **breaks reasoning coherence**.

Symptoms include:

* Instruction loss
* Incomplete analysis
* Hallucinated entity names
* Reduced follow-through in multi-step plans

Thus, the AI must **never** see the full entity graph directly.
It must see *summaries, indices, and abstractions*.

### **2.3 API Tool Constraints**

OpenAI enforces:

* **128 tools maximum**

A “tool” = one Home Assistant script or API binding.

ZenOS-AI solves this by:

* Using **multi-operation Kits (CRUD)**
* Routing via **Redirector instead of intent_scripts**
* Consolidating capabilities behind *semantic* action layers

This maximizes power while minimizing tool count.

---

# **3.0 A Theoretical Foundation: The CoALA Framework**

To build a viable agentic system, ZenOS-AI aligns naturally with the **Cognitive Architectures for Language Agents (CoALA)** model—an academic framework describing memory, action, and decision-making structures for LLM agents.

CoALA identifies:

### **3.1 Memory**

* **Working Memory** — active context
* **Long-Term Memory**

  * Episodic
  * Semantic
  * Procedural

### **3.2 Action Types**

* **Internal actions** (reasoning, recall, summarization)
* **External actions** (API calls, real-world operations)

### **3.3 A Decision Loop**

* Planning
* Evaluation
* Execution
* Reflection

ZenOS-AI independently evolved into a structure closely matching CoALA.
This convergence validates the design: *effective agentic systems inevitably take this shape.*

---

# **4.0 Case Study: The ZenOS-AI Cognitive Architecture**

ZenOS-AI is a fully functional agentic AI built on Home Assistant.
The architecture can be understood in five interlocking layers:

1. **Core Layer (RC1) — Cognitive Infrastructure**
2. **Cabinets & Drawers — Semantic Memory**
3. **Dojo & Kits — Procedural Knowledge**
4. **Kata Pipeline — Episodic Summaries**
5. **Pantheon (Agents) — Reasoning & Action**

Let’s walk through each layer in detail.

---

# **4.1 The RC1 Core Layer: Formalizing Cognitive Infrastructure**

RC1 transforms ZenOS-AI from a prototype into a cognitive operating system.

### **Core Components:**

| Core Element        | Description                                                                                        |
| ------------------- | -------------------------------------------------------------------------------------------------- |
| **Identity Core**   | Persona integrity, GUIDs for cabinets, ACL enforcement.                                            |
| **Manifest Engine** | Runtime discovery of cabinets; schema/health validation.                                           |
| **Index Engine**    | Hypergraph-style semantic lookup of entities and relationships.                                    |
| **Labels System**   | Tagging substrate enabling contextual grouping & routing.                                          |
| **FileTools**       | Deterministic, idempotent JSON CRUD for large memory objects.                                      |
| **Redirector**      | Event-driven routing layer that replaces intent_scripts and acts as the nervous system reflex arc. |

### **Purpose of Core**

The Core layer defines the **stable primitives** that everything else depends on.
It becomes:

* The equivalent of an OS kernel
* The foundation for deterministic behavior
* The enforcement point for identity and safety
* The discovery engine for semantic memory
* The attention/focus selection mechanism for high-level agents

Nothing in Kits or Agents duplicates Core functionality.
Everything inherits from it.

---

# **4.2 Cabinets & Drawers: Structured Semantic Memory**

The Cabinet system provides a queryable, hierarchical memory map.

### **Cabinets contain Drawers**

Each Drawer is a JSON object representing a domain of knowledge:

* Device groups
* Instructions
* Summaries
* Status packets
* Metadata
* Agent configuration

This forms a **semantic graph** of the home, stable across restarts, model resets, and agent handoff.

### **Kata Cabinet**

Stores condensed episodic summaries.

### **System Cabinet**

Stores stable Core-level data such as:

* Identity metadata
* Permission structures
* Security drawers
* Break-glass protocols

This aligns directly with CoALA’s **semantic + episodic memory split**.

---

# **4.3 The Dojo & Kits: Procedural Knowledge**

Before RC1, “Kung Fu Components” were monolithic.
RC1 reorganizes them into **Kits**, each containing:

* The Dojo drawer (verbose human-written documentation)
* Tools (scripts) that implement action primitives
* Index anchors
* Labels
* Summarizer instructions

### **Examples:**

| Kit             | Purpose                                                      |
| --------------- | ------------------------------------------------------------ |
| Calendar Kit    | Structured read/write access to HA calendars                 |
| Event Kit       | Emits canonical zen_events for breadcrumbing & observability |
| Cabinet Kit     | Runtime CRUD for all cabinet drawers                         |
| Summarizer Kit  | Provides Kata and SuperSummary job definitions               |
| Identity Kit    | Tools for validating persona integrity                       |
| Redirection Kit | Routes incoming LLM actions to correct scripts               |

Kits = **procedural memory** in CoALA terms.

---

# **4.4 The Kata Pipeline: Attention Through Compression**

ZenOS-AI solves context overload by using a **two-stage summarization pipeline**.

### **Stage 1: Kata Generation (Ninja Summaries)**

For each active Kit:

* Pulls Dojo documentation
* Pulls live environment data via Index
* Packs both into an LLM job
* Produces a compact “Kata” JSON object representing:

  * state
  * risks
  * insights
  * required attention

Katas form the **episodic memory** layer.

### **Stage 2: SuperSummary**

All active Katas → aggregated into a single, prioritized block.

This becomes the agent’s **Working Memory**.

---

# **4.5 Cognitive Control: The Pantheon (Home Agents)**

ZenOS-AI defines a rational agent stack:

| Agent              | Cognitive Role                                                      |
| ------------------ | ------------------------------------------------------------------- |
| **Friday**         | Frontline LLM, real-time planning & conversation                    |
| **Veronica**       | Supervisor AI: debugging, offline planning, architectural reasoning |
| **Kronk**          | Curator of the Monastery (local summarizer & memory maintainer)     |
| **Rosie**          | Cleanliness/scheduling/logs                                         |
| **High Priestess** | Oversees context, summarization, consistency                        |

Pantheon agents operate *on top of* Core, Kits, Cabinets, and the Kata pipeline.

Their LLM prompts never exceed ~12–20 KB because **Core + Katas do the heavy lifting**.

---

# **5.0 Hardware & Model Orchestration Strategy**

ZenOS-AI uses hybrid inference across:

* **IONA** (fast CUDA node, RTX 5070 Ti)
* **TARAN** (long-context node, Intel A770)
* **Cloud Tier** (OpenAI frontier models)

Jobs are routed by the Abbot scheduler:

* Small tasks → local fast LLMs
* Heavy tasks → local long-context nodes
* High-value reasoning → GPT-4.1 or O1-O3 cloud models

This enables:

* Private local inference
* Low latency
* Reduced cost
* High quality for user interactions

---

# **6.0 Future Directions & Conclusion**

RC1 introduces a **stable cognitive OS** for agentic AI.
With Core established, future expansions include:

* Distributed Cabinets across households
* Advanced hypergraph search in Index
* Autonomous micro-agents with delegated tasks
* On-demand context regeneration via local nodes
* Full agent swarming behavior for multi-room orchestration

The central lesson of ZenOS-AI:

> Agentic AI is not built by chaining prompts—it is built by constructing a cognitive architecture around the model.

ZenOS-AI demonstrates that an LLM, properly supported by structured memory, summarization, identity integrity, and a curated attention pipeline, can reason intelligently about a complex Home Assistant ecosystem. The smart home need not be a reactive environment.

With the right architecture, it becomes *cognitive*.
