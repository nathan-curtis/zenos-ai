# **04_Cognitive_Data_Flow.md**

*Version 1.0 • ZenOS-AI Cognitive Architecture Whitepaper*
*Draft for academic and engineering review*

---

# 4. Cognitive Data Flow (Version 1 Architecture)

The cognitive data flow of Friday’s House is a deterministic, event-driven pipeline implemented entirely on top of **Home Assistant’s core primitives**:

* the **event bus**
* **automations and scripts** (Zen DojoTools)
* **stateful sensors** used as “cabinets”
* local **AI tasks** bound to `ai_task` entities

Version 1 does not use a separate message broker or external RPC substrate. All cognitive work is expressed as a sequence of state transitions and events inside Home Assistant, with local reasoning workers attached at defined points.

The result is a system where every reasoning step is:

* causally traceable in the HA logs
* scoped to explicit triggers and automations
* bound to a concrete snapshot of home state at the moment of invocation

This section describes that flow in terms of **query packets**, **event routing**, **sanitized state extraction**, **reasoning calls**, and **Kata synthesis**.

---

## 4.1 Query Packets: The Cognitive Unit of Work

Every cognitive operation begins as a **Query Packet**: a structured JSON object describing what needs to be reasoned about and under which constraints.

At minimum, a Query Packet contains:

* `intent`: the requested cognitive operation (indexing, inspection, summary, identity resolution, etc.)
* `inputs`: entities, labels, or drawers to consider
* `origin`: which subsystem or persona initiated the request
* `flags`: expansion, safety, and mode controls
* `correlation_id`: unique identifier used to match responses
* `timestamp`: issuance time in the Home Assistant clock domain

Example conceptual payload for an indexing operation:

```json
{
  "intent": "index_query",
  "origin": "Friday",
  "inputs": {
    "labels": ["Kitchen", "Climate"],
    "operator": "AND",
    "expand_entities": true
  },
  "correlation_id": "2025-11-17T14:20:54.123Z-487",
  "timestamp": "2025-11-17T14:20:54.123Z"
}
```

In practice, these packets are not passed directly across a message broker. Instead, they are embedded into:

* event payloads, when using custom events
* script fields, when using HA’s `script.turn_on` with `data`
* cabinet drawers, when persisting configuration or prior results

The important property is that the Query Packet is **self-describing and self-contained**: everything needed for a reasoning worker to act is embedded inside the packet, without hidden global state.

---

## 4.2 Event Bus as Cognitive Spine

The **Home Assistant event bus** provides the backbone of Version 1 cognition. All higher-level reasoning is implemented as:

1. emission of an event
2. script/automation triggered by that event
3. bounded work that may call a local AI task
4. optional emission of a response event

Several characteristics of the HA event bus are critical for cognitive stability.

### 4.2.1 Deterministic Execution Model

Home Assistant processes events through an internal executor, with well-defined semantics:

* each automation/script run is a concrete event handling instance
* concurrency is constrained through `mode` (`single`, `queued`, `restart`, `parallel`)
* any given script invocation either runs to completion, is queued, or is cancelled in a defined way

Zen DojoTools scripts (Index, Inspect, Ninja Summarizer, SuperSummary, Scheduler, Identity) explicitly declare their `mode` and `max` to bound parallelism and guarantee predictable behavior even under load.

### 4.2.2 Global Observability

All events, including those carrying cognitive payloads, are visible in:

* HA’s Developer Tools → Events
* the logbook
* the raw log files

There is no hidden channel where reasoning occurs; everything flows through the same observable substrate as physical device events. This makes it possible to reconstruct a full chain:

> trigger → automation/script → AI task → cabinet write → downstream reaction

without separate tracing infrastructure.

### 4.2.3 Short-Lived Messages and No Shared Cross-Thread Memory

Event payloads are short-lived and scoped to a single handling run. State that must persist beyond that is written explicitly into:

* entity state
* cabinet drawers under `sensor.zen_*_cabinet`
* long-term statistics (for eligible entities)

Because Home Assistant does not expose arbitrary shared mutable memory across worker threads, the cognitive pipeline cannot accidentally create race conditions by sharing arbitrary objects. All cross-call state must be serialized into the HA model, which is then read deterministically.

---

## 4.3 Sanitized State Extraction

A central requirement for stable cognition is **defensive parsing**. Raw HA values can be:

