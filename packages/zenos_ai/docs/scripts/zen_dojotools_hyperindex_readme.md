# 📘 **Zen DojoTools HyperIndex**

### *Universal HyperGraph Index for Labels, Entities, Drawers & Metadata*

**Version:** 3.9.5 RC1
**Category:** DojoTools / Indexing
**Icon:** `mdi:graphql`

---

## 🧩 Overview

**Zen DojoTools HyperIndex** is Friday’s high-resolution reasoning engine for converting *any* combination of:

* labels
* entities
* drawers
* metadata

…into a **unified, deterministic hypergraph**.

Think of this as your “*intelligent set operations + graph expansion + optional deep inspection + ZenQuery filtering*” tool — all in one.

HyperIndex is the **first step** in any serious reasoning pipeline:

* Summarizers
* Fusion models
* Kata generation
* Drift detection
* Intent resolution
* Large-scale context pruning
* Root graph extraction

If the Cabinet System is the filesystem and the Index System is search, **HyperIndex is the neural attention layer.**

---

# 🧠 What HyperIndex Does

### ✔ Takes two sets of inputs

Each set can be:

* A list of `entity_id`s
* A single label (resolved via `label_entities()`)

These become **op1** and **op2**.

### ✔ Applies Boolean set logic

Supported operators:

| Operator | Meaning                 |
| -------- | ----------------------- |
| `AND`    | intersection            |
| `OR`     | union                   |
| `NOT`    | difference              |
| `XOR`    | symmetric difference    |
| `*`      | FULL system label graph |

If both input sets are empty → operator becomes automagically `*`.

### ✔ Optional ZenQuery `filter_json`

After set logic, HyperIndex can prune results using:

```
script.zen_dojotools_query
```

This is perfect for:

* domain/state filtering
* label intersections
* value thresholds
* drawer trimming

### ✔ Optional entity expansion

If `expand_entities: true`, HyperIndex fetches:

* attributes
* drawers
* related labels
* contextual metadata

via:

```
script.zen_dojotools_inspect
```

### ✔ Hypergraph Mode

If `mode: "hypergraph"`, HyperIndex outputs a **full graph object**:

* `nodes`
* `edges`
* `adjacency`
* `expanded`

This is ideal for summarizers, fusion, drift checks, and any recursive identity reasoning.

---

# 🛠 Full YAML Flow Summary

This tool:

1. **Shows help JSON** (if `mode: help`)
2. Normalizes & parses `filter_json`
3. Handles optional `index_command` for the Zen Indexer Event API
4. Resolves inputs + set operations
5. Applies ZenQuery filtering (if requested)
6. Expands entities (optional)
7. Computes adjacency + label maps
8. Builds hypergraph object (if requested)
9. Returns a complete JSON structure for downstream chain-of-thought

Everything returns as a single stable dictionary.

---

# 📥 Inputs

