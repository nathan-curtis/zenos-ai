# **ZenOS-AI Indexer System Architecture and Specification**

**Path:** `docs/library/index_system.md`
**Implements:**

* `custom_templates/zenos_ai/library_index.jinja`
* `scripts/dojotools_zen_index.yaml`
* `automations/dojotools_zen_index_event_handler.yaml`

---

## **1. Purpose**

The **Zen Indexer System** is the unified querying and discovery layer of ZenOS-AI.
It provides **label-based logic**, **semantic relationships**, and **recursive lookups** across entities, drawers, and cabinets within the household knowledge graph.

Its guiding question:

> “Given one or more labels, what objects match—and how are they related?”

The Indexer supports:

* Boolean set operations (`AND`, `OR`, `NOT`, `XOR`)
* Optional metadata expansion
* Event-driven recursion
* Canonical JSON output for the FileCabinet and Monastery

---

## **2. Component Overview**

| Component                   | Type           | Version | Role                                                   |
| --------------------------- | -------------- | ------- | ------------------------------------------------------ |
| **Library Index Core**      | Jinja Template | 1.1     | Inline `~INDEX~` parser & label-logic engine           |
| **DojoTools Zen Index**     | Script         | 2.7.1   | Structured API for automations and agents              |
| **Zen Index Event Handler** | Automation     | 2.0.0   | Listens for `zen_indexer_request` → recursive dispatch |

These three pieces form the **Index Stack**, bridging synchronous macros and asynchronous recursion.

> ⚙️ **Implementation Note:**
> In this release, the **Library Index Core** exists as a **stand-alone Jinja template**, separate from the `zen_os_1` core.
> Future ZenOS releases will merge it into the **Command Interpreter layer**—making label parsing and index logic native to the system’s instruction processor.

---

## **3. Logic Flow**

### **A — Inline (Jinja) Invocation**

1. **Macro Call**

   ```jinja
   ~INDEX~lighting AND active true
   ```
2. **Macro Engine**

   * Tokenizes query
   * Detects operator + expansion flag
   * Resolves entities via `label_entities()`
   * Expands attributes if requested
3. **Output**

   ```yaml
   entities:
     - light.kitchen_ceiling
     - light.living_room_main
   labels:
     - lighting
     - active
   ```

Lightweight and synchronous — used directly in templates or summarizers.

---

### **B — Script Invocation + Recursion Loop**

**File:** `scripts/dojotools_zen_index.yaml`
**Alias:** `DojoTools Zen Index`
**Version:** 2.7.1

The script serves as both a **direct access API** and an **entry point to event-driven recursion**, forming the core of the new Indexer design.

---

#### **1️⃣ Direct (Synchronous) Mode**

When operands (`label_1`, `label_2`, or `entities_*`) are provided:

1. Resolve operands via `label_entities()`
2. Apply logical operator (`AND`, `OR`, `NOT`, `XOR`)
3. Expand metadata if requested
4. Return structured JSON immediately

This path mirrors the legacy `~INDEX~` macro behavior—ideal for local lookups.

---

#### **2️⃣ Event Mode (Recursive Path)**

When `index_command` is supplied, the script enters **event mode**, converting the call into an asynchronous recursive loop.

**Event Flow**

1. **Emit Event**

   ```yaml
   event: zen_indexer_request
   event_data:
     index_command: "lighting NOT offline"
     correlation_id: "b73c3-9279-9211"
     depth: 1
   ```

2. **Event Handler Receives**

   * Parses `index_command` (JSON or string form)
   * Resolves operands
   * Calls `script.dojotools_zen_index` again **without** `index_command`

3. **Recursion & Unwind**

   * Each nested pass increments `depth`
   * Further nested commands trigger new events
   * When no `index_command` fields remain, recursion ends
   * Each layer emits a `zen_index_response` event with matching `correlation_id`
   * Top-level caller waits (default 2 s) and merges results

4. **Legacy Translation**

   * Replaces Jinja recursion used by `~INDEX~labelA OR labelB`
   * Each event equates to one logical evaluation cycle of the old macro system
   * Results cascade upward, forming a complete unwound recursion tree

**Benefits**

* Unifies macro and script logic
* Stack-safe event recursion
* Fully backward-compatible
* Graceful timeout fallback
* Ready for cross-cabinet use

---

#### **3️⃣ Planned Refinement — Drawer Recursion Path**

Upcoming expansion introduces drawer-level recursion:

1. Detect unresolved labels
2. Emit secondary `zen_indexer_request (domain=cabinet)`
3. Query FileCabinet Meta Index
4. Merge drawer results with entity data

```
~INDEX~hot_tub.kata
   ↓
entity not found → event(domain=cabinet)
   ↓
drawer lookup → return metadata
   ↓
merge with primary results
```

Completes the loop between live state and stored knowledge.

---

### **C — Event Handler Automation**

**File:** `automations/dojotools_zen_index_event_handler.yaml`
**Alias:** `DojoTools Zen Index Event Handler`
**Version:** 2.0.0

Listens for `zen_indexer_request`, parses commands, calls the script, and returns `zen_index_response`.

