# Zen DojoTools Camera — v1.3.0

**File:** `packages/zenos_ai/dojotools/dojotools_camera.yaml`
**Script:** `zen_dojotools_camera`
**CANONICAL HA DOMAIN TOOL — camera domain**

---

## Overview

The Camera tool is Friday's visual surface. It wraps HA camera entities with LLM vision analysis, a household-cabinet cache, and a label-driven sweep mechanism so Friday can reason about what cameras see without burning an LLM call on every question.

The recommended pattern: **read → look → scan**. Check the cache first (`read`). Refresh a single camera when the cache is stale (`look`). Warm the full labeled set on a schedule or event (`scan`). Audit configuration with `info` before deciding whether a refresh is needed.

---

## Modes

| Mode | LLM | Snapshot | FC Write | Description |
|---|---|---|---|---|
| `look` | yes | yes | yes | Analyze single camera, cache result |
| `read` | no | no | no | Return cached description instantly |
| `scan` | yes (per camera) | yes (per camera) | yes (per camera) | Sweep all labeled cameras |
| `info` | no | no | no | Entity attributes + configured `_default_ctx` + `_alert_policy` |
| `set_default_ctx` | no | no | yes | Validate + store per-camera default context call |
| `set_alert_policy` | no | no | yes | Write alert classification policy for this camera |
| `help` | no | no | no | Return schema |

---

## Mode Reference

### `look` — Analyze + Cache

Snapshots the camera, sends the frame to the LLM, caches the result to the household cabinet with an `expires_after` of 8 hours. The LLM receives the live stream via `media-source://camera/<entity>` — the snapshot is a reference copy for human review, not the LLM input.

Auto-loads `_default_ctx` from the camera's cache drawer unless `ignore_autocontext: true` is set or an explicit `context_index_call` is passed.

**Returns:**
```yaml
mode: look
camera_entity: camera.front_doorbell_camera
analysis: "One adult approaching the front door, package in hand..."
cached_at: "2026-04-23T14:32:00"
snapshot_path: "/config/www/tmp/snapshot/camera.front_doorbell_camera_20260423-143200.jpg"
cached: true
fc_confirmed: true
sendto_result: null   # or { target, type, dispatched, ... } if sendto was used
caller_token: ""
```

---

### `read` — Cached Description

Returns the last cached analysis instantly. No LLM call, no snapshot. Use `cache_age_minutes` to decide whether to refresh.

**Returns:**
```yaml
mode: read
camera_entity: camera.front_doorbell_camera
analysis: "One adult approaching the front door..."
cached_at: "2026-04-23T14:32:00"
cache_age_minutes: 12.4
snapshot_path: "/config/www/tmp/snapshot/..."
found: true
caller_token: ""
```

---

### `scan` — Label-Targeted Sweep

Discovers all camera entities carrying `label` (default: `security_camera`), analyzes each in sequence, caches results. Emits `zen_event kind: camera_scan_complete` when done.

Results are readable per-camera via `read` immediately after scan completes. Context index calls apply to the shared `_prompt` — all cameras in the sweep receive the same prompt unless `ignore_autocontext: true`.

`sendto` is ignored in scan mode.

**Returns:**
```yaml
mode: scan
label: security_camera
cameras_scanned: 4
cameras: [camera.front_doorbell_camera, ...]
caller_token: ""
```

---

### `info` — Entity Attributes + Settings

Returns the full configuration picture for a camera: what HA reports about the entity, what labels it carries, and what `_default_ctx` is stored in the cache drawer. No LLM call, no snapshot, no FC write.

Use before `look` to decide if a refresh is warranted. Use after `set_default_ctx` to confirm the stored context.

**Returns:**
```yaml
mode: info
camera_entity: camera.hot_tub_camera_high_resolution_channel
state: streaming
friendly_name: "Hot Tub Camera"
entity_picture: "/api/camera_proxy/camera.hot_tub_camera?token=abc123"
supported_features: 3
is_recording: false
motion_detected: null
brand: "Ubiquiti"
model: ""
labels: [security_camera, hot_tub_manager]
in_scan_scope: true          # carries security_camera label
in_cache_gate: false         # does NOT carry camera_cache label
cache_gate_mode: cache_all   # no cameras labeled camera_cache → cache all
alert_policy:                # stored _alert_policy from drawer (null if not set)
  enabled: true
  classify_on: motion
  notify_target: postman
  urgency: urgent
  channel_hint: security
default_ctx:                 # stored _default_ctx from drawer
  operator: OR
  index_1: { label_1: hot_tub_manager, label_2: hot_tub_deck, operator: OR }
  index_2: { entities_1: [weather.home] }
cached_at: "2026-04-23T14:32:00"
cache_age_minutes: 47.3
snapshot_path: "/config/www/tmp/snapshot/..."
has_cache: true
caller_token: ""
```

