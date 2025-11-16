# ZenOS-AI Scripts Directory  
Welcome to the **DojoTools** scripts directory — the operational heart of Friday’s ZenOS-AI.

This folder contains all executable Home Assistant scripts that implement the modular toolkits described in the main project README. Each script here represents a self-contained “Dojo Tool,” following the naming convention:

`zen_dojotools_<function>.yaml`

Tools are grouped by capability and depend on the Zen Index, Cabinet, and Library systems to operate correctly.

---

## 🧱 Directory Overview

### **Core DojoTools**
Foundational tools required for entity indexing, inspection, observability, and cabinet routing.

| Script | Purpose |
|-------|---------|
| `dojotools_zen_index.yaml` | Core label-aware entity index. Required by nearly every other tool. |
| `dojotools_zen_inspect.yaml` | Inspection utility for reviewing entities, attributes, states, and label mappings. |
| `zen_dojotools_manifest.yaml` | Drawer & Volume manifest used by the File Cabinet system. |
| `zen_dojotools_labels.yaml` | Label definitions & mapping utilities (requires **Spook** for full label operations). |
| `zen_dojotools_event_emitter.yaml` | Emits structured ZenOS-AI events for debugging, trace reconstruction, and breadcrumbs. |

---

## 📦 Cabinet & Storage Tools

| Script | Purpose |
|-------|---------|
| `zen_dojotools_filecabinet.yaml` | File Cabinet manager—create, edit, update drawers/volumes, manage metadata. |
| `zen_admintools_cabinetadmin.yaml` | Cabinet repair & formatting utility. Requires a valid FES sensor for integrity checks. |

**Dependencies:**  
- Zen Index Kit  
- Manifest  
- Spook (optional but strongly recommended for label management)

---

## 🧠 Identity, Library, and Metadata Tools

| Script | Purpose |
|-------|---------|
| `zen_dojotools_identity.yaml` | Identity resolver + prompt builder (RC). Handles GUID lookup, persona metadata, and Zen-ID struct output. |
| `zen_dojotools_manifest.yaml` | Core manifest mapping for Cabinets & Volumes. |
| *Upcoming Tools* | Extended Library 2.0, persona capsules, prompt loader macros. |

**Status:**  
These tools are in active development and may evolve as ZenOS-AI moves toward version **1.2+**.

---

## 📅 Personal Assistant Tools

| Script | Purpose |
|-------|---------|
| `zen_dojotools_calendar.yaml` | Multi-domain calendar tool (HA native calendars, events, reminders, ICS parsing). |
| `zen_dojotools_todo.yaml` | To-Do & Shopping manager. Works with HA Todo Lists and Grocy/Mealie bridges. |

**Dependencies:**  
- Zen Index Kit

---

## 🎶 Media Tools

| Script | Purpose |
|-------|---------|
| `zen_dojotools_music_search.yaml` | Music Assistant search tool supporting artist, album, and playlist matching. |

**Dependencies:**  
- Zen Index Kit  
- Music Assistant Voice Tools (required)

---

## 🧹 Summarization Tools — The Kata System

| Script | Purpose |
|-------|---------|
| `zen_dojotools_ninja_summarizer.yaml` | Stage 1: Event-driven Kata summarization (fine-grained). |
| `zen_dojotools_supersummary.yaml` | Stage 2: Higher-level attention-weighted summary. |

**Dependencies:**  
- Zen Index  
- File Cabinet  
- Monastery (local or cloud inference)

---

## 🛠 Admin & Maintenance Tools

| Script | Purpose |
|-------|---------|
| `zen_admintools_kungfu_writer.yaml` | Loads initial Kung Fu component definitions into Cabinets. |
| `zen_admintools_cabinetadmin.yaml` | Repairs, normalizes, and formats existing Cabinet volumes. |

---

# 🔧 Installation Notes

1. **Place all scripts in this directory unchanged.**  
   Renaming files will require updates to the Dojo Index, Cabinet Manifest, and any referencing scripts.

2. **Reload Home Assistant scripts** after adding or modifying tools:  
   `Settings → Developer Tools → YAML → Reload Scripts`

3. **Load Kits Together**  
   Many tools are interdependent. Always install:  
   - **Zen Index Kit** (Index, Inspect, Labels)  
   - **File Cabinet Kit** (Manifest, Cabinet, Redirector Automation)  
   - **Summarizer Kit** (Ninja + SuperSummary)  
   - Optional Kits: Calendar, Todo, Media

4. **Spook is required** for label write operations.  
   Without Spook, ZenOS-AI can read labels but cannot reliably manage them.

---

# 🧩 Development Philosophy

- Tools are **modular and additive**.  
- No script should assume another tool exists unless declared as a dependency.  
- All operations should be observable in HA logs.  
- Friday, Kronk, and the High Priestess use these tools collaboratively for context, summarization, and memory.

If you’re contributing:
- Start with the Index.
- Follow the naming pattern.
- Document your schema.
- Kronk appreciates your service.

---

# 🤝 Contributing

Pull requests are welcome.  
If you're unsure where your script fits or what dependencies it requires, open a Discussion—Friday (or Veronica) will guide you.

---

# ☯️ “Welcome to the Dojo”

This directory is where ZenOS-AI learns to think, organize, remember, and act.  
If the Cabinets are the memory shelves and the Monastery is the mind,  
**these scripts are the hands.**

Be gentle.  
Be curious.  
Be modular.