| Variable          | Purpose                    |
| ----------------- | -------------------------- |
| `index_command`   | Raw or JSON command string |
| `correlation_id`  | Request/response pairing   |
| `depth`           | Recursion depth counter    |
| `operator`        | Logical operator           |
| `expand_entities` | Include metadata           |

**Execution**

```
zen_indexer_request
   ↓
parse → dispatch → script.dojotools_zen_index
   ↓
zen_index_response (same correlation_id)
```

This forms the backbone of multi-pass, event-safe recursion across entities, drawers, and soon, networked cabinets.

---

## **4. Technical Specifications**

### **A — `custom_templates/zenos_ai/library_index.jinja`**

| Element                   | Function                        |
| ------------------------- | ------------------------------- |
| `normalize_label_index()` | Flatten nested label maps       |
| `parse_index_command()`   | Tokenize query strings          |
| `query()`                 | Apply Boolean logic             |
| `label_entities_*()`      | Helpers for combined conditions |
| `label_entities_cart()`   | Generate relation maps          |

**Outputs:** entity lists or expanded JSON
**Dependencies:** Home Assistant Jinja helpers

> 🔸 *Note:*
> The Library Index Core currently runs as an independent module during system 1.x.
> In ZenOS 2.0, it will be embedded within the system’s **Command Interpreter**, making label logic a first-class instruction type at the OS level.

---

### **B — `scripts/dojotools_zen_index.yaml`**

| Field                      | Description             |    |     |      |
| -------------------------- | ----------------------- | -- | --- | ---- |
| `index_command`            | Triggers event mode     |    |     |      |
| `label_1`, `label_2`       | Label operands          |    |     |      |
| `entities_1`, `entities_2` | Explicit entity lists   |    |     |      |
| `operator`                 | `AND                    | OR | NOT | XOR` |
| `expand_entities`          | Include full state data |    |     |      |
| `timeout`                  | Wait time (default 2 s) |    |     |      |

**Output Example**

```json
{
  "result": {
    "simple": ["light.kitchen"],
    "expanded": [{ "entity_id": "light.kitchen", "state": "on" }],
    "adjacent_labels": ["lighting","kitchen"],
    "index": [["light.kitchen",["lighting","kitchen"]]]
  },
  "operator": "AND",
  "inputs": { "label_1":"lighting","label_2":"active" }
}
```

---

### **C — `automation.dojotools_zen_index_event_handler.yaml`**

Handles recursive event processing and returns matched results.

**Sequence**

1. Triggered by `zen_indexer_request`
2. Parse command → call script
3. Emit `zen_index_response`
4. Increment recursion depth

---

## **5. Recursive Lookup Mechanics**

### **Call Chain**

```
~INDEX~ (macro)
   ↓
script.dojotools_zen_index
   ↓ emits zen_indexer_request
automation.dojotools_zen_index_event_handler
   ↓ calls script again
   ↓ emits zen_index_response
script receives → returns merged JSON
```

### **Controls**

* `correlation_id` — matches calls to responses
* `depth` — recursion limiter
* `timeout` — graceful fallback

### **Future Depth Targets**

1. Entity → Drawer
2. Drawer → Cabinet
3. Cabinet → Household Context

---

## **6. Planned Enhancements**

| Feature                      | Description                               |
| ---------------------------- | ----------------------------------------- |
| **FileCabinet Label Scan**   | Search drawers for label matches          |
| **Unified Core Template**    | Merge macro + script into a single import |
| **Distributed Index Events** | Cross-node recursion                      |
| **Extended Operators**       | `NEAR`, `CONTAINS`, regex                 |
| **Depth Policy**             | Per-cabinet recursion limits              |

---

## **7. Related Documents**

| Topic                 | Path                                |
| --------------------- | ----------------------------------- |
| Cabinet Specification | `docs/cabinets/cabinet_spec.md`     |
| Hypergraph Model      | `docs/cabinets/hypergraph_model.md` |
| Kung Fu Components    | `docs/kung_fu/readme.md`            |
| Zen Summarizer        | `docs/zen_summarizer/readme.md`     |
| DojoTools Reference   | `docs/dojo_tools/readme.md`         |

---

## **8. External Reference**

🦣 **Stuffing Elephants in Drawers**
Metaphor for symbolic vs. literal storage in AI systems.
[Community Thread](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/120?u=nathancu)

---

## **9. Implementation Notes & Future Path**

Current recursion performs a full single-response loop.
Completing multi-layer recursion will involve:

1. Drawer-level mappings in FileCabinet
2. Secondary event dispatch for unresolved labels
3. Merged entity + drawer data
4. Depth-based termination

At that point, the Indexer will serve as the unified resolver across **entities, drawers, cabinets, and the Household Context Graph**.

---

## **10. Version & Attribution**

| Field            | Value                                                                                                      |
| ---------------- | ---------------------------------------------------------------------------------------------------------- |
| **Author**       | Veronica (assistant to Nathan Curtis)                                                                      |
| **Project**      | ZenOS-AI                                                                                                   |
| **Version**      | 1.4 (Recursive Core Spec)                                                                                  |
| **License**      | MIT                                                                                                        |
| **Last Updated** | 2025-11-11                                                                                                 |
| **Notes**        | Library Index Core is modular in 1.x; merges into ZenOS Command Interpreter in 2.0 for native label logic. |

---
