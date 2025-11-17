# **ZenOS-AI Scripts Directory**

*Operational DojoTools powering Friday’s cognition, storage, and reflexes.*

Welcome to the **DojoTools scripts directory** — the execution layer of ZenOS-AI.

This folder contains all executable Home Assistant scripts that implement Friday’s internal toolkits.
Each script follows the naming pattern:

```
zen_dojotools_<function>.yaml
```

Heavy documentation lives in `/scripts_docs/`.
This README gives a **light, navigable overview** of every tool in this directory.

---

# 🧱 **Core DojoTools (Ring-0)**

**These tools are the foundation of ZenOS-AI.**
Every other component — Friday, Veronica, Kronk, the Monastery, and all Katas — depends on these.

| Script                                 | Purpose                                                                                          |
| -------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **`zen_dojotools_index.yaml`**         | Entity + label index. Backbone of DojoTools. Required everywhere.                                |
| **`zen_dojotools_inspect.yaml`**       | Safe entity inspection: attributes, cabinet headers, stat eligibility.                           |
| **`zen_dojotools_manifest.yaml`**      | Master manifest for Cabinet volumes & drawers. Provides routing metadata.                        |
| **`zen_dojotools_identity.yaml`**      | Identity resolver + Zen-ID generator. GUIDs, personas, module lookup.                            |
| **`zen_dojotools_filecabinet.yaml`**   | File Cabinet manager — CRUD for drawers, volumes, metadata, integrity.                           |
| **`zen_dojotools_event_emitter.yaml`** | Emits structured `zen_event` telemetry. Breadcrumbs, traces, observability.                      |
| **`zen_dojotools_labels.yaml`**        | Label definitions + mapping utilities. Core routing feature. *(Spook only required for writes.)* |

**Why all of these are Core:**
They define how Friday *sees*, *labels*, *stores*, *retrieves*, *validates*, *classifies*, and *reports* information inside ZenOS-AI.

Without these, Friday is blind, mute, and unable to reason about the environment.

---

# 📦 **Cabinet & Storage Extensions**

| Script                             | Purpose                                                               |
| ---------------------------------- | --------------------------------------------------------------------- |
| `zen_admintools_cabinetadmin.yaml` | Deep integrity checking, repairs, and formatting for Cabinet volumes. |

**Note:**
This is **not Core** — it’s a heavy admin tool, not used in normal operation.

---

# 🧠 **Identity, Library & Metadata Extensions**

| Script           | Purpose                                               |
| ---------------- | ----------------------------------------------------- |
| *Upcoming Tools* | Library 2.0, persona capsules, prompt loader systems. |

Identity has already been moved into **Core** above.

---

# 📅 **Personal Assistant Tools**

| Script                        | Purpose                                                               |
| ----------------------------- | --------------------------------------------------------------------- |
| `zen_dojotools_calendar.yaml` | Multi-calendar engine. Unified read/create/update/delete with safety. |
| `zen_dojotools_todo.yaml`     | To-Do & Shopping manager. Integrates HA Todo, Grocy, Mealie.          |

---

# 🎶 **Media Tools**

| Script                            | Purpose                                              |
| --------------------------------- | ---------------------------------------------------- |
| `zen_dojotools_music_search.yaml` | Music Assistant search with entity + label matching. |

---

# 🧹 **Summarization Tools — The Kata System**

| Script                                | Purpose                                                                |
| ------------------------------------- | ---------------------------------------------------------------------- |
| `zen_dojotools_ninja_summarizer.yaml` | Stage 1 summarizer — converts raw triggers to fine-grained Katas.      |
| `zen_dojotools_supersummary.yaml`     | Stage 2 summarizer — higher-level narrative + attention-weighted meta. |

---

# 🛠 **Admin & Maintenance Tools**

| Script                              | Purpose                                                    |
| ----------------------------------- | ---------------------------------------------------------- |
| `zen_admintools_kungfu_writer.yaml` | Loads initial Kung Fu component definitions into Cabinets. |
| `zen_admintools_cabinetadmin.yaml`  | Repairs Cabinets (formatting, normalization, validation).  |

---

# 🔧 Installation Notes

1. **Keep filenames exactly as-is.**
   Renaming requires updates to Index, Manifest, Cabinet, and redirectors.

2. Reload HA scripts after changes:
   **Settings → Developer Tools → YAML → Reload Scripts**

3. **Always install kits in groups:**

   * **Core Kit** (Index, Inspect, Labels, Manifest, Identity, FileCabinet, EventEmitter)
   * **Cabinet Kit** (Manifest, FileCabinet, Redirector Automation)
   * **Summarizer Kit** (Ninja + SuperSummary)

4. **Spook required for label writes.**
   Reads work without it. Updates do not.

---

# 🧩 Development Philosophy

* Tools must be **modular**, **atomic**, and **additive**
* No hidden dependencies — declare them explicitly
* All operations should be observable
* All scripts should be safe for LLMs to use

If you’re contributing:
Start with the Index, follow the naming pattern, and document every output structure.

---

# ☯️ “Welcome to the Dojo”

If the Cabinets are the memory shelves
and the Monastery is the mind,

**these scripts are the hands.**

Be precise.
Be modular.
Be kind to your future self.
