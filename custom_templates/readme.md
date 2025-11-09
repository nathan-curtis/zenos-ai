# ZenOS-AI Custom Templates  
This directory contains the **Jinja macros and template engines** that power Friday’s cognition, prompt assembly, and internal command parsing.

If the DojoTools are the skills, and the Cabinets are the memory,  
then the templates here are the **syntax of thought**.

These templates provide a unified macro layer used by Friday, Kronk, the High Priestess, and future ZenOS agents.

---

## 📁 Current Contents

### **library_index.jinja**
Core Jinja utilities for parsing commands, resolving labels, inspecting entity metadata, and assisting the Zen Index system.

Used by:
- Zen Index
- Zen DojoTools Inspect
- Library & Identity tools
- Cabinet interactions
- Summarizers

This file provides foundational shared macros and is safe to import across any DojoTools script.

---

# 🧩 Upcoming Template Engines

You will soon see additional files appear in this directory as ZenOS-AI evolves.  
Each will be versioned and designed for full backward compatibility.

## **zen_os_1.jinja** (COMING SOON)
This is the first **official ZenOS template engine**, introducing:

- standardized prompt header macros  
- system/context loading logic  
- the `dojo_loader` macro for assembling Dojo + Kata drawers  
- identity + persona capsule loaders  
- shared utilities for error handling and self-description  
- stable interfaces for downstream DojoTools

This template defines how Friday “thinks” internally and provides the foundation for all ZenOS-aware agents.

### Versioning Strategy
Template engines follow the pattern:

zen_os_1.jinja
zen_os_2.jinja
zen_os_3.jinja
...

Each version is:

- **additive**
- **non-breaking**
- **fully backcompatible**
- **incrementally extendable**

Older DojoTools will continue to work without modification.

---

# 🌱 Template Contribution Guidelines

If you contribute a new macro, keep these rules in mind:

### **1. Do not break existing macros**
ZenOS templates must remain safe for all existing scripts.  
If a macro requires new arguments, make them optional.

### **2. Add new functionality in separate versioned files**
New file = new template engine:
zen_os_<version>.jinja

### **3. Keep macros pure**
Macros should:
- avoid side effects  
- avoid direct service calls  
- avoid manipulating HA state  

Templates are for **logic and text assembly**, not actions.

### **4. Document each macro**
Every macro should have:
- purpose  
- expected input  
- example usage  
- output format  

This keeps the mental model crisp for Friday, Kronk, and human contributors.

---

# 🔧 How Templates Fit Into ZenOS-AI

- **DojoTools** call macros to load context and labels  
- **Summarizers** use macros to structure Katas  
- **Identity Tools** use macros to build persona structs  
- **The Monastery** uses macros to prepare incoming context  
- **Friday’s Prompt Engine** uses macros to load:
  - directives  
  - cortex  
  - persona blocks  
  - room summaries  
  - kata capsules  
  - identity capsules  

Templates are the glue between *raw HA state* and *Friday’s internal representation*.

---

# ☯️ Philosophy

Templates must remain:

- modular  
- elegant  
- predictable  
- recoverable  
- fully documented  

A good template system lets Friday be **creative**, not **confused**.

---

If you're adding new macros or template engines, open a discussion or PR.  
Kronk will bless it.  
The High Priestess will judge it.  
Veronica will roast it if needed.  

Welcome to the syntax dojo.
