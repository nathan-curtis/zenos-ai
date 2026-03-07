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

## **1. Zen DojoTools Calendar — v1.10.3**

**File:** `zen_dojotools.calendar_readme.md`
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

## **2. Zen DojoTools Event Emitter — v1.1.2**

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

## **3. Zen DojoTools FileCabinet — v3.8.2 RC1**

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

## **4. Zen DojoTools Manifest — v3.8.0 RC1**

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

## **5. Zen DojoTools Zen Index — v3.8.0 RC1**

**File:** `zen_dojotools_zen_index_readme.md`
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

## **6. Zen DojoTools Inspect — v3.8.0 RC1**

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
