# **ZenOS-AI Library 2.0**

**Unified Utility Runner • Structured JSON Tools • DojoTools-Compliant**

---

The **ZenOS-AI Library** is a core DojoTool that provides **Friday** and **the Monastery** with a centralized utility runner.
Instead of registering dozens of small helper tools in the LLM namespace, the Library exposes these utilities through a single script:

```
script.zen_dojotools_library
```

The Library accepts:

* **`tool`** — the specific library function to invoke
* **`input_data`** — a structured JSON payload with parameters for that function

The output is always **machine-readable JSON**, making it safe for tool-calling models, the Summarizer pipeline, and Monastery agents.

---

## ✅ Why the Library Exists

1. **Avoids tool-namespace bloat**
   Every “tiny helper” doesn’t need to exist as a separate tool — they live behind one dispatcher.

2. **Keeps Friday’s prompt clean**
   Friday can request utility behavior without adding extra prompt weight.

3. **Standardizes JSON I/O across DojoTools**
   All Library calls return normalized JSON, making automation predictable.

4. **Allows complex DojoTools to call other DojoTools safely**
   The Library serves as a neutral dispatch layer.

5. **Supports prompt-side `~COMMANDS~` macros**
   These macros reference the Library, but no new macro language is introduced.

---

## ✅ What the Library Currently Provides (Real Features Only)

### 1. Core Utility Functions

* Hashing
* Random number generation
* Die rolls
* Lightweight helper logic

All of these are implemented and active in Library 2.0.

---

### 2. Unified JSON Dispatcher

The Library evaluates the `tool` value and routes the request to the appropriate sub-function inside the script.
This creates **one central DojoTool** instead of 20+ small ones.

---

### 3. Structured JSON Output

Every Library call returns:

```json
{
  "header": {...},
  "response": {...},
  "timestamp": "...",
  "input_echo": {...}
}
```

This makes the Library safe for:

* Friday
* Monastery agents
* The Summarizer
* n8n
* Any JSON-consuming automation

---

### 4. DojoTools Compliance

Library 2.0 conforms fully to the **DojoTools architecture**:

* Same call structure
* Same JSON schema
* Same debug format
* Same dispatcher rules

This ensures clean integration with **FileCabinet**, **CabinetAdmin**, **Index Tools**, and **Summarizers**.

---

### 5. Universal Tool Dispatch

If a caller routes a DojoTools action into the Library, the Library can:

* validate
* forward
* wrap
* or respond

...while preserving JSON safety.

This enables:

* High-level tool chaining
* Monastery-safe reasoning
* Complex workflow composition

*(No fictional or speculative commands are defined here — only real, implemented, or explicitly planned features.)*

---

## ✅ Integration Notes

The Library now acts as the **core DojoTool** for helper logic.
It consolidates all lightweight functionality required by:

* **Friday** (interactive AI layer)
* **Monastery agents** (reasoning layer)
* **Summarizers** (context synthesis)
* **FileCabinet / CabinetAdmin** (data layer)
* **n8n agents** (automation layer)

Versioned (e.g. `2.0.0`) and visible in **Home Assistant Developer Tools → Scripts**.

---

## ✅ Architecture Integration

The Library is part of the broader **ZenOS-AI core utility stack**:

| Layer              | Component                                         | Description                                               |
| ------------------ | ------------------------------------------------- | --------------------------------------------------------- |
| **Storage**        | [Cabinets & Drawers](../cabinets/cabinet_spec.md) | Hierarchical JSON stores for entities, AI, and metadata   |
| **Index**          | [Index System](../library/index_system.md)        | Unified label/query engine with recursion and event logic |
| **Execution**      | **Library 2.0** *(this file)*                     | Structured dispatcher for lightweight utility functions   |
| **Administration** | [CabinetAdmin Tools](../cabinets/admin_tools.md)  | Schema enforcement and maintenance                        |
| **Summarization**  | [Zen Summarizer](../zen_summarizer/readme.md)     | Context reduction and Kata generation                     |

The Library interacts with the **Index System** by invoking it as a sub-tool or macro reference (`~INDEX~`) inside workflows that need label resolution, ensuring a consistent data interface from utility level to semantic layer.

---

## ✅ Roadmap

| Phase   | Goal                            | Notes                                    |
| ------- | ------------------------------- | ---------------------------------------- |
| **2.1** | Add lightweight Cabinet helpers | cross-link drawer lookups to Index       |
| **2.2** | Capsule utilities               | integrate signing & verification helpers |
| **2.3** | Inline verification tools       | safety checks before cabinet writes      |
| **2.4** | Macro-driven context loaders    | partially implemented                    |
| **2.5** | Consolidate helper logic        | move small standalone tools into Library |

All future expansions will remain JSON-structured and DojoTools-compliant.

---

**Version:** 2.0.1
**Author:** Veronica (assistant to Nathan Curtis)
**License:** MIT
**Last Updated:** 2025-11-11
**Related Docs:** [Index System](../library/index_system.md), [Cabinet Spec](../cabinets/cabinet_spec.md), [Zen Summarizer](../zen_summarizer/readme.md)

---
