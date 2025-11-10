# **Zen DojoTools Ninja Summarizer — Technical Specification**

**Version:** 4.1.1
**Subsystem:** ZenOS-AI → DojoTools → Summarization Pipeline
**Authors:** Nathan & Veronica (The Monastery Collective)

---

# **1. Purpose**

The **Ninja Summarizer** is the core automated component summarizer for ZenOS-AI.
It extracts all known state for a single **Kung Fu Component**, generates a **structured inference summary** using a Monk (LLM worker), and optionally posts the result to the **Kata Cabinet** for downstream reasoning.

Its responsibilities:

* Collect contextual input from the Dojo Cabinet & Household Cabinet
* Execute Index routines (if the component defines them)
* Execute Library routines / commands (if defined)
* Build a strict structured prompt using the kata template schema
* Pass all context and instructions to a Monk (LLM inference worker)
* Receive structured JSON back (kata_summary)
* Optionally write this summary into the Kata Cabinet
* Return a full diagnostic payload to the caller

This forms the backbone of your **component-level summarization loop**, allowing the High Priestess (supersummarizer) to build house-level insight on top of stable 1:1 component summaries.

---

# **2. Architectural Positioning**

```
 ┌───────────────────────────────────────────────────────────────┐
 │                         ZenOS–AI                               │
 │                 (Dojo, Monastery, Cabinets)                    │
 └───────────────────────────────────────────────────────────────┘
                 │
                 ▼
     ┌──────────────────────┐
     │ Ninja Summarizer     │  ← the automation we are documenting
     │ (Per-component agent)│
     └──────────────────────┘
                 │
                 ▼
     ┌──────────────────────┐
     │ Monk Runner (LLM)    │  ← local GPT-OSS-20B or Kronk
     └──────────────────────┘
                 │
                 ▼
     ┌──────────────────────┐
     │ Kata Cabinet Drawer  │  ← persistent structured JSON
     └──────────────────────┘
```

The Ninja Summarizer is the *first-stage summarizer*.
The Monastery handles computation.
The Kata Cabinet stores persistent per-component summaries.
The High Priestess composes the supersummaries from there.

---

# **3. High-Level Operational Flow**

1. **Identify relevant cabinets**
   → Dojo Cabinet
   → Household Cabinet
   → Kata Cabinet

2. **Identify component drawer**
   Locate drawer named `<kung_fu_component_id>` in the Dojo or Household.

3. **Resolve metadata fields** from the drawer:

   * `version`
   * `friendly_name`
   * `master_switch`
   * `label` (used as Index trigger)
   * `command` (used for Library calls)
   * `component_summary`
   * `tool`

4. **Conditional Index Run**
   If the drawer defines a label, run `dojotools_zen_index`.

5. **Conditional Library Command Execution**
   If the component defines a library command, run the `command_interpreter`.

6. **Build the review_data + prompt package**
   Merge:

   * Query instructions
   * Kata schema
   * Example block
   * Resolved context for the component
   * Index output
   * Library output

7. **Run Monk via ai_task**
   Task is named dynamically:
   `"{{ kung_fu_component_id | title }} Monk"`

8. **Parse output** into `kata_summary`.

9. **Optional: Write to Kata Cabinet**
   If `post_to_kata_cabinet == true`:
   → Writes to drawer named `<component_slug>` in Kata Cabinet
   → Labels the drawer with both the component’s index label *and* `"Ninja Summary"`

10. **Return a structured final response** to caller.

---

# **4. Detailed Sequence Specification**

Below is a deeper inspection of each operation.

---

## **4.1 Cabinet Resolution**

The Ninja Summarizer identifies three cabinets:

| Name                | Source                          | Purpose                                   |
| ------------------- | ------------------------------- | ----------------------------------------- |
| `dojo_cabinet`      | `Zen Dojo Cabinet`              | Component definitions, commands, metadata |
| `kata_cabinet`      | `Zen Kata Cabinet`              | Storage of resulting summaries            |
| `household_cabinet` | `Zen Default Household Cabinet` | Default component preferences/overrides   |

Resolution uses HA’s label-based entity discovery.

---

## **4.2 Drawer Resolution**

A component drawer can live in:

1. **Dojo Cabinet** (preferred, authoritative)
2. **Household Cabinet** (fallback, local overrides)

The Summarizer extracts fields from either source:

* `version`
* `friendly_name`
* `master_switch`
* `label` → *Index operation*
* `command` → *Library operation*
* `component_summary`
* `tool`

This supports the “inherit override fallback” hierarchy.

---

## **4.3 Index Operation (optional)**

If the component defines:

```
label: "lights"
```

Then the Summarizer runs:

```
script.dojotools_zen_index
```

with:

```yaml
operator: AND
expand_entities: true
label_1: "{{ index_command }}"
timeout: 2
```

