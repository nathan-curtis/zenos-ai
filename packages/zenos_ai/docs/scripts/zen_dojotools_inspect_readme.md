# 📘 Zen DojoTools Inspect — v3.8.0 RC1
**File:** `zen_dojotools_inspect_readme.md`  
**Type:** Technical Documentation  

---

## Overview

The **Zen DojoTools Inspect** module is ZenOS-AI’s structured introspection engine.  
It performs a safe, deterministic deep dive on one or more Home Assistant entities,
returning a sanitized, LLM-safe snapshot of everything an agent is allowed to see.

Its mission is simple and strict:

> **Show everything that is safe to show.  
> Never reveal anything that could break the system.  
> Never mutate anything.  
> Never touch Cabinet internals.**

Cabinet volumes are identified and surfaced via **header-only metadata**,  
ensuring agents like Friday or Veronica can *recognize* cabinets  
without being able to tamper with them (that stays the job of FileCabinet).

Inspect is the **eyes** of ZenOS-AI.

---

## Core Capabilities

### 🔍 Entity Snapshot (Safe-Mode)
For each entity inspected, the module returns:

- entity_id  
- friendly_name  
- state  
- domain  
- timestamps  
- labels  
- sanitized attributes  
- cabinet header metadata (if applicable)  
- Home Assistant statistics eligibility  
- optional extended device/integration forensics  

Unsafe or unserializable values (objects, unsupported types, HA stringified dicts)  
are automatically normalized into JSON-compatible equivalents.

This guarantees *all* Inspect results are future-safe for LLM consumption.

---

### 🗄 Cabinet Sensor Recognition (Header-Only)
Inspect can detect Cabinet entities by signature:

```

AI_Cabinet_VolumeInfo → validation == "ALLYOURBASEBELONGTOUS"

```

When identified, Inspect returns ONLY:

```

{
"is_cabinet": true,
"id": "...",
"version": "...",
"friendly_name": "...",
"name": "...",
"description": "...",
"flags": {...},
"uninitialized": true|false
}

```

No drawers  
No volumes  
No internal storage  
No Cabinet contents  
No label indexes

Cabinet internals remain hidden by design —  
LLMs must use FileCabinet or Manifest for that.

---

### 🏷 Attribute Sanitization (String-Safe)
Inspect implements two safety layers:

1. **Normal Mode:**  
   Attributes already in JSON-safe format pass through untouched.

2. **Fallback Mode:**  
   If Home Assistant stringified the attribute block (common for large or exotic integrations),  
   Inspect safely rebuilds a JSON-compatible dict by individually extracting attributes via state_attr.

This means the output is always:

- safe  
- predictable  
- consistent  
- valid JSON  

Perfect for Friday’s consumption.

---

### 📊 Statistics Eligibility Engine
Each entity is evaluated for HA long-term statistics:

- measurement  
- total  
- total_increasing  

Eligibility is determined by evaluating:

- state_class  
- unit_of_measurement  
- presence of state object  
- supported or unsupported stat classes

Results include:

- eligibility flag  
- machine-readable reason  

Friday uses this in analytics, trend analysis, and future forecasting modules.

---

### 🔧 Extended Mode (Device/Integration Forensics)
When `extended: true`:

Inspect extracts:

- device_id  
- area_id  
- identifiers  

This is limited but highly useful for:

- correlating device groups  
- resolving integration IDs  
- mapping room context  
- cross-checking entity clusters  

Extended mode never exposes harmful or overly deep device metadata.

---

### 🧠 Infer Mode (Reserved)
Field: `infer`

Currently reserved for future local inference modules (O3/O1 sidecar work).  
The field is parsed and surfaced but does not activate behavior yet.

---

### ⏱ Timeout Behavior
The script itself performs no long-running operations,  
but the field is included to maintain compatibility with future multi-device or  
multi-integration deep-walk inspection flows.

Default: `60s`  
Valid range: 30–300 seconds

---

## Output Structure

Inspect outputs a unified, LLM-safe envelope:

```

{
"results": [
{
"entity_id": "...",
"friendly_name": "...",
"domain": "...",
"state": "...",
"last_changed": "ISO-8601",
"last_updated": "ISO-8601",
"labels": [...],
"attributes": {...},
"volume": {...},         # Cabinet header only
"statistics": {
"eligible": true|false,
"reason": "..."
},
"extended": {...}        # optional, see Extended Mode
},
...
],
"inputs": {
"entity_id": [...],
"extended": false,
"infer": false,
"timeout": 60
},
"error": null
}

```

All entities appear in the order provided.

---

## Behavioral Notes

### 1. Multi-Entity Safe Iteration
Input entities may be a single value or list.  
Inspect normalizes internally:

```

entity_list = [ ... ]

```

### 2. Labels Are Always Safe
Labels are always returned as a clean list;  
string edge cases are normalized.

### 3. Cabinet Sensors Are Protected
Agents cannot read drawers or Cabinet contents through Inspect.  
Volume module signatures are exposed so agents don’t “guess.”

### 4. Unknown Entities Return Clean Placeholder Data
If an entity does not exist, Inspect still returns a well-formed entry.

### 5. Every Attribute Is Key-Scanned
Every value is inspected for:

- type  
- safety  
- serializability  

Then normalized accordingly.

### 6. No Mutations Ever
Inspect is a read-only module—  
never writes, never emits events, never alters global state.

---

## Example Calls

### Inspect a single entity
```

entity_id: sensor.living_room_temperature
extended: false

```

### Inspect with extended device metadata
```

entity_id:

* sensor.office_climate
  extended: true

```

### Multiple entities
```

entity_id:

* sensor.a
* switch.kitchen_lights

```

### Cabinet sensor recognition
```

entity_id: sensor.family_cabinet

```

Inspect returns Cabinet header metadata only.

---

## Why Inspect Exists

The ZenOS-AI runtime needs a **safe, guaranteed-clean introspection layer**  
that agents can rely on without ever risking:

- cabinet corruption  
- unsafe attribute access  
- mis-typed values  
- weird HA internals  
- stringified dict hazards  
- recursive/unserializable objects  
- cabinet drawer leakage  
- inconsistent entity metadata  

Inspect ensures *every agent sees the same world, the same way.*

Friday depends on Inspect during:

- reflex scans  
- entity correlation  
- Zen Index expansion  
- FileCabinet preflight checks  
- security policy evaluation  
- anomaly detection  
- user command explanation  
- self-awareness and internal narration  

If she says “I understand this entity,”  
it's because Inspect told her what it is — safely.

---

## Summary

The Zen DojoTools Inspect RC1 provides:

- multi-entity snapshot  
- fully sanitized attributes  
- safe cabinet identification  
- statistics eligibility detection  
- optional device forensics  
- JSON-compatible, LLM-stable outputs  
- strict no-write behavior  
- guaranteed safety against HA quirks  

Inspect is Friday’s **safe x-ray machine**.  
It shows everything she needs —  
and nothing she shouldn’t see.
