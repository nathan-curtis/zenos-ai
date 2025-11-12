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

## 🧩 Index & Library Integration

**Index Mode**
If the drawer defines `label: "lights"`, the Summarizer triggers:

```yaml
service: script.dojotools_zen_index
data:
  operator: AND
  expand_entities: true
  label_1: "lights"
  timeout: 2
```

Result:

```json
{ "command": "lights", "result": { ... } }
```

**Library Mode**
If the drawer defines `command: "get system status"`, the Summarizer runs:

```jinja
{{ command_interpreter.interpreter('get system status') }}
```

Result:

```json
{ "command": "get system status", "result": "<interpreter output>" }
```

Both results feed directly into the summarization context.

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