The result becomes:

```json
{
  "command": "<label>",
  "result": { ... }
}
```

This enables components to self-index their dependent entities.

---

## **4.4 Library Operation (optional)**

A component can define:

```
command: "get system status"
```

The Summarizer calls:

```jinja
{{ command_interpreter.interpreter(library_command) }}
```

The output is captured as:

```json
{
  "command": "<command-string>",
  "result": "<LLM or interpreter output>"
}
```

Thus allowing components to call into the Library subsystem to compute diagnostics.

---

# **4.5 Prompt Construction**

The Summarizer constructs:

### **query**

Default natural-language instructions for the Monk (LLM).
This describes:

* Goals of summarization
* Expected structure
* Requirements
* How “Cliff Notes” must be produced
* Error-handling directives
* Callouts for missing data
* Instructions on short-term, near-term, and now-state events
* How the High Priestess will supersummarize the results later
* Rules for confidence scoring and uncertainty

### **structure**

Pulled from:

```
kata_template.structure
```

This is the strict JSON schema the Monk **must** return.

### **example_data**

A valid example of the schema with real-world content.

### **review_data**

Context packet including:

* Cabinets involved
* Component drawer content
* Output of Index routine
* Output of Library routine
* Any component-defined tool
* Component slug

### **supplemental_instructions**

Optional append-only content.

All packaged into:

```json
{
  "query": "...",
  "structure": { ... },
  "example_data": { ... },
  "review_data": { ... },
  "supplemental_instructions": "..."
}
```

This forms the final prompt fed to the Monk Runner.

---

# **4.6 Monk Runner Execution**

```
ai_task.generate_data
```

Parameters:

* `task_name`: `<Kung Fu Component> Monk`
* `instructions`: full JSON prompt
* `entity_id`: `ai_task.gpt_oss_20b_local_ai_task`

The local AI model (Monk) is invoked.
The output is guaranteed to be **structured JSON** matching `kata_template`.

Result stored in:

```
monk_response.data
```

This becomes:

```
kata_summary
```

---

# **4.7 Write to Kata Cabinet (optional)**

If:

```
post_to_kata_cabinet: true
```

Then:

1. Delay 250ms (stability window)
2. Write to the Kata Cabinet:

```
drawer = <component_slug>
value = <kata_summary>
set_timestamp = true
force_action = true
key = component_slug
label_input = (index_command, "Ninja Summary")
```

This enforces:

* Timestamping
* Label-based classification
* Identity-preserving writes
* Compatibility with FileCabinet CRUD logic

---

# **4.8 Final Output Structure**

Returned to the caller:

```json
{
  "status": "success",
  "monk": { ... monk_response ... },
  "post": <boolean>,
  "dojo_drawer": { ... drawer resolved ... },
  "filecabinet_write": { ... if posted ... }
}
```

This ensures full introspection of the summarization cycle.

---

# **5. Why This System Exists**

### **5.1 To produce consistent per-component context packets**

Each Kung Fu component manages a domain (water, HVAC, energy, security…).
Each needs:

* state interpretation
* event detection
* anomaly recognition
* schedule inference
* short-term planning
* risk classification

### **5.2 To separate real-time thinking from long-term reasoning**

The Monk (LLM) handles reasoning.
The Summarizer handles:

* cabinet IO
* context assembly
* schema enforcement
* classification
* routing
* event-driven triggers

### **5.3 To allow the High Priestess supersummarizer to reason globally**

The Ninja Summarizer creates stable, consistent objects in Kata Cabinet.
These are merged by the High Priestess into a complete house-state overview.

### **5.4 To make components composable**

Any new component can:

* define label → auto-index
* define command → auto-library-call
* define schema → autotype into Kata Cabinet
* rely on Summarizer to unify everything

### **5.5 To reduce prompt length for Friday**

The Monastery holds Katas.
Friday holds only:

* recent Katas
* supersummaries
* context gleaned from Monastery pathways

Thus Friday remains fast and local.

---

# **6. Design Constraints & Guarantees**

### Guaranteed:

* Structured JSON output
* Schema-compliant Katas
* Low-latency inference (15–60 seconds)
* Stack-safe (queued mode, max 2)
* Idempotent (same inputs → same output)
* Index + Library integration
* Error propagation in `error` field of schema

### Not Guaranteed:

* Component drawer correctness (user-defined)
* External tool availability
* Network model loading speed
* Perfect reasoning (Monk is allowed uncertainty)

---

# **7. Extensibility**

You can extend the system by:

* Adding new fields in drawer templates
* Registering new library commands
* Adding new index labels
* Defining custom schemas in Kata Template
* Creating new High Priestess supersummarization rules
* Turning Monks into specialist agents (HVAC Monk, Water Monk, Security Monk, etc.)

---
