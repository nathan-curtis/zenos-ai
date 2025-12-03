ZenOS-AI Template Engine: Zen Query Engine (ZQ-1) Technical Specification  
Version: 1.3.4 RC1 (Paging + Numeric Sort)  
Status: Stable and required for all Zen DojoTools Query flows  

---

## Purpose of the Zen Query Engine

The Zen Query Engine (ZQ-1) is the canonical entity filter for ZenOS-AI inside Home Assistant. It is a pure Jinja template that accepts a JSON filter description and produces a final, ordered list of `entity_id` values.

ZQ-1 is the "WHERE clause" of ZenOS. Scripts like `script.zen_dojotools_query` rely on it to select the exact entities that Friday and the Monastery should reason about at any given time. It never writes state, never uses unsafe Python, and is designed to be safe inside the Home Assistant template sandbox. :contentReference[oaicite:0]{index=0}  

---

## What the Template Engine Actually Does

ZQ-1 is implemented as a pair of Jinja macros:

1. `zen_filter(filter_json, entity_list)`  
   The core filter engine. Takes an input set plus a JSON description of filters, shortcuts, sorting, and paging. Returns a JSON array of `entity_id` values.

2. `zen_query_help()`  
   Returns a JSON document that describes the engine, its fields, and the execution model. This is consumed by `Zen DojoTools Query` when its `mode: help` path is called.

When invoked, `zen_filter` performs a deterministic pipeline:

1. Parse the filter JSON safely.
2. Normalize the entity list.
3. Optionally auto-build the working set based on `domain`.
4. Apply all explicit filters (domain, label, area, device, state, numeric thresholds, regex).
5. Apply shortcut packs to narrow or reshape the set.
6. Apply sorting.
7. Apply paging with `offset` and `limit`.
8. Emit the final list as JSON.

Every step only reduces or reorders the working set. No step adds entities that were not already in the pipeline, except for the initial domain expansion when the input list is empty.

---

## Macro Overview

### `zen_filter(filter_json, entity_list)`

**Signature**