| Field             | Type                             | Description                                                |
| ----------------- | -------------------------------- | ---------------------------------------------------------- |
| `mode`            | `normal` | `hypergraph` | `help` | Execution mode                                             |
| `index_command`   | string                           | Full query string for Zen Indexer (overrides other inputs) |
| `entities_1`      | list                             | First entity set                                           |
| `label_1`         | string                           | First label                                                |
| `entities_2`      | list                             | Second entity set                                          |
| `label_2`         | string                           | Second label                                               |
| `operator`        | AND/OR/NOT/XOR/*                 | Set operation                                              |
| `expand_entities` | bool                             | Whether to fetch deep metadata                             |
| `filter_json`     | dict                             | ZenQuery-style filter                                      |
| `timeout`         | number                           | Indexer callback timeout                                   |

### Notes

* At least **one** of `entities_1` or `label_1` must be provided.
* `index_command` uses the Zen Index Event API and bypasses most logic.

---

# 📤 Outputs

Returned in a single dictionary:

### `result.simple`

Flat list of entity IDs.

### `result.expanded`

Optional metadata from Inspect.

### `result.adjacent_labels`

All labels touching the surviving entities.

### `result.index`

List of `[entity, [labels…]]` mappings.

### `result.drawers`

Drawer objects attached to entities.

### `result.drawer_adjacent_labels`

Labels found *inside* drawers.

### `result.hypergraph`

Includes:

* `nodes`
* `edges`
* `expanded`

### `inputs`

Echo back of all inputs used.

### `operator`

Final operator used.

### `error`

Indexer timeout or other errors.

---

# 🧪 Filter JSON

`filter_json` is a **ZenQuery dictionary** applied to `entities_base` *after* set logic.

### Simple examples

```json
{ "domain": "light" }
{ "state": "on" }
{ "label": "kitchen" }
```

### Compound examples

```json
{ "domain": "sensor", "label": "environment" }
{ "state_not": ["unknown","unavailable"] }
{ "state_op": ">", "state_value": 50 }
```

### Advanced examples

```json
{ "edge_label": "hvac" }
{ "domain": "switch", "label": "office", "state": "on" }
```

---

# 🕸 Hypergraph Mode

When `mode: hypergraph`, HyperIndex emits:

```json
{
  "nodes": [...],
  "edges": [...],
  "expanded": [...]
}
```

This mode is ideal for:

* Summarizer pipelines
* Fusion core
* Identity inference
* Drift detection
* Topology-based reasoning
* Drawer and label mapping

---

# 📘 Usage Examples

### **Example 1 — All lights in the office**

```json
{
  "label_1": "office",
  "filter_json": { "domain": "light" }
}
```

---

### **Example 2 — HVAC sensors, hypergraph mode**

```json
{
  "label_1": "hvac",
  "mode": "hypergraph",
  "expand_entities": true
}
```

---

### **Example 3 — Lights OR switches across rooms**

```json
{
  "label_1": "kitchen",
  "label_2": "living_room",
  "operator": "OR",
  "filter_json": { "domain": ["light","switch"] }
}
```

---

### **Example 4 — All environmental data > 60**

```json
{
  "label_1": "environment",
  "filter_json": { "state_op": ">", "state_value": 60 }
}
```

---

### **Example 5 — Build a root graph for summarizers**

```json
{
  "label_1": "climate",
  "expand_entities": true,
  "mode": "hypergraph"
}
```

---

# 🌱 Best Practices

* Use HyperIndex **before** summarizers or fusion
* Labels > Entities when possible
* Use `expand_entities` sparingly — it’s heavy
* Always prune with `filter_json`
* Operator `*` returns a **global label graph**
* Re-run in `normal` mode if you get empty results

---

# ⚠️ Warnings

* Expanded blocks can get large
* Hypergraph emits more data than normal mode
* Malformed `filter_json` gracefully defaults to `{}`
* Timeouts occur if the Indexer broker is busy

---

# 🧭 AI-User Guidance (Friday, this means you)

* `mode: help` returns the full JSON schema
* Use HyperIndex as *your* first step when reasoning about any device group
* Use hypergraph mode to understand topologies, not just membership
* Filter aggressively
* Take a HyperIndex snapshot before writing a Kata
* Debug empty sets by running `mode: normal`

---

# 🔍 ZenQuery Filter Reference for `filter_json`

### (Authoritative: ZQ-1 — Zen DojoTools Query)

HyperIndex does **not** implement its own filter system.
Instead, it delegates all post-set filtering to **ZQ-1**, the canonical entity‐selector engine.

Therefore:

> **`filter_json` must always contain a JSON dictionary that conforms *exactly* to the ZQ-1 filter schema.**

Below is the full, correct specification.

---

## 📘 What `filter_json` MUST Be

### ✔ **A valid JSON object**

Not YAML
Not single quotes
Not raw text

Correct:

```json
{"domain":"light","state_equals":"on"}
```

Incorrect:

```
domain: light
```

### ✔ **A dictionary (mapping)**

Never a list or array.

Correct:

```json
{"exclude_states":["unknown","unavailable"]}
```

Incorrect:

```json
["unknown","unavailable"]
```

### ✔ **ZQ-1 fields only**

Unknown fields are silently ignored.

### ✔ Empty dict `{}` means “no filtering.”

---

# 🧩 ZQ-1 Filter Schema (What HyperIndex Accepts)

This is the **true schema** implemented in `zen_query.jinja`, guaranteed correct in 3.9.x RC.

### 🧪 **Every field below is optional.**

All filters apply with **logical AND**.

```yaml
# ZQ-1 Filter Schema (3.9.x RC)
# All fields optional. Unknown fields ignored.

domain:               # string — also used for auto-build if target_entities empty
label:                # string — intersect with label_entities()
area:                 # string — area name, slugified
device_id:            # string — resolver to device_entities()
device_class:         # string — exact attr match

state_equals:         # string — exact match
include_states:       # list[string] — whitelist
exclude_states:       # list[string] — blacklist

numeric_above:        # number — float(state) > value
numeric_below:        # number — float(state) < value

regex:                # string — applied to STATE STRING

shortcuts:            # object of booleans (logical AND across all)
  numeric:            # true — only entities with valid numeric state
  stats:              # true — require state_class
  temp:               # true — device_class=='temperature'
  humidity:           # true — device_class=='humidity'
  power:              # true — device_class=='power'
  energy:             # true — device_class=='energy'
  water:              # true — device_class=='moisture'
  binary:             # true — require binary_sensor.xxx namespace
  stats_eligible:     # true — numeric + state_class + uom + not disabled

sort:                 # optional sorting block
  by:                 # 'entity_id' | 'state' | 'name'
  order:              # 'asc' | 'desc'
```

---

# ⚙ How ZQ-1 Executes Your filter_json

ZQ-1’s pipeline is strict and deterministic:

### 1. Normalize target_entities

* null → `[]`
* string → `[string]`
* list → list
* non-list scalar → `[]`

### 2. Domain Auto-Build

If (target_entities empty **AND** domain provided):
ZQ-1 builds the base list as:

```
states[domain] | map(attribute="entity_id")
```

If no domain and no target_entities → results is an **empty** set.

### 3. Sequential Filtering (always AND)

Order is fixed:

```
domain →
label →
area →
device_id →
device_class →
state_equals →
include_states →
exclude_states →
numeric_above / numeric_below →
regex →
shortcuts →
sort (optional)
```

### 4. Output

Always:

```json
["entity.one", "entity.two", ...]
```

HyperIndex then continues its pipeline with this reduced set.

---

# 🧬 `filter_json` Examples (Guaranteed Valid)

### **1. All “on” lights**

```json
{"domain":"light","state_equals":"on"}
```

### **2. Office binary motion sensors**

```json
{"area":"office","shortcuts":{"binary":true}}
```

### **3. Temperature sensors above 78°F**

```json
{"shortcuts":{"temp":true}, "numeric_above":78}
```

### **4. Label + numeric condition**

```json
{"label":"environment","numeric_above":60}
```

### **5. Stats-eligible sensors in kitchen**

```json
{"area":"kitchen","shortcuts":{"stats_eligible":true}}
```

### **6. Regex applied to state**

```json
{"domain":"climate","regex":"heat|aux|on"}
```

### **7. Power spike detection**

```json
{"label":"laundry","shortcuts":{"power":true},"numeric_above":500}
```

---

# 🔒 Error Handling (ZQ-1 Guarantees)

ZQ-1 never throws template exceptions.
Failures always produce safe falls, not crashes.

* Invalid float → entity skipped
* Missing attributes → skipped
* Regex on non‐string → skipped
* Unknown area/device/label → result becomes intersection with empty
* Unknown filter fields → ignored
* Broken JSON → `{}` silently

HyperIndex sees either:

* A valid entity list
* Or an empty workable list

Never a broken structure.

---

# 🔗 How HyperIndex Uses ZQ-1

HyperIndex runs your `filter_json` exactly like this:

```
entities_base → ZQ-1 filter_json → filtered_entities_base
```

Your set logic (AND/OR/XOR/NOT) happens **before** filtering.
Expansion and hypergraph building happen **after** filtering.

So filter_json is essentially:

> **ZQ-1 final trim before expansion and graph generation.**

---

# 🎯 Summary — What You Should Pass to `filter_json`

**You MUST provide:**

* A JSON dictionary
* Only ZQ-1 fields
* Strings, numbers, booleans, arrays of strings
* No YAML, no lists-as-root, no non-JSON structures

**HyperIndex will:**

* Convert it to a dict
* Pass it to ZQ-1
* Continue with its reasoning pipeline

Everything stays clean, deterministic, and fully aligned.

