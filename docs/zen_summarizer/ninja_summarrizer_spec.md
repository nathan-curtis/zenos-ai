# **Zen DojoTools Ninja Summarizer**

**Subsystem:** ZenOS-AI → DojoTools → Summarization Pipeline
**Version:** 4.1.1
**Authors:** Nathan & Veronica (The Monastery Collective)

The **Ninja Summarizer** is Friday’s first-stage summarizer — a modular AI agent that gathers data from the Dojo, resolves labels through the Index, runs Library commands, and generates structured **Kata summaries** stored in the **Kata Cabinet** for higher-level reasoning by the **High Priestess**.

---

## 🧘 Purpose

The Ninja Summarizer transforms raw Home Assistant state into structured, schema-compliant insight.
It’s designed to:

* Collect contextual input from **Dojo**, **Household**, and **Kata** Cabinets
* Run **Index** queries for relevant entities
* Optionally call **Library** utilities for diagnostics
* Pass everything to a **Monk** (LLM worker) for reasoning
* Write results into the **Kata Cabinet**
* Return a JSON payload to downstream automations

This process ensures Friday’s cognition remains compact, modular, and data-rich — keeping reasoning local while maintaining global awareness.

---

## 🏯 Architectural Flow

```
 ┌──────────────────────────────────────────────────────────────┐
 │                      ZenOS–AI Core                           │
 │        (Dojo • Monastery • Cabinets • AI Layer)              │
 └──────────────────────────────────────────────────────────────┘
                 │
                 ▼
     ┌──────────────────────┐
     │  Ninja Summarizer    │   ← per-component summarizer
     └──────────────────────┘
                 │
                 ▼
     ┌──────────────────────┐
     │  Monk Runner (LLM)   │   ← GPT-OSS-20B / Kronk
     └──────────────────────┘
                 │
                 ▼
     ┌──────────────────────┐
     │  Kata Cabinet Drawer │   ← structured JSON storage
     └──────────────────────┘
```

---

## ⚙️ Operational Overview

| Step  | Action                         | Description                                                         |
| ----- | ------------------------------ | ------------------------------------------------------------------- |
| **1** | Cabinet discovery              | Identify the Dojo, Household, and Kata cabinets.                    |
| **2** | Drawer resolution              | Locate the target component’s drawer.                               |
| **3** | Metadata extraction            | Gather `version`, `friendly_name`, `label`, `command`, `tool`, etc. |
| **4** | Index run *(optional)*         | Run `dojotools_zen_index` for entity sets.                          |
| **5** | Library call *(optional)*      | Run defined command through the Library interpreter.                |
| **6** | Monk execution                 | Pass merged context to the local LLM (Monk).                        |
| **7** | Validation                     | Enforce schema and error-check JSON.                                |
| **8** | Cabinet writeback *(optional)* | Write finalized summary to the Kata Cabinet.                        |
| **9** | Return payload                 | Emit full structured results to the caller.                         |

---

## 📦 Cabinet Logic

| Cabinet               | Source        | Purpose                                         |
| --------------------- | ------------- | ----------------------------------------------- |
| **Dojo Cabinet**      | Authoritative | Component definitions, schema, and commands     |
| **Household Cabinet** | Contextual    | Local state, overrides, and preferences         |
| **Kata Cabinet**      | Persistent    | Per-component summaries for Monastery reasoning |

Resolution follows the chain: **Dojo → Household → Kata**.

---

## 🧺 **Collection Phase — Context Assembly & Preflight**

The **Collection Phase** is the opening act of the Ninja Summarizer — the part that gathers everything a Monk could possibly need before a single line of inference begins.
Think of it as Friday’s data mise en place: every cabinet opened, every drawer inspected, every ingredient labeled before the cooking starts.

---

### ⚙️ **Overview**

The Summarizer starts by resolving cabinet locations and loading the contents of the relevant drawers.
It pulls metadata from the **Dojo Cabinet** (canonical definitions) and **Household Cabinet** (context overrides) to construct a merged working snapshot for the target component.

This snapshot defines:

* What subsystem we’re summarizing
* What it’s called and versioned as
* What Index and Library operations apply
* Whether the component is even “on” (via `master_switch`)

Every field gathered here becomes the basis for all downstream reasoning.

---

### 🧭 **Sequence Summary**

| Step  | Variable                                            | Purpose                                                                |
| ----- | --------------------------------------------------- | ---------------------------------------------------------------------- |
| **1** | `dojo_cabinet`, `household_cabinet`, `kata_cabinet` | Locate key cabinet entities via label lookup.                          |
| **2** | `dojo_drawer`                                       | Pull component definition JSON from Dojo Cabinet variables.            |
| **3** | `household_drawer`                                  | Pull override or live state JSON from Household Cabinet.               |
| **4** | `version`, `friendly_name`, `master_switch`         | Merge base and override metadata (Dojo preferred, Household fallback). |
| **5** | `index_command`                                     | Pull the component’s assigned Index label, if any.                     |
| **6** | `library_command`                                   | Pull the component’s Library or interpreter command, if defined.       |
| **7** | `component_summary`                                 | Load the component’s previous summary or self-description.             |
| **8** | `tool`                                              | Record any declared external or automation tool reference.             |
| **9** | `index_result`                                      | Optionally populate via live Index query when a label is defined.      |