```jinja
{%- macro zen_filter(filter_json, entity_list) -%}
  ...
  {{ pipe.ids | tojson }}
{%- endmacro -%}
````

**Inputs**

* `filter_json`

  * A JSON object describing filters, shortcuts, sort, and paging.
  * May be passed as a string or as a mapping. The macro will parse strings with `from_json` and pass mappings through unchanged.

* `entity_list`

  * Optional. Seed list of entity ids.
  * May be:

    * `null` or omitted
    * a single `entity_id` string
    * a list of `entity_id` strings
    * any iterable that ultimately converts to a list

**Return value**

* Always returns a JSON array of `entity_id` strings, for example:

  ```json
  ["light.living_room_main", "light.kitchen_table"]
  ```

This output shape is designed to be safe to feed into other scripts, into the Monastery, or back into Jinja filters that expect a JSON list.

---

## Execution Model

ZQ-1 is strictly sequential and deterministic.

1. **Filter JSON parsing**

   ```jinja
   {% set f = filter_json | from_json if filter_json is string else filter_json %}
   ```

   The macro supports both stringified JSON and native mappings. If parsing fails in Home Assistant, the template itself will fail earlier, so upstream tools are expected to send valid JSON.

2. **Field extraction**

   The macro pulls known fields from `f`:

   * Selection fields: `domain`, `label`, `area`, `device_id`, `device_class`
   * State filters: `state_equals`, `include_states`, `exclude_states`
   * Numeric filters: `numeric_above`, `numeric_below`
   * Regex filter: `regex`
   * Sort configuration: `sort` (mapping)
   * Shortcuts pack: `shortcuts` (mapping, default `{}`)
   * Paging: `limit`, `offset`

3. **Entity list normalization**

   `entity_list` is normalized into `pipe.ids`:

   * `null`             → `[]`
   * string             → `[string]`
   * iterable           → `list(iterable)`
   * any other scalar   → `[]`

   The working set is always a list of entity ids.

4. **Domain auto build**

   If no entities are present and `domain` is supplied:

   ```jinja
   states[domain] | map(attribute='entity_id') | list
   ```

   This yields all entities in the given domain as the starting pipeline. If `entity_list` already provided entities, this expansion is skipped, and later domain filtering only performs intersection.

5. **Sequential filters**

   The macro then applies filters in a fixed order:

   1. Domain filter
   2. Label filter
   3. Area filter
   4. Device id filter
   5. Device class filter
   6. `state_equals`
   7. `include_states`
   8. `exclude_states`
   9. Numeric filters
   10. Regex filter

   Each step builds a new list and replaces `pipe.ids`. If any step produces an empty list, later steps still run but have no effect.

6. **Shortcut stages**

   After the explicit filters, ZQ-1 applies shortcut packs described in `shortcuts`. These are convenience operations that narrow, dedupe, or slice the current result set.

7. **Sorting**

   If a `sort` spec is present, the macro builds intermediate `[ {eid, st}, {eid, val} ]` objects as needed and sorts on the selected key.

8. **Paging**

   Finally, `offset` and `limit` are applied to the sorted list. Paging never changes order. It only returns a window of the current working set.

9. **JSON output**

   The final `pipe.ids` list is emitted via `tojson`, which ensures a predictable JSON array shape, even when the internal list is empty.

---

## Filter JSON Field Reference

This section describes every top level field that `zen_filter` reads from the filter JSON object.

### `domain` (string)

* Use to restrict the working set to a single Home Assistant domain.
* Behavior:

  * If `entity_list` is empty and `domain` is provided, ZQ-1 auto-builds from `states[domain]`.
  * If `entity_list` is non empty, ZQ-1 keeps only entities whose ids start with `"domain."`.

Common usage:

```json
{"domain": "light"}
```

---

### `label` (string)

* Logical AND with the label system.
* Behavior:

  * The string is slugified.
  * The engine calls `label_entities(slug)` and intersects the pipeline with that list.

No expansion happens here. If the label resolves to no entities, the result becomes an empty list.

---

### `area` (string)

* Filters by Home Assistant area.

Execution path:

1. Slugify the input.
2. Resolve `area_id(normalized)`.
3. Call `area_entities(aid)` and intersect the result with the current pipeline.

---

### `device_id` (string)

* Filters by a specific device, using the Home Assistant device registry.

Execution path:

1. Slugify the input.
2. Resolve `device_id(normalized)`.
3. Call `device_entities(did)` and intersect the result with the pipeline.

---

### `device_class` (string)

* Filters entities by the `device_class` attribute.

Only entities with `state_attr(ent, "device_class") == device_class` are retained.

---

### `state_equals` (string)

* String equality check on the entity state.

Example:

```json
{"domain": "light", "state_equals": "on"}
```

Retains only entities whose `states(ent)` string exactly equals `"on"`.

---

### `include_states` (list of strings)

* Keep only entities whose state is in the given list.

Example:

```json
{"include_states": ["playing", "paused"]}
```

---

### `exclude_states` (list of strings)

* Drop any entity whose state is in the list.

Example:

```json
{"exclude_states": ["unavailable", "unknown"]}
```

---

### Numeric filters

* `numeric_above` (number)
* `numeric_below` (number)

The macro attempts to parse each entity state as `float(none)` and then applies numeric bounds.

Rules:

* If parsing fails, the entity is skipped for numeric evaluation.
* If `numeric_above` is set, values must be strictly greater.
* If `numeric_below` is set, values must be strictly less.
* Both may be combined to create a numeric window.

Example:

```json
{"shortcuts": {"temp": true}, "numeric_above": 78}
```

---

### `regex` (string)

* Filters entities by applying a Home Assistant `is regex` test to the state string.

Rules:

* Non string states are skipped.
* The regular expression is treated as given to the `regex` test.
* If the test matches, the entity is kept.

Example:

```json
{"domain": "climate", "regex": "heat|on|aux"}
```

---

### `shortcuts` (mapping)

The `shortcuts` dictionary provides higher level operations that narrow or reframe the current working set.

Each key is optional and may be combined with others. When multiple shortcuts are set, they are applied in the order they are coded in the macro.

Supported shortcuts in this version:

#### Core numeric and stats

* `numeric: true`

  * Keep only entities whose state can be parsed as a float.

* `stats: true`

  * Keep only entities that expose a `state_class` attribute.
  * Intended to select entities that are eligible for Home Assistant long term statistics, but does not currently validate unit or recorder configuration.

> Note: The `zen_query_help()` macro mentions a more strict `stats_eligible` pattern that would also enforce valid units and non disabled entities. That stricter behavior is reserved for a future extension of ZQ-1 and is not fully implemented in this template version.

#### Semantic device class filters

* `temp: true`

  * Keep only entities with `device_class == "temperature"`.

* `humidity: true`

  * Keep only entities with `device_class == "humidity"`.

* `power: true`

  * Keep only entities with `device_class == "power"`.

* `energy: true`

  * Keep only entities with `device_class == "energy"`.

* `water: true`

  * Keep only entities with `device_class == "moisture"`.

#### Domain scoped shortcut

* `binary: true`

  * Keep only `binary_sensor.*` ids.

#### Set slicing and ranking

These shortcuts operate on the current ordering of `pipe.ids`.

* `first_n: N`

  * Take the first `N` items.

* `last_n: N`

  * Take the last `N` items.

* `top_n: N`

  * After sorting, take the first `N` items. This is functionally identical to `first_n` but semantically paired with sorting.

* `bottom_n: N`

  * After sorting, take the last `N` items.

* `top_by_numeric: N`

  * Build objects `{eid, val}`, parse numeric state, drop entities with non numeric state, sort in descending order, and take the first `N`.

* `bottom_by_numeric: N`

  * Same as above, but sort in ascending order.

* `top_by_state: N`

  * Sort entities by state string descending, take first `N`.

* `bottom_by_state: N`

  * Sort entities by state string ascending, take first `N`.

* `random_n: N`

  * Shuffle the working set with Home Assistant’s `shuffle` filter and take the first `N`.

#### De duplication and uniqueness

* `distinct: true`

  * Remove duplicate entity ids while preserving first seen order.

* `unique_domains: true`

  * Keep only the first entity encountered for each domain prefix (the part before the dot).

* `unique_devices: true`

  * Look up `device_id(ent)` for each entity and keep only the first entity for each device.

#### Device class limiting

* `limit_by_device_class: { "class": "<device_class>", "n": N }`

  Example:

  ```json
  {
    "shortcuts": {
      "limit_by_device_class": {
        "class": "temperature",
        "n": 5
      }
    }
  }
  ```

  ZQ-1 will filter by the given device class and then keep only the first `N` entities.

---

### `sort` (mapping)

Sorting is controlled by an optional `sort` object inside the filter JSON.

Fields:

* `by`

  * One of:

    * `"entity_id"`
    * `"state"`
    * `"numeric_state"` (safely float parses state and drops non numeric)

* `order`

  * `"asc"` or `"desc"` (defaults to ascending)

Behavior:

* For `"entity_id"`, sorting is applied directly to the list of ids.
* For `"state"` and `"numeric_state"`, the macro constructs helper objects, sorts them by the requested key, then maps back to entity ids.

---

### `limit` and `offset`

Paging is always the last step after sorting and shortcuts.

* `offset` (number, default `0`)

  * Number of items to skip from the start of the working set.

* `limit` (number, optional)

  * Maximum number of items to return after applying `offset`.

Rules:

* If `limit` is a number, ZQ-1 computes a `[start:end]` slice.
* If `limit` is not provided but `offset` is, ZQ-1 returns `pipe.ids[start:]`.

Examples:

```json
{"limit": 10}
{"offset": 10, "limit": 10}
```

---

## `zen_query_help()` Macro

The second macro, `zen_query_help()`, returns a JSON object that documents ZQ-1 for downstream tools and UI layers.

It covers:

* Tool name and version.
* High level overview and philosophy.
* Input behavior and normalization rules.
* Field by field descriptions for:

  * `domain`, `label`, `area`, `device_id`, `device_class`
  * `state_equals`, `include_states`, `exclude_states`
  * `numeric_above`, `numeric_below`
  * `regex`
  * `shortcuts` and each known shortcut key
  * `sort` options
  * `limit` and `offset`
* Filter execution model.
* Error handling rules.
* Return value contract.
* Example filter JSON snippets for common use cases.

This macro is intended to be called from `script.zen_dojotools_query` in `mode: help`, then surfaced as a JSON help document in logs, developer tools, or future UI panels. ([GitHub][1])

---

## Error Handling and Safety Model

ZQ-1 is designed to be safe in the Home Assistant Jinja sandbox:

* Invalid numeric states are skipped rather than causing exceptions.
* Missing attributes are treated as non matches.
* Regex tests are only applied to string states.
* Empty intersections are allowed at any stage.
* Domain, area, or device lookups that fail simply result in an empty working set.
* The macro never alters Home Assistant state and never calls unsafe Python code.

If the filter JSON is malformed such that Home Assistant cannot parse it with `from_json`, the template will fail before ZQ-1 runs. Upstream tools are responsible for generating syntactically valid JSON.

---

## Example Usage Patterns

### 1. Find all lights that are on

```jinja
{{ zen_filter('{"domain":"light","state_equals":"on"}', null) }}
```

Result: JSON array of all `light.*` entities currently on.

---

### 2. All motion binary sensors in the living room

```jinja
{{ zen_filter('{"area":"living room","shortcuts":{"binary": true}}', null) }}
```

Result: All `binary_sensor.*` entities that belong to the living room area.

---

### 3. Top 5 hottest temperature sensors

```jinja
{{ zen_filter('{
  "shortcuts": {"temp": true},
  "sort": {"by": "numeric_state", "order": "desc"},
  "limit": 5
}', null) }}
```

Result: The five highest temperature sensors available.

---

### 4. Bottom 3 humidity sensors

```jinja
{{ zen_filter('{
  "shortcuts": {"humidity": true},
  "sort": {"by": "numeric_state"},
  "limit": 3
}', null) }}
```

Result: The three entities with the lowest numeric humidity values.

---

### 5. Regex match for heating modes on climate entities

```jinja
{{ zen_filter('{
  "domain": "climate",
  "regex": "heat|on|aux"
}', null) }}
```

Result: Climate entities whose state string matches any of the listed heating related words.

---

## Developer Requirements and Integration Notes

* ZQ-1 must be imported and used by any DojoTool that needs flexible entity selection based on label, area, or device metadata.
* Scripts should treat the JSON output of `zen_filter` as canonical and avoid re interpreting entity ids through ad hoc filters.
* The help macro `zen_query_help()` should be wired into your `help` paths so that Friday, Kronk, and future UIs can inspect capabilities at runtime.
* When extending ZQ-1 (for example to add a new shortcut), you must:

  1. Update the `zen_filter` implementation.
  2. Update the `zen_query_help()` JSON description.
  3. Update any script level documentation in `docs/scripts/zen_dojotools_query.md` to match.

---

## Philosophy

ZQ-1 exists to keep entity selection boring, predictable, and explainable. There is no hidden magic and no silent inference. Everything it does is visible in the filter JSON and in the help macro.

If Cabinets are the filesystem and HyperIndex is the attention layer, then ZQ-1 is the filter bar for the whole mind. It lets Friday ask "show me exactly these entities" and get an answer that is repeatable and audit friendly.

rcontent.com/nathan-curtis/zenos-ai/refs/heads/main/docs/readme.md "raw.githubusercontent.com"
