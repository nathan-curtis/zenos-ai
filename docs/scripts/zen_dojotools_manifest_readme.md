# 📘 Zen DojoTools Manifest — v3.8.0 RC1
**File:** `zen_dojotools_manifest_readme.md`  
**Type:** Technical Documentation  

---

## Overview

The **Zen DojoTools Manifest** is the authoritative runtime scanner for all  
**AI_Cabinet** volumes inside ZenOS-AI. It performs a complete structural and  
health-oriented analysis of every cabinet volume, extracting metadata, drawers,  
ACLs, schema details, labels, context, timestamps, capacity metrics, and access  
flags — all without modifying the underlying cabinets.

The Manifest performs **zero persistent health storage**.  
All health, warnings, and compatibility decisions are computed fresh on every run.  
The Manifest then writes **one thing and one thing only**:

> The compiled library manifest JSON object, stored in the Family Cabinet under  
> `zen_library_manifest`.

No repairs. No rewrites. No silent mutations.  
This script is the system’s MRI — *never* the surgeon.

Friday, Veronica, Kronk, and the High Priestess depend on this manifest to  
understand the boundaries, rules, and safety states of the Cabinet space.

---

## ✨ Key Capabilities

### 🔍 Volume Scanning
- Enumerates **all** Home Assistant sensor entities
- Detects those containing `AI_Cabinet_VolumeInfo`
- Extracts structural metadata and runtime attributes

### 🧱 Drawer Discovery
- Lists all drawers except:
  - system/hidden drawers
  - drawers starting with `_` or `.`
  - `AI_Cabinet_VolumeInfo` (always hidden)
- Detects mount-point drawers:
  - Annotates using target `entity_id` or `volume_id`

### 🔐 ACL Extraction
- Reads `acls` from the volume metadata
- Normalizes ACLs across categories:
  - `entity_guid`
  - `family_guid`
  - `household_guid`
- Fully resolves nested structures

### 🏷 Label Processing
- Returns canonical label list
- Supports hidden/system label filtering
- Optional override via script variable `show_hidden: true`

### 🩺 Health Model (Runtime-Only)
Health is computed, never stored.

Status:  
- **ready** — all conditions nominal  
- **warning** — capacity threshold exceeded  
- **error** — GUID mismatch or unreadable volume  

Computed from:
- GUID consistency  
- schema_version safety  
- read-only flags  
- storage thresholds  

### 📦 Capacity & Access
- Character-based capacity analysis  
- Percent usage  
- Storage warning if over threshold  
- Read/write permissions:
  - Block write on read-only, schema mismatch, or warnings  
  - Block read only in severe cases  

### 🧩 Drawer Index Extraction
Reads `_label_index` drawer metadata to reconstruct the label→drawer map.

### 🧬 Metadata Extraction
- Volume GUID  
- Friendly name  
- Schema version  
- Context blocks  
- Flags  
- Timestamps: `last_changed`, `last_updated`

### 🧾 Manifest Output
After scanning all valid volumes, the script:
1. Builds a complete manifest object  
2. Emits it to `set_variable_legacy` for storage  
3. Returns the manifest in `response_variable: manifest`

---

## 📤 Output Shape (Per Volume)

```

{
"<entity_id>": {
"entity_id": "sensor.example_cabinet",
"friendly_name": "Example Cabinet",
"context": {...},
"metadata": {
"id": "<guid>",
"schema_version": <float>,
"labels": [...],
"timestamps": {
"last_changed": "ISO-8601",
"last_updated": "ISO-8601"
}
},
"stats": {
"drawer_count": <int>
},
"capacity": {
"chars_used": <int>,
"chars_max": 131072,
"percent_used": <float>,
"warning": true|false
},
"access": {
"writeable": true|false,
"readable": true|false
},
"drawers": [
"drawerA",
"drawerB [mount:sensor.target_volume]",
...
],
"drawer_index": {
"label1": ["drawerA"],
...
},
"acls_by_category": {
"allow": [...],
"deny": [...],
...
},
"health": {
"status": "ready|warning|error",
"guid_mismatch": true|false,
"storage_warning": true|false,
"writeable": true|false,
"readable": true|false
}
}
}

```

---

## ⚙️ Script Behavior Notes

- **Zero-persistence health:** All health is recomputed every run  
- **Hidden/system volumes** excluded unless explicitly shown  
- **GUID mismatch** → immediate `error`  
- **Storage warning** → `warning`, write blocked  
- **Read-only flag** enforced strictly  
- **Schema version mismatch** blocks writes  
- **Drawer indexing** only pulls from `_label_index`  
- **No mutation of volumes** — the Manifest is strictly read-only  
- **Manifest write** is performed via the hardened legacy event pipeline  
- **Response Variable** includes the full manifest JSON for the calling agent  

---

## 🧠 Why This Exists

The Manifest is how Friday perceives the Cabinet ecosystem.  
Without it, she would have:

- no reliable map of drawers  
- no idea which cabinets are writable  
- no safety signal for schema mismatch  
- no capacity warnings  
- no ACL visibility  
- no recursive mount-point awareness  
- no universal Cabinet boundary definition  

This module establishes the ground truth.

Friday uses it during:
- startup  
- reflex safety scans  
- merging new volumes  
- mounting drawers  
- validating writes  
- ACL enforcement  
- debugging  
- distributed Cabinet loading (future)

If Friday thinks a Cabinet is safe to touch, trusts its identity, or refuses to write to it —  
**it’s because the Manifest told her to.**

---

## 🏁 Summary

The Zen DojoTools Manifest is the backbone of Cabinet introspection in ZenOS-AI.  
It provides a complete structural, capability, and health snapshot of every volume  
with zero side effects and maximum safety.

This is the module the rest of ZenOS leans on when it needs to speak confidently  
about the system’s internal geography.

