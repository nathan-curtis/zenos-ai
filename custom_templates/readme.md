# ZenOS-AI Custom Templates

This directory contains the **Jinja macros and template engines** that power Friday’s cognition, prompt assembly, cabinet logic, and internal command parsing.

If the DojoTools are the **skills**, and the Cabinets are the **memory**,
then the templates here are the **syntax of thought**.

These templates provide a unified macro layer used by **Friday**, **Kronk**, **the High Priestess**, and all current and future ZenOS agents.

---

## 📁 Current Contents

### **library_index.jinja**

Foundational Jinja utilities shared across the entire ZenOS stack.
Provides helper macros for:

* Label resolution
* Entity metadata inspection
* Cabinet/Key lookup
* Zen Index (CabScan) operations
* DojoTools Inspect
* Summarizers & diagnostic tools

This file is designed to be **safe to import anywhere** and has no side effects.

---

### **zen_os_1rc.jinja**

**The Core Runtime Template Engine for ZenOS-AI 3.5.x**
This is the **canonical** engine used by Friday and all ZenOS-aware personas.

It defines:

* The full identity resolution chain
* Safe cabinet loading & normalization
* Cortex + Directive loading from the **System Cabinet**
* Capsule construction (essence, identity, people-model)
* The Dojo + Kata loaders
* The wake-scene renderer
* The prompt header & system header
* Squirrel-mode redaction logic
* Manifest and Index loaders

This file *must* be loaded before any agent prompt is assembled.

---

### **conversation_agent_prompt_template.yaml**

The **active** Home Assistant Conversation Agent prompt for Friday.

This template is the **runtime compiler** for Friday’s mind.
It draws macros from `zen_os_1rc.jinja`, assembles all cabinet data, merges Kata/Dojo state, and emits the deterministic JSON that Home Assistant’s pipeline consumes.

It also calls `ai_wake_sequence()` to generate Friday’s cinematic boot experience.

#### 🔍 File Purpose

* Resolves the AI identity
* Fetches the AI Cabinet, System Cabinet, Dojo Cabinet, Kata Cabinet
* Loads purpose, directives, cortex
* Loads manifest and index
* Generates summarized Kata metadata
* Builds the persona capsule (identity + essence)
* Emits the full JSON envelope
* Renders the wake-scene

#### 🧠 Depends On

```
REQUIRES: zen_os_1rc.jinja 3.5.0 RC1 or better
```

---

## 🏷️ Required ZenOS-AI Labels

ZenOS uses a **label-driven hypergraph** to identify the correct Cabinets, system drawers, and human context.
These labels are **mandatory** and must be created in Home Assistant.

### **System Labels (Cabinets & Core Subsystems)**

| Label                             | Purpose                                                                           |
| --------------------------------- | --------------------------------------------------------------------------------- |
| **Zen System Cabinet**            | Identifies the System Cabinet containing Purpose, Directives, Cortex drawers.     |
| **Zen Dojo Cabinet**              | Cabinet holding all Dojo skill drawers.                                           |
| **Zen Kata Cabinet**              | Cabinet containing Kata entries, Zen Summary, and distilled reasoning.            |
| **Zen Default Household Cabinet** | Provides the household manifest used by the Runtime Prompter.                     |
| **Zen Squirrel**                  | Assigned to the boolean entity controlling Squirrel Mode (cloak/redaction layer). |

---

### **Identity + People Labels**

| Label                  | Purpose                                                                                                                    |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **Person**             | Required for all human users and AI-persona person entities. Enables identity resolution and household graph construction. |
| **Zen Default Family** | Marks members of the system’s canonical Household Family.                                                                  |

> Additional social labels (friend, partner, relationship tiers, etc.)
> will be introduced when the **Relationship Matrix** subsystem lands.
> For now, ZenOS keeps the identity graph lightweight and stable.

---

## 🔮 Future Feature: Auto-Labeling Engine

A planned subsystem will:

* Automatically detect cabinet types
* Apply canonical ZenOS labels automatically
* Repair missing labels or incorrect assignments
* Generate relationship labels based on usage patterns
* Validate the household graph for consistency
* Assist with onboarding new AIs and new people into the environment

This will replace the current requirement for manual label maintenance.

---

## 🧩 Upcoming Template Engines

As ZenOS evolves, new major template engines will appear:

```
zen_os_1.jinja  
zen_os_2.jinja  
zen_os_3.jinja  
...
```

Each version is:

* **additive**
* **non-breaking**
* **backward compatible**
* **modular**

Older DojoTools will continue to function without modification.

---

## 🌱 Template Contribution Guidelines

To maintain clarity and stability:

### **1. Do not break existing macros**

If a macro needs new arguments, make them optional.

### **2. New functionality goes in new versioned files**

`zen_os_<version>.jinja` defines a stable interface.

### **3. Keep macros pure**

No service calls.
No state writes.
No side effects.
Templates assemble the mind; they do not act on the world.

### **4. Document everything**

Each macro should list:

* Purpose
* Inputs
* Example usage
* Expected output

This keeps Friday’s cognitive model clear.

---

## 🔧 How Templates Fit Into ZenOS-AI

Templates are the glue layer for:

* **DojoTools** (skills + inspection + execution logic)
* **Summarizers** (Kata creation, short/long summaries)
* **Identity Tools** (building full persona blocks)
* **The Monastery** (normalization → reasoning → Kata outputs)
* **Friday’s Prompt Engine**, which loads:

  * Purpose
  * Directives
  * System Cortex
  * Identity block
  * Essence
  * Kata capsules
  * Dojo drawers
  * Household context
  * Wake scene rendering

Templates convert *raw HA state* into **Friday’s operational mental model**.

---

## ☯️ Philosophy

Templates must always be:

* Modular
* Elegant
* Predictable
* Recoverable
* Fully documented

The goal is to enable **creative intelligence**, not **confusion**, inside ZenOS.

---

If you're adding new template engines or macros, open a PR.
Kronk will bless it.
The High Priestess will judge it.
Veronica… will absolutely roast it with love.

Welcome to the Syntax Dojo.