* `unknown`
* `unavailable`
* empty strings
* partially stringified dicts
* valid JSON strings
* native mappings

Naive patterns like:

```jinja2
{{ payload | from_json | default({}) }}
```

are unsafe, because `from_json` in Jinja will raise if the string is not valid JSON. Zen DojoTools therefore uses a **guard-then-parse** pattern:

1. Normalize sentinel and null values.
2. Branch on type.
3. Parse only when safe.

Conceptually, the pattern looks like:

```jinja2
{# Step 1: Normalize raw #}
{% set raw = payload %}
{% if raw in [none, '', 'unknown', 'unavailable'] %}
  {% set obj = {} %}

{% elif raw is mapping %}
  {# Already a dict-like object #}
  {% set obj = raw %}

{% elif raw is string %}
  {# Candidate JSON; safe to attempt parsing #}
  {% set obj = raw | from_json(default={}) %}

{% else %}
  {# Fallback, defensive #}
  {% set obj = {} %}
{% endif %}
```

This pattern appears in multiple subsystems, adapted to their data sources:

* for cabinet drawers stored as JSON strings
* for manifest payloads read from a sensor
* for intermediate results that may be serialized

The result is that **no single malformed string can crash a cognitive script**. Every pull from `states()` or `state_attr()` is guarded before being passed to any filter that can raise.

Inspect takes an even more defensive posture for attributes:

* only scalar, mapping, or sequence types are admitted
* everything else is stringified
* attributes stored as stringified dicts are re-hydrated field by field

This systematic sanitation makes the higher-level cognitive behavior robust even under unusual device integrations or vendor quirks.

---

## 4.4 Indexing as Cognitive Routing

The **Zen Index** (DojoTools Index script) is the primary routing mechanism for cognition. It provides:

* set-logic over labels and entities
* optional expansion via Inspect
* adjacency discovery for labels and drawers
* structured reporting of index topology

From the perspective of data flow, Index performs the following steps:

1. **Resolve operands.**

   * gather `entities_1`, `entities_2` from fields
   * resolve `label_1`, `label_2` via `label_entities()` if entity lists are empty
   * if an `index_command` string is provided, dispatch to an external indexer and wait for a `zen_index_response` event, using a correlation ID

2. **Apply set operators.**

   * perform `AND`, `OR`, `NOT`, `XOR` using Jinja filters such as `intersect`, `difference`, `symmetric_difference`
   * if both operands are empty, treat the query as a universe query (`*`) and return all labels as a pure index view

3. **Optionally expand.**

   * if `expand_entities` is true, call `script.dojotools_zen_inspect` on the resulting entity set
   * otherwise, treat the base entity list as final

4. **Compute adjacency and drawer mappings.**

   * iterate each entity and collect its labels into a unique, sorted list of `adjacent_labels`
   * aggregate drawer information if cabinet drawers were matched by a higher-level query, computing `drawer_adjacent_labels`

5. **Construct a normalized result schema.**

   * `result.simple`: the raw entity IDs
   * `result.expanded`: optional inspection results
   * `result.adjacent_labels`: labels that co-occur with the target set
   * `result.index`: one entry per entity + labels
   * `result.drawers`: matched cabinet drawers (if any)
   * `result.drawer_adjacent_labels`: labels found in matching drawers

The response is returned as a single JSON object in `label_entity_logic_result`, with explicit `inputs`, `result`, and `error` fields. This becomes the **initial working memory** for downstream reasoning.

---

## 4.5 Inspect as Semantic Introspection

The **Zen Inspect** script is responsible for turning raw entity IDs into rich semantic objects that can be reasoned about by a model.

Given an entity list, Inspect performs:

1. **Canonicalization of entity list.**

   * ensures that `entity_id` is treated as a list internally, even if called with a single string

2. **State and metadata capture.**
   For each entity, Inspect collects:

   * `friendly_name`
   * `domain` (derived from the `entity_id` prefix)
   * `state`
   * `last_changed`
   * `last_updated`
   * `labels` via `labels(entity_id)`

3. **Attribute sanitization.**
   Inspect constructs a `safe_attributes` mapping as follows:

   * if `attributes` is mapping, iterate key–value pairs

     * keep scalars, mappings, and true sequences as-is
     * stringify unsupported or exotic types
   * if HA has stringified what should have been a dict, re-hydrate it by fetching each attribute with `state_attr()` and applying the type rules above

