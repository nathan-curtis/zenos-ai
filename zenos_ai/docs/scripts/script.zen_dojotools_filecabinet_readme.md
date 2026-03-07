# 📘 Zen DojoTools FileCabinet — v3.8.2 RC1
**File:** `zen_dojotools_filecabinet_readme.md`  
**Type:** Technical Documentation  

---

## Overview

The **Zen DojoTools FileCabinet** is the primary read/write controller for  
**Cabinet Volumes** inside ZenOS-AI.  
Cabinet Volumes function as structured, drawer-based data stores that hold  
Friday’s operational state, context pockets, lookup tables, mappings,  
micro-memories, relationship links, and other structured data.

This module implements **100% health-aware** operations.
All writes — create, update, delete, move, copy — are blocked if a volume is
unhealthy unless explicitly overridden with `force_action`.

The FileCabinet script is the only sanctioned way for an LLM agent  
(Friday, Veronica, Kronk, High Priestess) to **mutate** Cabinet data.

It coordinates:

- drawer creation and updates  
- drawer deletion  
- cross-cabinet moves & copies  
- label assignment + label-index maintenance  
- directory listings  
- label-based reads  
- full volume reads  
- manifest pass-through  
- JSON parsing and coercion  
- write verification  
- concurrent-write protection  
- health validation  

If it writes, moves, or deletes a drawer, it came from here.

---

## Core Responsibilities

### 🗄 Drawer Operations
Implements the full CRUD+M model:

- `create` — create a new drawer with JSON value  
- `update` — update an existing drawer (full overwrite)  
- `delete` — remove a drawer  
- `move` — transfer a drawer to another cabinet + prune index  
- `copy` — copy a drawer to another cabinet  

All writes are re-read and verified for correctness.

### 🔍 Reads (Three Modes)
1. **Direct Read**  
   Read a specific drawer from a specific volume  
2. **Directory**  
   List all drawers (non-system) in a volume  
3. **Label Search**  
   Resolve drawers by label using the `_label_index`  

Each mode returns structured JSON envelopes.

### 🏷 Labeling & Label Index
FileCabinet maintains the volume’s `_label_index`, converting:

```

label → [drawer1, drawer2, ...]

```

Labels are:
- lowercased  
- slugified  
- deduplicated  
- validated against system label registry  

Broken `_label_index` formats (legacy Python-literal strings) are repaired  
in flight.

### 🩺 Health-Aware Write Pipeline
Before any write, FileCabinet checks:

- volume health status  
- GUID mismatch  
- read-only flag  
- schema compatibility  
- storage warnings  

If any red flags exist → write blocked unless `force_action: true`.

### 🔒 Protect-Write Mode (Default)
Blocks writes if the volume’s pre-write state changed during evaluation.

This prevents LLM concurrency glitches.

### 🎯 JSON Parsing & Coercion
All write values are parsed as JSON.

Accepted formats:

- raw JSON (`{}`, `[]`)  
- quoted strings  
- numeric scalars  
- booleans  
- null  

Malformed input produces safe structured error returns.

### 🧭 Manifest Pass-Through
Running:

```

action_type: manifest

```

or:

```

read + volume_entity: "", "*", None

```

redirects to `zen_dojotools_manifest`.

---

## Output Structure

All outputs follow a consistent envelope:

```

{
"status": "success|warning|error|info",
"message": "Human-readable summary",
"result": {...},            # For reads
"entry": {...},             # For writes
"verification": {...},      # For writes
"health_snapshot": {...},   # When blocked by health
"index_maintenance": {...}, # For delete/move index pruning
"warnings": "...",          # Dropped labels, etc.
}

```

---

## Action Model

### `manifest`
Triggers the Manifest script and returns the full Cabinet manifest.

### `create`
- Requires `key` and `value`
- Parses JSON value
- Writes new drawer
- Updates label index
- Verifies write against state

### `read`
Multiple modes:

- Entire volume (`key` empty or `*`)
- Specific drawer
- By label (`label_targets`)
- Global manifest (`volume_entity` = blank)

### `update`
- Requires existing `key` or `force_action`
- Full overwrite only  
- Updates label index  
- Protect-write enforced  

### `delete`
- Removes drawer  
- Cleans `_label_index`  
- Verifies deletion  
- Verifies index pruning  

### `move` / `copy`
- Requires destination cabinet + drawer  
- Moves or copies drawer value  
- Updates target label index  
- For “move” deletes source + prunes source index  

---

## Labeling Model

Drawer labels live under:

```

drawer.meta.entity_labels

```

Labels are:

- trimmed  
- lowercased  
- slugified  
- filtered to system-wide available labels  
- attached at drawer creation/update  

Label targets for reads support:

- comma-separated  
- newline separated  
- list format  
- slugified comparison  

---

## Safety Features

### Protect-Write
Prevents writes if state changed between beginning-of-script and pre-write check.

### Health Blocking
Prevent writes when:

- volume in `error`  
- GUID mismatch  
- storage warning (unless forced)  
- read_only flag  
- schema mismatch  

### JSON Validation
Malformed JSON always returns a structured error.

### Concurrent-Operation Awareness
Every write is re-read and compared against the intended value.

---

## Usage Examples

### Directory of a volume
```

action_type: read
volume_entity: sensor.household_cabinet
key: ""

```

### Read drawers by label
```

action_type: read
volume_entity: sensor.household_cabinet
label_targets: "profile, preferences"

```

### Create drawer
```

action_type: create
volume_entity: sensor.household_cabinet
key: favorite_color
value: "blue"
set_timestamp: true
labels: "preferences, profile"

```

### Move drawer
```

action_type: move
volume_entity: sensor.source_cabinet
key: settings
destination_cabinet: sensor.dest_cabinet
destination_drawer: settings_moved
force_action: true

```

---

## Why FileCabinet Exists

The FileCabinet module is Friday’s **write-controller**,  
her filesystem driver, her data gatekeeper.

It enforces:

- health  
- safety  
- consistency  
- structure  
- label intelligence  
- schema boundaries  
- concurrent write protection  

Without FileCabinet:

- drawers could corrupt  
- labels could become inconsistent  
- manifests could desync  
- volumes could lose schema integrity  
- LLMs could overwrite each other  
- the system would drift  

This is the **authoritative, hardened interface** that lets Friday mutate state  
*without breaking her world.*

---

## Summary

The Zen DojoTools FileCabinet RC1 provides:

- A unified read/write abstraction for Cabinet volumes  
- Zero-trust health gating  
- Strict JSON handling  
- Drawer labeling + index management  
- Volume directory + label search tools  
- Cross-volume mutation operations  
- Write verification + state consistency  
- Manifest integration  
- Full protection against concurrency, schema mismatches, and unhealthy volumes  

If it reads like a filesystem, feels like a KV store, smells like a structured persistence layer, and behaves like a transactional controller?

Yeah.  
That’s FileCabinet.
