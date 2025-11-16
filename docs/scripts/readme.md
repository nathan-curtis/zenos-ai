# 📘 **ZenOS-AI Script Modules**

Welcome to the **Script Modules** section of the ZenOS-AI documentation.
This directory contains formal documentation for all DojoTools and ZenOS-AI operational scripts that power Friday’s real-time automation, reasoning, and system-level capabilities.

These scripts are part of Friday’s *“hands and reflexes”* layer — the components that interact directly with Home Assistant services, calendar providers, inspection tools, cabinet storage, and other subsystems.

Each script is documented with technical behavior, required inputs, output structure, provider limitations, and expected integration flows.

---

## 📂 **Included Script Documentation**

### **1. Zen DojoTools Calendar — v1.10.3**

**File:** `zen_dojotools.calendar_readme.md`
**Type:** Technical Documentation
**Summary:**
This module provides unified calendar access for all Home Assistant calendar entities, including:

* Multi-calendar reads
* Unified timestamp handling
* Event inspection (where supported)
* Safe create operations
* Provider-safe update/delete (event_id-only)
* Label-based targeting
* Structured error handling
* Deterministic response formats for LLM agents

Additional features include robust provider-safety blocks, help documentation, and internal guardrails for multi-target or ambiguous calendar requests.

---

## 🎯 Purpose of This Folder

`/docs/scripts` exists to:

* Provide **clear, versioned technical documentation** for all ZenOS-AI and DojoTools script modules
* Enable onboarding of new contributors (human **or** AI)
* Maintain a trusted, single source of truth for script behavior
* Ensure that Friday, Kronk, and future agents always have stable reference material

Every script entering production **must** be documented here with:

1. Version
2. Changelog summary
3. Feature overview
4. Input/field specifications
5. Output/response schema
6. Known provider limitations
7. Safety constraints
8. Integration advice

---

## 🧱 Structure & Naming

* Script documentation files follow the pattern:
  `zen_dojotools.<scriptname>_readme.md`

* All files in this folder describe a *script*, not a cabinet, component, or summarizer.

* Changelog sections should be mirrored in the repo’s main `CHANGELOG.md` if applicable.

---

## 🚀 Adding a New Script Doc

1. Place your new documentation file under:
   `/docs/scripts/<your_filename>.md`

2. Update the index (`/docs/scripts/zen_dojotools.calendar_readme.md` or equivalent)

3. Add an entry to the global documentation index at:
   `/docs/index.json`

4. Submit a PR with:

   * The new doc file
   * The updated index
   * A brief commit summary

---

## 🤖 Notes for AI Contributors (Friday, Kronk, etc.)

AI assistants working inside ZenOS-AI should:

* Reference these docs *before* attempting tool execution
* Surface user-friendly errors when a limitation is known
* Use script metadata to guide behavior when timestamps or provider variations appear
* Treat this folder as canonical truth