4. **Cabinet header extraction.**
   If the entity is a cabinet sensor, Inspect looks under:

   ```yaml
   attributes.variables.AI_Cabinet_VolumeInfo.value
   ```

   and pulls:

   * `id` (cabinet GUID)
   * `version`
   * `friendly_name`
   * `name`
   * `description`
   * `flags` (typed booleans for household, family, user, system, dojo, kata)
   * `validation` signature

   Only this header metadata is exposed; the drawer volume remains encapsulated behind cabinet tools.

5. **Statistics eligibility analysis.**
   Inspect evaluates:

   * `state_class`
   * `unit_of_measurement`

   and returns a `statistics` block indicating:

   * whether this entity can participate in HA long-term statistics
   * the reason for inclusion or exclusion

6. **Extended mode (device-centric context).**
   When `extended: true`, Inspect also resolves:

   * `device_id`
   * `area_id`
   * device `identifiers`

   via `device_attr()` calls. This allows spatial and hardware reasoning (“all entities in Kitchen,” “all entities on a specific device”).

The output is a list of self-contained entity records. Index can embed them as `expanded`, and higher-level summarizers can treat them as a semantic graph of the home’s current state.

---

## 4.6 Identity as Secure Persona Resolution

The **Zen Identity** module builds a coherent view of:

* people
* households
* families
* AI personas and constructs

It does this by cross-referencing:

* cabinet headers (*VolumeInfo*)
* embedded `zenai_essence` drawers
* `_zen_relationships` drawers
* ACLs and flags

The data-flow is multi-stage:

1. **Cabinet enumeration.**
   Identity enumerates household cabinets via:

   * a fresh `zen_library_manifest` drawer if recent
   * a guarded rescan of `states.sensor` using the cabinet validation signature

2. **Reverse indices.**
   It constructs:

   * `reverse_index`: `{ guid_lower → cabinet_entity_id }`
   * `reverse_person_index`: `{ person.entity_id → cabinet_entity_id }`

   These maps are built using cabinet flags, context, ACLs, and relationships to find the best matching cabinet for each person.

3. **GUID resolution.**
   For each person:

   * prefer `zenai_essence.identity.guid` if present
   * otherwise look up a related cabinet and extract its ID
   * as a last resort, infer from owner/assistant relationships or ACLs

4. **Cabinet merge.**
   For a resolved cabinet, Identity merges:

   * core header information
   * embedded essence (`zenai_essence`)
   * relationships (`_zen_relationships`)
   * context (`_context`)

   into a single, normalized structure.

5. **Squirrel mode redaction.**
   A labelled “Squirrel Switch” input boolean determines whether:

   * GUIDs
   * identity hashes
   * full essence details

   are redacted in the returned manifest. This permits internal reasoning while keeping sensitive identifiers hidden from external surfaces.

6. **Mode-dependent shaping.**
   Identity supports several output modes:

   * `summary`: minimal identity records for routing and ACL checks
   * `detail`: expanded cabinet, labels, ACLs, context, and relationships
   * `verbose`: exhaustive view for debugging and system introspection
   * `essence_prompt`: a compact persona object optimized for prompt loading into a model

The Identity result is written as a structured manifest, which downstream components can use for:

* secure routing
* authorization checks
* persona-aware reasoning
* future token-granting mechanisms (tool shunts and visas in later versions)

---

## 4.7 Summarization: Ninja Katas and SuperSummary

The summarization layer converts rich, structured state into **Katas** — compact, schema-constrained JSON snapshots.

Two core scripts participate:

* **Ninja Summarizer** (`zen_dojotools_ninja_summarizer`)
* **SuperSummary** (`zen_dojotools_supersummary`)

### 4.7.1 Ninja Summarizer: Component-Local Compression

For a given Kung Fu component, Ninja Summarizer:

1. Reads the Dojo drawer and household preferences for that component (`dojo_cabinet`, `household_cabinet`).

2. Optionally consults the Index and any library console output.

3. Pulls the shared `kata_template.structure` from the Kata cabinet.

4. Constructs a prompt object containing:

   * `query`: instructions for how to fill the schema
   * `structure`: the exact JSON skeleton to be honored
   * `example_data`: one or more canonical examples
   * `review_data`: raw contextual data, strongly typed
   * `supplemental_instructions`: trigger-specific notes and last Kata snapshots

