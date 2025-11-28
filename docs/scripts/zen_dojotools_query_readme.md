# **Zen DojoTools Query (ZQ-1)**

### **Deterministic Entity Selector for Home Assistant**

**Version:** 3.9.0 RC1
**Namespace:** `script.zen_dojotools_query`
**Icon:** `mdi:filter-cog`

---

# 🧠 Overview

**ZQ-1** is the foundational **deterministic entity filtering engine** used throughout Project Friday.
It acts like a **database WHERE clause** for Home Assistant devices.

It consumes:

* a **base set** of entities (optional)
* a **JSON filter dictionary**
* and produces a **clean, stable JSON array of entity_ids**

Every filtering step is:

* **strict**
* **logical AND**
* **non-expanding** (except domain auto-build)
* **predictable**
* **safe** (no template crashes)
* **idempotent** (same input → same output)

This makes ZQ-1 the “primitive” that higher-level DojoTools rely on, including:

* **HyperIndex**
* **Summarizers**
* **Fusion models**
* **Drift detection**
* **RoomState logic**
* **Kata generation**
* **Automation reasoning**

---

# 🎯 Purpose

ZQ-1 gives Friday and the Monastery a **guaranteed-correct, deterministic, reproducible method** for selecting entities in a complex Home Assistant environment.

It ensures:

* No guessing
* No implicit expansions
* No hallucinatory label resolution
* No surprises in output
* Full safety during template runtime

This becomes the “trust anchor” for all higher reasoning.

---

# 🔧 Core Behavior

ZQ-1’s behavior is governed by three simple rules:

### **1. If you provide `target_entities`:**

ZQ-1 will filter **only** that list.

### **2. If `target_entities` is empty AND a `domain` is provided:**

ZQ-1 automatically constructs the starter list from:

```
states[domain] → map(attribute='entity_id')
```

### **3. If both are empty:**

Result is an **empty set**.

**No other logic expands the list. Ever.**

---

# 🧩 Inputs / Fields

```yaml
mode:              # "help" or "query"
log_result:        # boolean — write filter + baselist + result to logbook
emit_event:        # boolean — emit structured zen_event
target_entities:   # optional list of entity_ids
filter_json:       # required JSON dict describing ZQ-1 filters
```

### Mode Behavior

| Mode    | Description                                                       |
| ------- | ----------------------------------------------------------------- |
| `help`  | Returns a full JSON specification (schema, examples, philosophy). |
| `query` | Executes filtering pipeline and returns JSON array of entities.   |

---

# 🧬 Filter JSON Specification

*(Authoritative — ZQ-1 Schema)*

Your `filter_json` must be a **valid JSON object** containing any of these optional fields:

```yaml
domain:             # "light" → restrict or auto-build
label:              # "kitchen" → intersection with label_entities()
area:               # "office" → resolves via area_id() / area_entities()
device_id:          # "abcd1234" → device_entities()
device_class:       # "temperature"

state_equals:       # string — exact match
include_states:     # list[string]
exclude_states:     # list[string]

numeric_above:      # float threshold
numeric_below:      # float threshold

regex:              # string, matched against STATE STRING

shortcuts:
  numeric:          # require numeric state
  stats:            # require state_class
  temp:             # device_class == temperature
  humidity:         # device_class == humidity
  power:            # device_class == power
  energy:           # device_class == energy
  water:            # device_class == moisture
  binary:           # domain must be binary_sensor
  stats_eligible:   # numeric + state_class + uom + not disabled

sort:
  by:               # "entity_id" | "state" | "name"
  order:            # "asc" | "desc"
```

### All filters apply with **logical AND**.

There is no OR logic — run multiple queries if needed.

---

# ⚙ Execution Pipeline

ZQ-1 executes filters in deterministic order:

```
1. Normalize target_entities
2. Domain auto-build (optional)
3. Filter: domain
4. Filter: label
5. Filter: area
6. Filter: device_id
7. Filter: device_class
8. Filter: state_equals
9. Filter: include_states
10. Filter: exclude_states
11. Filter: numeric_above / numeric_below
12. Filter: regex
13. Filter: shortcuts (all must match)
14. Sort (optional)
```

No step expands results except *domain auto-build*.

---

# 🛡 Error Handling (Guaranteed)

ZQ-1 **never throws** a template error.

All failures become safe behavior:

* invalid float → entity dropped
* missing attribute → dropped
* regex on non-string → dropped
* unknown area/label/device → intersection with empty
* malformed JSON → `{}`
* unknown keys → ignored

This ensures Friday can call ZQ-1 even in uncertain or partial state conditions.

---

# 📤 Return Value

Always returns a JSON object:

```json
{
  "status": "ok",
  "domain": "light",
  "baselist": ["light.kitchen_main", ...],
  "result": "[\"light.kitchen_main\", \"light.table_lamp\"]"
}
```

Internally, `result` is a JSON array encoded as a string.

---

# 🧪 Example Filters

### **Find all lights that are on**

```json
{"domain":"light","state_equals":"on"}
```

---

### **Binary motion sensors in the living room**

```json
{"area":"living room","shortcuts":{"binary":true}}
```

---

### **Temperature sensors above 78°F**

```json
{"shortcuts":{"temp":true}, "numeric_above":78}
```

---

### **Laundry power draw spike**

```json
{"label":"laundry","shortcuts":{"power":true},"numeric_above":500}
```

---

### **Stats-eligible devices in the kitchen**

```json
{"area":"kitchen","shortcuts":{"stats_eligible":true}}
```

---

# 🧭 Usage Instructions

1. Construct a JSON filter object.
2. Run:

```yaml
action: script.zen_dojotools_query
data:
  mode: query
  filter_json: '{"domain":"light","state_equals":"on"}'
```

3. Provide `target_entities` if you want to limit the base set.
4. If you want domain auto-build, leave `target_entities` empty.
5. Use shortcuts whenever possible (optimized path).
6. Treat output as authoritative.
7. ZQ-1 is stateless and idempotent — safe for loops, cycles, and Monastery calls.

---

# 🧱 Position in the DojoTools Architecture

ZQ-1 forms the **ground truth entity selection layer** beneath:

* HyperIndex
* Summarizers
* Fusion
* Drift Analysis
* Reasoning Tools
* Kata Generation
* RoomState Subconscious

Everything that needs a clean, correct, non-hallucinatory device list starts with ZQ-1.

---

# 🔚 Summary

ZQ-1 is:

* deterministic
* safe
* explicit
* non-expanding (except domain auto-build)
* AND-composed
* optimized
* the “truth engine” for entity selection

It is one of the most critical primitives in Project Friday’s internal reasoning stack.