---

### `set_default_ctx` — Store Default Context Call

Validates a `context_index_call` via `dry_run` (must return > 0 entities) then writes it as `_default_ctx` to the camera's cache drawer. Rejected without writing if validation fails.

On subsequent `look` calls the stored context is loaded automatically unless overridden.

**Returns:**
```yaml
mode: set_default_ctx
success: true
camera_entity: camera.hot_tub_camera_high_resolution_channel
default_ctx: { ... }
entity_count: 34
fc_confirmed: true
caller_token: ""
```

---

### `set_alert_policy` — Write Alert Classification Policy

Writes a `_alert_policy` drawer to the camera's cache entry. Defines how motion events from this camera are classified and where they route. Readable via `info`.

```yaml
zen_dojotools_camera:
  mode: set_alert_policy
  camera_entity: camera.front_door
  policy_payload: >
    {
      "enabled": true,
      "classify_on": "motion",
      "notify_target": "postman",
      "urgency": "urgent",
      "channel_hint": "security"
    }
```

The policy is preserved across all subsequent `look` and `scan` cycles — private keys (`_*`) are no longer clobbered on write (v1.3.0 fix).

**Returns:**
```yaml
mode: set_alert_policy
camera_entity: camera.front_door
alert_policy: { enabled: true, classify_on: motion, ... }
fc_confirmed: true
caller_token: ""
```

---

## Field Reference

| Field | Mode(s) | Default | Description |
|---|---|---|---|
| `tool` | all | `look` | Mode selector |
| `camera_entity` | look, read, info, set_default_ctx | — | Target entity_id. Required for these modes. |
| `camera_hint` | look, read, info, set_default_ctx | — | Free-text hint — matched against entity_id slug of all `security_camera`-labeled cameras. First match wins. Ignored if `camera_entity` is set. |
| `label` | scan | `security_camera` | Label used to discover cameras for sweep. |
| `instructions` | look, scan | — | Override default LLM prompt. |
| `context_index_call` | look, scan, set_default_ctx | — | Index call dict to run before analysis. Injects entity states as 'Current context' block in the LLM prompt. Camera entity is auto-excluded from its own context. |
| `ignore_autocontext` | look, scan | `false` | When true, skips `_default_ctx` lookup from the drawer. Use for raw looks or when passing an explicit context call you don't want shadowed. |
| `sendto` | look | — | After look completes, dispatch the result. See sendto section below. |
| `caller_token` | all | — | Opaque token echoed in all responses. |

### `context_index_call` schemas

**Flat:**
```json
{ "label_1": "hot_tub", "label_2": "weather", "operator": "OR",
  "expand_entities": true, "output_fields": [], "filter_json": {} }
```

**Compound / recursive (routes through zen_indexer_v2):**
```json
{ "index_command": {
    "operator": "OR",
    "index_1": { "label_1": "hot_tub_manager", "label_2": "hot_tub_deck", "operator": "OR" },
    "index_2": { "entities_1": ["weather.home"] }
  },
  "expand_entities": true }
```

The camera entity is automatically excluded from the context block in both paths so it never appears in its own prompt. ZQ-1 `exclude_entity_ids` injection handles the flat path; a Jinja guard handles the compound path.

---

## `sendto` Field

After `look` completes (analysis + cache write), optionally dispatch to a target. Ignored by all other modes.

### `sendto: image.zen_image_<slot>`

Fires `zen_event kind: image_generated` with the camera's `entity_picture` URL so the zen image slot entity fetches the current frame. Then writes to disk via `image.snapshot` using the same dual-path pattern as `zen_dojotools_generate_image`:

- Primary: `/config/www/zen_images/<slot>.jpg` (`continue_on_error: true`)
- Fallback: `/config/www/zen_<slot>.jpg` (always fires)

Available slots: `canvas`, `portrait`, `wallpaper`, `id_pic`, `dashboard_snap`, `home_state`

The look's `snapshot_path` still points to the tmp archive copy. The zen slot is a separate parallel write.

`sendto_result` in the look response:
```yaml
sendto_result:
  target: image.zen_image_canvas
  type: image_entity
  slot: canvas
  dispatched: true
  paths:
    organized: /config/www/zen_images/canvas.jpg
    fallback: /config/www/zen_canvas.jpg
```

### `sendto: person.*`

Calls `zen_dojotools_postman` in `resolve_and_dispatch` mode with:
- `target`: the person entity
- `message`: the LLM analysis
- `image_url`: the snapshot path converted to `/local/...`
- `urgency: 1`

Respects the full authority stack (sleep gate, urgency tiers, away policy).

`sendto_result` in the look response:
```yaml
sendto_result:
  target: person.alex
  type: person
  dispatched: true
  postman_status: sleep_blocked  # or delivered, etc.
```

