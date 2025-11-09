# 📘 ZenOS-AI Documentation

Welcome to the ZenOS-AI documentation hub.  
This section provides the conceptual and technical foundations of the ZenOS-AI architecture, focusing on the components already implemented in v1.0.

ZenOS-AI transforms Home Assistant into a fully agentic, persona-aware AI operating system.  
The documents here explain *how* that works: the cabinets, the Monastery, the summarizers, the semantic labeling system, and the Dojo-driven kung fu subsystem architecture.

Below is an index of the documentation currently included.

---

## 📚 Included Documentation

### **1. Zen Summarizer**
**Path:** `docs/zen_summarizer/readme.md`  
A complete walkthrough of the multi-layer summarization pipeline:  
- Dojo drawers and subsystem definitions  
- Kata generation  
- Monastery reflection  
- Zen Summary consolidation  
- Live Prompt assembly  
Explains how Friday and Kronk maintain awareness without flooding the context window.

---

### **2. Kung Fu Components**
**Path:** `docs/kung_fu/readme.md`  
Explains the structure and purpose of Kung Fu Components:  
- How each component defines a subsystem  
- Required context  
- Operational instructions  
- Safety constraints  
- Master switches  
- Interaction with the Index and Monastery

Shows how to write, version, and maintain components using the **KungFu Writer** tool.

---

## 🧭 How to Navigate
Each subsection contains a self-contained overview with links, examples, and reference structures.  
Start with the **Zen Summarizer**, then explore the **Dojo** and **Kung Fu Components**, as these form the active working model Friday uses to reason about your home.

For OS-level behavior and identity, see the **SYSTEM Cabinet**.

---

## 🛠 Contributing
All documentation is versioned alongside ZenOS-AI’s codebase.  
Contributions, improvements, and clarifications are welcome — especially around subsystem design patterns and label taxonomies.

---

If you're building your own agent with ZenOS-AI, this docs directory is your map.  
Welcome to the Monastery.
```
