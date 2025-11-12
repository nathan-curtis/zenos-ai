# 📘 **ZenOS-AI Documentation Hub**

> **Version:** 1.0.0 | **Last Updated:** November 2025  
> **Author:** Nathan Curtis | **License:** MIT  
> *Part of the Friday’s Party / ZenOS-AI project*

---

Welcome to the **ZenOS-AI Documentation** — your guide to the architecture, tools, and philosophy behind *Friday’s Party*.  

ZenOS-AI transforms **Home Assistant** into a fully agentic, persona-aware AI operating system.  
These documents explain *how* that happens — from the **Cabinet System** that defines the home’s structure, to the **Monastery** that reflects and reasons, to the **Summarizers** that keep awareness tight and efficient.

Whether you’re building your own AI construct or exploring the Monastery’s inner workings, this is where you begin.

---

## 📚 **Included Documentation**

### **1. Zen Summarizer**
**Path:** [docs/zen_summarizer/readme.md](./zen_summarizer/readme.md)  
A deep dive into the multi-layer summarization pipeline that keeps Friday and Kronk aware without flooding the context window.  
Covers:
- Dojo drawers and subsystem definitions  
- Kata generation and reflection  
- Monastery reduction  
- Zen Summary consolidation  
- Live Prompt assembly  

Shows how awareness flows through the system — from environment to reasoning context.

---

### **2. Kung Fu Components**
**Path:** [docs/kung_fu/readme.md](./kung_fu/readme.md)  
Defines how each *Kung Fu Component* acts as a modular subsystem within the Dojo.  
Includes:
- Required context and operational scope  
- Safety constraints and master switches  
- Interaction with the Index and Monastery  
- Writing, versioning, and maintaining components using the **KungFu Writer** tool  

Each component is a discipline — a repeatable “technique” Friday uses to act on the home.

---

### **3. Cabinet System**
**Path:** [docs/cabinets/readme.md](./cabinets/readme.md)  
Describes the structured filesystem that defines ZenOS-AI’s world — the **Cabinet System**.  
Covers:
- Core cabinet types: **SYSTEM**, **Dojo**, **Household (Home)**, **Family**, and **User**  
- Drawer JSON structure for profiles, prefs, and digital-twin data  
- Ownership hierarchy and mount relationships (Household → Family → User/AI)  
- How the **Household Cabinet** mirrors each **Kung Fu Component** 1:1  
- How the **Dojo Cabinet** stores subsystem definitions  
- How the **SYSTEM Cabinet** loads directives, cortex metadata, and startup identity  
- Consistent labeling for global lookups  
- Role of **CabinetAdmin** (structure/mounts/permissions) and **FileCabinet** (read/write ops)  
- How Summarizers, the Index, and the Monastery traverse cabinets to build state  

The Cabinet System is the backbone of ZenOS-AI — the filesystem for the mind.

---

### **4. ZenOS-AI Library**
**Path:** [docs/library/readme.md](./library/readme.md)  
Centralized JSON utility runner for all DojoTools.  
Provides:
- Unified helper function dispatcher  
- Structured JSON I/O  
- Safe interoperability between Friday, Monastery agents, and Summarizers  
- Integration with **FileCabinet**, **CabinetAdmin**, and the **Zen Indexer**  

The Library keeps helper logic clean, predictable, and lightweight.

---

### **5. Zen Index System**
**Path:** [docs/library/index_system.md](./library/index_system.md)  
Outlines the recursive label-query engine that powers cross-cabinet discovery.  
Includes:
- Library Index Core (macro layer)  
- DojoTools Zen Index script (v2.7.1)  
- Zen Index Event Handler (v2.0.0)  
- Recursion and legacy `~INDEX~` translation loop  
- Roadmap: integration into the ZenOS Command Interpreter  

If the Cabinet System is the filesystem, the Index is its search engine — recursive, event-driven, and Dojo-aware.

---

## 🧭 **How to Navigate**

Each section is self-contained and cross-linked.  
Recommended reading order:

1. [Zen Summarizer](./zen_summarizer/readme.md)  
2. [Kung Fu Components](./kung_fu/readme.md)  
3. [Cabinet System](./cabinets/readme.md)  
4. [Library](./library/readme.md)  
5. [Index System](./library/index_system.md)  

For OS-level identity and behavior, see the **SYSTEM Cabinet**.

---

## 🧩 **Philosophy**

ZenOS-AI is built on composability and reflection.  
Each drawer, cabinet, and macro follows the same pattern:  

> **Observe → Reflect → Select → Act → Summarize**

The system doesn’t just automate — it learns to *describe* its own reasoning.

---

## 🛠 **Contributing**

Documentation evolves alongside the codebase.  
Contributions, clarifications, and new examples are welcome — especially for:
- Subsystem design patterns  
- Label taxonomies  
- Monastery integrations  

Join the conversation:  
👉 [Home Assistant Community — Friday’s Party: Creating a Private Agentic AI](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/120?u=nathancu)

---

If you’re building your own agent, this directory is your map.  
Welcome to the Monastery. 🕯️