---

### 🗄️ **How It Works**

1. **Cabinet Resolution**
   The script resolves each cabinet by its label:

   ```jinja
   dojo_cabinet = label_entities('Zen Dojo Cabinet') | first
   household_cabinet = label_entities('Zen Default Household Cabinet') | first
   kata_cabinet = label_entities('Zen Kata Cabinet') | first
   ```

   This guarantees the summarizer always targets the correct volume, even if multiple exist.

2. **Drawer Lookup**
   Using `component_slug` (the lowercase, slugified name of the Kung Fu component), it retrieves matching drawers from both cabinets:

   ```jinja
   dojo_drawer = state_attr(dojo_cabinet, 'variables')[component_slug]
   household_drawer = state_attr(household_cabinet, 'variables')[component_slug]
   ```

   The Dojo drawer defines *what* the component is.
   The Household drawer defines *how it’s currently behaving.*

3. **Metadata Merge**
   Each key attribute is extracted with a simple but deliberate fallback order:

   ```jinja
   {{ ( dojo_drawer.version ) or ( dojo_drawer_value.version ) }}
   ```

   Dojo values always win unless explicitly overridden.

4. **Dynamic Field Assembly**
   The collector builds runtime variables such as:

   * `index_command` → label for entity discovery
   * `library_command` → diagnostic or system command
   * `component_summary` → last known contextual description
   * `tool` → linked automation helper

5. **Conditional Index Run**
   If `index_command` is non-empty, the Summarizer triggers:

   ```yaml
   service: script.dojotools_zen_index
   data:
     operator: AND
     expand_entities: true
     label_1: "{{ index_command }}"
   ```

   The result populates `zen_index_result`, which becomes `index_result` — the first live data packet attached to the component’s summarization context.

---

### 🧩 **Result of the Collection Phase**

By the time this phase completes, the Ninja Summarizer holds a fully prepared dataset:

```json
{
  "component_slug": "hot_tub",
  "dojo_source": "Zen Dojo Cabinet",
  "household_source": "Zen Default Household Cabinet",
  "version": "1.3.0",
  "friendly_name": "Hot Tub System",
  "master_switch": "on",
  "index_command": "hot_tub",
  "library_command": "",
  "component_summary": "...",
  "tool": "script.hot_tub_manager",
  "index_result": { ... entity data ... }
}
```

This JSON block is the **review_data** base for later phases — fed into prompt construction (detailed in its own spec) before being passed to the **Monk Runner**.

---

## 🧘 Monk Execution

```
service: ai_task.generate_data
data:
  task_name: "<Kung Fu Component> Monk"
  instructions: <full prompt>
  entity_id: ai_task.gpt_oss_20b_local_ai_task
```

The **Monk** processes all merged context and returns structured JSON (`kata_summary`) validated against the Kata schema.

---

## 🗄️ Cabinet Writeback (optional)

If configured:

```yaml
post_to_kata_cabinet: true
```

Then:

1. Delay 250 ms for stability.
2. Write `kata_summary` into the **Kata Cabinet** under `<component_slug>`.
3. Label the drawer with `<index_label>` + `"Ninja Summary"`.
4. Timestamp and enforce write integrity.

Result:

```yaml
drawer: <component_slug>
value: <kata_summary>
set_timestamp: true
force_action: true
key: <component_slug>
label_input: [ "<index_label>", "Ninja Summary" ]
```

---

## 🧾 Output Schema

```json
{
  "status": "success",
  "monk": { ... },
  "post": true,
  "dojo_drawer": { ... },
  "filecabinet_write": { ... }
}
```

Every operation is traceable for auditing and Monastery reflection.

---

## 🧩 Design Intent

* Produce **consistent per-component context packets**
* Separate **real-time I/O** from **reasoning loops**
* Feed clean summaries into the **High Priestess**
* Enable composable components (labels → Index, commands → Library)
* Keep Friday’s working memory fast, lean, and local

---

## 🛡 Guarantees

| Type | Guarantee                                   |
| ---- | ------------------------------------------- |
| ✅    | JSON-compliant structured output            |
| ✅    | Schema-validated Kata summaries             |
| ✅    | Deterministic, repeatable behavior          |
| ✅    | Optional cabinet persistence                |
| ✅    | Integration with Index + Library subsystems |

---

## 🧩 Extensibility

* Add new fields to drawer templates
* Register new Library commands
* Create specialized Monks (HVAC, Water, Security…)
* Extend Kata schemas
* Define new High Priestess aggregation rules
