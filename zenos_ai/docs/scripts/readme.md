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

## **2. Flynn — Stepgate Sentinel & Bootstrap Engine — 4.2.0**

**File:** `zen_flynn_readme.md`
**Type:** Technical Documentation

**Summary:**
ZenOS-AI's boot guard, initializer, and OOBE driver.

* Gates 0–4: labels, cabinet init, schema seed, content bootstrap, system ready
* Early exit when system is clean and stable
* OOBE flow: names the AI, seeds household profile, flags completion
* Agent Builder (MCP-exposed): interactive config through conversation
* Auto-resolves reasoning task and AI task entity at bootstrap

If the system isn't coming up clean, Flynn is why — and where to look first.

---

## **3. Zen DojoTools Profile Editor — 4.2.0**

**File:** `zen_dojotools_profile_readme.md`
**Type:** Technical Documentation

**Summary:**
Universal read/write interface for ZenOS-AI identity profiles.

* `mode: read` — inspect current ai_user, household, user, or family profile
* `mode: write` — patch fields non-destructively (skips existing values unless `force: true`)
* `mode: help` — full field reference with examples

Expose this to your conversation agent — Friday can then help you configure your household, update persona details, and fill in profile fields conversationally. Four cabinet types: `ai_user` (persona essence), `household` (location/contact), `user` (household members), `family` (extended family).

---

## **3. Zen DojoTools Labels — 4.2.0**

**File:** `zen_dojotools_labels_readme.md`
**Type:** Technical Documentation

**Summary:**
Core primitive for label CRUD and entity tagging. Backbone of the label index that powers HyperIndex, KFC components, and cabinet resolution.

* `create` / `delete` — manage labels (confirm required)
* `read` — full index or filtered by label/entity
* `tag` / `untag` — assign or remove labels from entities

Includes label design guidance: cross-entity labels outperform single-use labels; layer broad domain labels with specific sub-labels for richer HyperIndex traversal.

---

## **4. Zen DojoTools Identity — 4.2.0**

**File:** `zen_dojotools_identity_readme.md`
**Type:** Technical Documentation

**Summary:**
Identity resolver for ZenOS-AI household members and AI constructs. Stateless, read-only, MCP-exposed.

* No target → full redacted roster
* Target (label, person entity, cabinet entity, or GUID) → single identity record
* Normalizes and validates all input forms before lookup
* Delegates resolution to `zen_cabinets()` macro in zen_os_1rc.jinja

Current scope is resolution only. Privilege enforcement, session tokens, and security masking are planned post-GA.

---

## **5. Zen DojoTools Core (FileCabinet GC) — 4.2.0**

**File:** `zen_dojotools_core_readme.md`
**Type:** Technical Documentation

**Summary:**
FileCabinet garbage collector — enforces the drawer hide/delete/recycle lifecycle across all AI Data Storage cabinets.

* `gc` — full daily scan (scheduled at midnight via Scheduler)
* `recycle` — hide a drawer + set 24h expiry
* `hide` / `unhide` — manual visibility control
* `dry_run` — preview what would be evicted

Protected drawers (`_prefix`, VolumeInfo, schema keys) are never touched. Post-eviction SuperSummary trigger built in.

---

## **6. Zen DojoTools SystemTools & Home Mode — 4.2.0**

**File:** `zen_dojotools_systemtools_readme.md`
**Type:** Technical Documentation

**Summary:**
HA lifecycle management, log reading, event emission, and the home mode engine.

* `zen_dojotools_systemtools` — Config check, safe restart (auto-validates before restarting), update install/skip
* `zen_dojotools_ha_log_viewer` — Five log modes + HA 2025.11+ journal detection
* `zen_dojotools_event_emitter` — Structured zen_event emission (see also dedicated emitter readme)
* `zen_dojotools_ha_api` — Internal API wrapper (do NOT expose)
* Home Mode — 8-state time-scheduled mode lifecycle + presence detection, configurable anchors

Requires a long-lived HA token in `secrets.yaml` (ha_bearer).

---

## **4. Zen DojoTools Office — 4.2.0**

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

**File:** `zen_dojotools_filecabinet_readme.md`
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