---

### `sendto: sensor.*` — Cabinet Entity Reference (v1.3.0)

Pass a cabinet sensor entity ID resolved at call time. The tool looks up the cabinet's member entry in the household roster, retrieves `person_entity_id`, and routes through postman with the snapshot attached.

The caller is responsible for resolving the sensor to an entity ID before passing it:

```yaml
zen_dojotools_camera:
  mode: look
  camera_entity: camera.front_door
  sendto: "{{ states('sensor.zen_default_user_cabinet_resolved') }}"
```

Use this for dynamic routing (e.g., "send to whoever the default user is right now") without hardcoding a `person.*` entity ID in the automation.

`sendto_result`:
```yaml
sendto_result:
  target: sensor.zenos_default_user_cabinet   # the resolved cabinet entity ID
  type: cabinet
  dispatched: true
  postman_status: delivered
```

If the cabinet has no `person_entity_id` wired, `dispatched: false` with `error: no_person_entity_id`.

---

## Caching Architecture

Results are stored in the **household cabinet** keyed by the camera's slugified entity_id (dots and dashes → underscores).

`camera.front_doorbell_camera` → drawer key `camera_front_doorbell_camera`

**Drawer shape:**
```json
{
  "analysis": "One adult approaching...",
  "cached_at": "2026-04-23T14:32:00",
  "camera_entity": "camera.front_doorbell_camera",
  "snapshot_path": "/config/www/tmp/snapshot/...",
  "_default_ctx": { ... }
}
```

All private keys (`_*`) in the camera's cache drawer are preserved across `look` and `scan` cycles. Before writing, both modes read the existing drawer and carry forward any `_*` key (including `_default_ctx` and `_alert_policy`) into the updated entry. This was a v1.3.0 fix — prior versions clobbered private keys on every write.

### Cache Gate Labels

| Label | Behavior |
|---|---|
| `security_camera` | Scan scope — which cameras `scan` analyzes |
| `camera_cache` | Cache gate — which cameras get results persisted |

**No cameras labeled `camera_cache`** → cache all (default-all, safe for new installs).
**Any camera labeled `camera_cache`** → cache only labeled cameras.

The two labels are orthogonal. A camera can carry either, both, or neither.

### Two ways to retrieve cached analysis without calling `look`

1. `zen_dojotools_camera tool=read camera_entity=<id>` — direct drawer read
2. `zen_dojotools_index expand_entities=true` on a camera label — the inspect enrichment includes `domain_context.camera[entity_id]` with `analysis`, `cached_at`, `cache_age_minutes`, `snapshot_path`, `found`

---

## Label Conventions

**Multi-stream cameras** (UniFi G5, G4, etc.) expose hi-res, lo-res, and sub-stream entities per physical camera. Apply `security_camera` to the **preferred stream only** (typically `high_resolution_channel`). `scan` sweeps whatever is labeled — labeling is the de-dupe mechanism.

**Non-security cameras** (Rosie cleaning map, meal image, printer webcam): omit `camera_cache` to prevent their results from landing in the household cabinet. They can still carry `security_camera` and appear in scans.

---

## Examples

```yaml
# 1. Read before you look
- action: script.zen_dojotools_camera
  data: { tool: read, camera_entity: camera.front_doorbell_camera }
  response_variable: cached
# If cached.cache_age_minutes > 15 → look

# 2. Fresh analysis
- action: script.zen_dojotools_camera
  data: { tool: look, camera_entity: camera.front_doorbell_camera }
  response_variable: result

# 3. Specialized prompt
- action: script.zen_dojotools_camera
  data:
    tool: look
    camera_entity: camera.driveway_camera_high_resolution_channel
    instructions: >-
      Focus on vehicle presence and plate visibility. Note any vehicles
      that appear unfamiliar or parked unusually.

# 4. Send to dashboard scratchpad
- action: script.zen_dojotools_camera
  data:
    tool: look
    camera_entity: camera.hot_tub_camera_high_resolution_channel
    sendto: image.zen_image_canvas

# 5. Notify a person
- action: script.zen_dojotools_camera
  data:
    tool: look
    camera_entity: camera.front_doorbell_camera
    sendto: person.member_name

# 6. Audit camera configuration
- action: script.zen_dojotools_camera
  data:
    tool: info
    camera_hint: hot tub

# 7. Store default context
- action: script.zen_dojotools_camera
  data:
    tool: set_default_ctx
    camera_entity: camera.hot_tub_camera_high_resolution_channel
    context_index_call:
      index_command:
        operator: OR
        index_1: { label_1: hot_tub_manager, label_2: hot_tub_deck, operator: OR }
        index_2: { entities_1: [weather.home] }

# 8. Warm all exterior cameras
- action: script.zen_dojotools_camera
  data: { tool: scan, label: security_camera }
```
