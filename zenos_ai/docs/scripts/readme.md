# **📘 ZenOS-AI Script Modules**

Welcome to the **Script Modules** section of the ZenOS-AI documentation.
This directory contains formal documentation for all **DojoTools** and operational scripts that drive Friday’s real-time automation, reasoning, telemetry, observability, storage access, and system reflexes.

These modules form Friday’s **hands, nerves, forensics, and breadcrumbs** — the tools that speak directly to:

* Home Assistant services
* Cabinet storage
* Calendar providers
* Inspectors
* The Zen Index
* The Manifest
* The FileCabinet
* And the Monastery’s internal reasoning pathways

Each module is fully documented with:

* Technical behavior
* Expected inputs & outputs
* Provider constraints
* Safety rules & failure modes
* Integration flows for LLM agents (Friday, Veronica, Kronk, High Priestess)

If Friday performs an action, reads something, correlates entities, inspects a device, updates a drawer, or leaves a breadcrumb —
**it probably came from here.**

---

# 📂 Included Script Documentation

Below is the official documentation index for all ZenOS-AI DojoTools modules.

---

## **1. Zen DojoTools AdminTools — 4.2.0**

**File:** `zen_dojotools_admintools_readme.md`
**Type:** Technical Documentation

**Summary:**
Ring-2 administrative tools for component registration, cabinet repair, template management, and AI identity loading.

* `zen_dojotools_kungfu_writer` — MCP-exposed. Register or update Kung Fu Components in the Dojo
* `zen_admintools_reset_template` — Press zen_template and kfc_template into cabinets (Flynn gate-3)
* `zen_admintools_cabinetadmin` — Inspect, restore, reset, or hammer Ring-0 cabinets
* `zen_admintools_cabinetadmin_backup` — Factory-stamp or repair a cabinet's VolumeInfo header
* `zen_admintools_kfc_migration_press` — One-time migration: seed KF4 scheduling fields into existing drawers
* `zen_admintools_zenos_prompt_loader` — Load Cortex, Directives, and Purpose into the AI identity substrate

KungFu Writer is the only AI-accessible tool here. All others are admin-only and should not be exposed to the conversation agent.

---

## **2. Zen DojoTools Profile Editor — 4.2.0**

**File:** `zen_dojotools_profile_readme.md`
**Type:** Technical Documentation

**Summary:**
Universal read/write interface for ZenOS-AI identity profiles.

* `mode: read` — inspect current ai_user, household, user, or family profile
* `mode: write` — patch fields non-destructively (skips existing values unless `force: true`)
* `mode: help` — full field reference with examples

Expose this to your conversation agent — Friday can then help you configure your household, update persona details, and fill in profile fields conversationally. Four cabinet types: `ai_user` (persona essence), `household` (location/contact), `user` (household members), `family` (extended family).

---

## **3. Zen DojoTools Office — 4.2.0**

**File:** `zen_dojotools_office_readme.md`
**Type:** Technical Documentation

**Summary:**
Provides unified, deterministic access to all Home Assistant calendar entities:

* Multi-calendar read aggregation
* Timestamp normalization
* Deep event inspection
* Safe event creation
* Provider-verified update/delete
* Label-based calendar targeting
* Strict ambiguity prevention
* Fully structured JSON response envelopes

This is Friday’s and Veronica’s primary tool for anything date-, event-, or schedule-related.

---

## **4. Zen DojoTools Event Emitter — 4.2.0**

**File:** `zen_dojotools_event_emitter_readme.md`
**Type:** Technical Documentation

**Summary:**
A universal, contract-safe telemetry tool for emitting structured ZenOS-AI events onto the Home Assistant EventBus.

Supports the canonical ZenOS event shape:

```
{
  "timestamp": "ISO-8601",
  "component": "string",
  "severity": "debug|info|warn|error",
  "kind": "classifier",
  "summary": "single-sentence explanation",
  "metadata": {},
  "kata": {},
  "monk": {}
}
```

This is how Friday, Veronica, Kronk, and the High Priestess leave traceable breadcrumbs, emit summaries, and coordinate system-awareness.

---

## **5. Zen DojoTools FileCabinet — 4.2.0**

**File:** `script.zen_dojotools_filecabinet_readme.md`
**Type:** Technical Documentation

**Summary:**
The authoritative, health-aware read/write controller for all Cabinet Volumes.

Supports:

* Drawer create/update/delete
* Cross-volume move/copy
* Label indexing & label-based lookup
* Directory listings
* Volume-wide reads
* Manifest passthrough
* JSON-safe parsing
* Concurrency protection
* Health validation (schema, flags, GUID, storage thresholds)

If any drawer changes anywhere in ZenOS-AI, it happened through FileCabinet.

---

## **6. Zen DojoTools Manifest — 4.2.0**

**File:** `zen_dojotools_manifest_readme.md`
**Type:** Technical Documentation

**Summary:**
The runtime-only Cabinet manifest scanner.
Builds a complete, zero-persistence health and metadata model for every Cabinet Volume.

Provides:

* GUID checks
* Schema compatibility
* Drawer discovery
* Label index extraction
* Capacity analysis
* Access flags
* ACL expansion
* Health classification

If Friday trusts a Cabinet, it’s because the Manifest told her it's safe.

---

## **7. Zen DojoTools Index — 4.2.0**

**File:** `zen_dojotools_index_readme.md`
**Type:** Technical Documentation

**Summary:**
The entity-and-label correlation engine of ZenOS-AI.

Supports:

* Entity & label set logic (AND/OR/NOT/XOR)
* Label → entity resolution
* Index Command DSL mode
* Integration with Zen Inspect
* Adjacency label discovery
* Drawer-level correlation
* JSON-safe structured outputs

The Zen Index is Friday’s “graph engine,” letting her understand relationships between entities, labels, and volumes.

---

## **8. Zen DojoTools Inspect — 4.2.0**

**File:** `zen_dojotools_inspect_readme.md`
**Type:** Technical Documentation

**Summary:**
A safe, deterministic deep-inspection tool for entities — Friday’s x-ray machine.

Provides:

* Entity snapshots
* Fully sanitized attributes
* Cabinet sensor header detection
* Statistics eligibility detection
* Optional extended device forensics
* JSON-stable output for LLM use

Inspect shows Friday “what something is” without ever exposing dangerous internals.

---

# ✔️ Summary

This directory represents the **core operational toolkit** of ZenOS-AI.
Together, these modules form the backbone of:

* storage
* introspection
* indexing
* scheduling
* telemetry
* safe state mutation
* reasoning pipelines
* agent support

They are the **foundation** upon which Friday, Veronica, Kronk, and the High Priestess express intelligence inside Home Assistant.