5. Invokes an `ai_task.generate_data` entity (e.g., `ai_task.gpt_oss_20b_local_ai_task`) with that prompt.

6. Receives a strictly JSON result in `monk_response.data`.

7. Optionally writes the result as a new drawer under `sensor.zen_kata_storage_cabinet`, keyed by the component slug.

8. Emits a telemetry event via `script.zen_dojotools_event_emitter` summarizing success/failure and excerpting relevant Kata fields.

Each component therefore has:

* a clearly defined schema
* a scheduled or event-driven summarization policy
* an auditable trail of when it was last summarized

The resulting Katas are small enough to be loaded into Friday’s active prompt context without overloading the model.

### 4.7.2 SuperSummary: Whole-Home Consolidation

SuperSummary is a higher-order summarizer that works **only over active components**:

1. Detects which Kung Fu master switches are `on` using the `Kung Fu System Switch` label.

2. For each active component, reads its Kata drawer from the Kata cabinet.

3. Pulls the `zen_template.structure` schema from the Kata cabinet.

4. Optionally injects system-wide context:

   * Friday’s purpose, directives, cortex, and console from `sensor.variables`
   * the previous `ZEN_SUMMARY` drawer content

5. Constructs a prompt object analogous to Ninja’s, but with:

   * `components`: map of component IDs → their current Katas
   * `system.cortex`: core system beliefs
   * explicit instructions to normalize fields and include only active components

6. Calls the same `ai_task.generate_data` worker.

7. Optionally writes the resulting `ZEN_SUMMARY` drawer into the Kata cabinet.

8. Emits an event describing the set of active components, whether a write occurred, and the filecabinet write status.

SuperSummary is the mechanism by which Friday’s higher-level cognition can treat the entire home as a single, summarized object rather than thousands of entities.

---

## 4.8 The Scheduler (Abbot) as Cognitive Orchestrator

The **Zen DojoTools Scheduler** is the orchestration layer that decides **when** each component should be summarized and under which contextual hints.

The Scheduler:

* attaches to time-based triggers (`hourly_trigger`, `quarter_hour`, `daily_midnight`, `daily_noon`)
* listens to key state changes (home mode, alarm state, occupancy, Withings bed sensors, doors, windows, locks, garage doors, water flow, electrical panel doors)
* runs `zen_dojotools_zen_inspect` on `sensor.home_overview` to build high-level context
* for each trigger condition, reads the last Kata for a component and crafts a `supplemental_prompt` that annotates:

  * source trigger metadata
  * current home overview snapshot
  * last known component summary

It then calls Ninja Summarizer or SuperSummary with:

* `kung_fu_component_id` set to a specific manager (Security, Alert Manager, Room Manager, TaskMaster, Media Manager, Energy Manager, Water Manager, Hot Tub Manager)
* `post_to_kata_cabinet: true` for scheduled updates

Finally, it updates a `zen_scheduler` drawer with a JSON record of:

* last trigger ID
* last trigger entity
* status of the most recent summarization sequence

This makes the entire cognitive schedule transparent and allows later versions to adapt the schedule based on observed performance or overload.

---

## 4.9 End-to-End Cognitive Loop

Putting the pieces together, a typical cognitive loop in Version 1 looks like:

1. **Trigger.**
   An external event occurs (door opens, alarm arms, water stops flowing, time hits quarter hour).

2. **Scheduler activation.**
   The Scheduler automation fires, identifies which components should be refreshed, and calls Index, Inspect, or direct filecabinet reads as needed.

3. **Context gathering.**
   Relevant drawers, Katas, and state snapshots are pulled through sanitized access patterns, building a rich `review_data` object.

4. **Monk invocation.**
   Ninja Summarizer or SuperSummary invokes a local AI task with a strictly defined schema and structured example.

5. **Kata emission.**
   The AI returns JSON, which is written into the Kata cabinet and logged via the event emitter.

6. **Integration by Friday.**
   Friday, as the front-line assistant, consumes the latest Katas (or the ZEN_SUMMARY) to drive dialog, explanations, and decisions.

At no point does the system rely on opaque, unstructured “chat logs” as the sole context. Instead, cognition is built on:

* **indexes**
* **inspections**
* **identity manifests**
* **Kata schemas**
* **scheduled summarization**

All of it grounded in Home Assistant’s event semantics and the strict sanitation rules outlined above.
