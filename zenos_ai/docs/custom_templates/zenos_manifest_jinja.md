# zenos_manifest.jinja — Shared Manifest Macro Library

**Version:** 1.0.0 (ZenOS-AI 2026.8.0 'Chef')
**File:** `custom_templates/zenos_ai/zenos_manifest.jinja`
**Status:** New in 2026.7.0. Required for all Level 1+ manifest-compliant tools.

---

## Purpose

`zenos_manifest.jinja` provides the single shared macro that all DojoTools, Sutras, Roots, and AdminTools call to build their `tool_manifest` response. Every compliant tool self-describes — its identity, health, labels it consumes, what it returns, and where it sits in the OS.

Before this library, each tool hand-rolled its own manifest JSON, leading to field drift and schema inconsistency. This macro is the canonical source: add a field here and it propagates to every compliant tool automatically.

---

## Import

```jinja
{%- import 'zenos_ai/zenos_manifest.jinja' as MF -%}
```

Import once at the top of the script's variables step. Do not import inside loops.

---

## Compliance Levels

| Level | Minimum fields | Description |
|-------|---------------|-------------|
| 0 | none | No manifest mode — tolerated, recorded as `compliant: false` |
| 1 | `tool`, `display_name`, `tier`, `version` | Min viable self-description |
| 2 | Level 1 + `health`, `required_labels`, `optional_labels`, lens fields | Full struct |
| 3 | Level 2 + `icon`, `color` | UI styling hints for autotag and dashboard |

---

## `MF.tool_manifest()` — Parameter Reference

```jinja
{{ MF.tool_manifest(
    tool,
    display_name,
    tier,
    version,
    health=None,
    required_labels=None,
    optional_labels=None,
    schema_version='1.0.0',
    consumes=None,
    returns=None,
    failure_policy=None,
    content_policy=None,
    risk_class=None,
    security_act=None,
    modes=None,
    limitations=None,
    evidence_shape=None,
    children=None,
    icon=None,
    color=None,
    managed=None,
    lens_provider=None,
    stack=None,
    mcp_exposed=None,
    self_repair=None,
    audit=None,
    inference=None,
    caller_token=''
) }}
```

### Required parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `tool` | string | Canonical script entity name (e.g., `zen_dojotools_servicedesk`) |
| `display_name` | string | Human-readable tool name (e.g., `Service Desk / Zammad`) |
| `tier` | string | Tool tier — one of `dojotools`, `sutra`, `root`, `admintools` |
| `version` | string | Tool's own semver string (e.g., `0.3.1`) |

### Optional parameters — identity and health

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `health` | mapping | `{'configured': false, 'status': 'unknown', 'notes': ['health not reported']}` | Health block. Must be a mapping with at minimum `configured` (bool) and `status` (string). `notes` is an optional list of strings. |
| `schema_version` | string | `'1.0.0'` | Manifest schema version. Do not change unless the schema itself is versioned. |

### Optional parameters — label contract (lens fields)

These fields are merged into the output at the top level alongside the base identity fields.

| Parameter | Type | Description |
|-----------|------|-------------|
| `required_labels` | list of strings | HA labels this tool requires to function. Omit or pass `None` to suppress. |
| `optional_labels` | list of strings | HA labels this tool uses when present but does not require. |
| `consumes` | list of strings | Semantic input types this tool accepts (e.g., `['label', 'person']`). |
| `returns` | list of strings | Semantic output types this tool produces (e.g., `['ticket_evidence']`). |
| `failure_policy` | any | Describes how the tool behaves on failure. Free-form — string or mapping. |
| `content_policy` | any | Content restrictions or audience rules. |
| `risk_class` | any | Risk classification for this tool's operations. |
| `security_act` | any | Security action or permission requirement. |
| `modes` | list or mapping | Supported operating modes. |
| `limitations` | list or any | Known limitations or constraints. |
| `evidence_shape` | any | Schema description of what the tool writes as evidence. |
| `children` | list | Child tool names for composite/router tools. Must be a sequence. |

### Optional parameters — deployment and Lens Bus

| Parameter | Type | Description |
|-----------|------|-------------|
| `managed` | any | Deployment manager identifier (e.g., `"zenos_ai"` for OS-managed tools). Only emitted when passed. |
| `lens_provider` | bool | `true` if this tool participates in Lens Bus evidence routing. Only emitted when passed. |
| `stack` | string | Stack key this tool registers under in the Lens Bus (e.g., `"firefly_iii"`). Only emitted when passed. |
| `mcp_exposed` | bool | `true` if this tool is exposed to the conversation agent via MCP. Only emitted when passed. |
| `self_repair` | bool | `true` if this tool supports self-repair / auto-remediation on failure. Only emitted when passed. |

### Optional parameters — UI hints (Level 3)

| Parameter | Type | Description |
|-----------|------|-------------|
| `icon` | string | MDI icon string (e.g., `mdi:ticket-outline`). Must be a string or omitted. |
| `color` | string | Color name for dashboard/autotag styling (e.g., `indigo`). Must be a string or omitted. |

### Optional parameters — audit and inference (2026.9.0)

| Parameter | Type | Description |
|-----------|------|-------------|
| `audit` | mapping | Opt-in, describes an audit entry only — the macro cannot emit actions. The calling script must persist it itself (e.g. via `script.zen_dojotools_event_emitter`), same pattern as `essence_signed` events in `dojotools_profile.yaml`. Only merged in when a mapping is passed. |
| `inference` | mapping | Declaration only, e.g. `{'calls_inference': true}`. Set on any tool that itself invokes a model/LLM call. No usage/cost estimation logic yet — designed to grow additively (`estimated_tokens`/`estimated_cost` keys) without breaking tools that only declare `calls_inference`. Only merged in when a mapping is passed. |

### Optional parameters — runtime

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `caller_token` | string | `''` | Passed through from the calling script's `caller_token` variable. Used for tracing. |

---

## Missing-Label Detection (always computed, 2026.9.0)

Unlike every other field above, `missing_required_labels` and `missing_optional_labels` are **not parameters** — they're always computed and returned automatically, checking each tool's own `required_labels`/`optional_labels` (as passed to this same call) against the live label registry (`labels()`). Read-only, no caller action needed. This is the field `zen_dojotools_manifest mode=label_audit` and `mode=repair` aggregate across every tool to build a system-wide label-gap scan (with optional auto-create remediation under `mode=repair` — see that tool's own docs).

---

## OS Version Fields (auto-injected)

The macro reads the `os_release` drawer from `sensor.zen_system_cabinet_resolved` at render time and injects the following fields automatically. Callers do not set these.

| Field | Source |
|-------|--------|
| `os_version` | `os_release.os_version` |
| `os_tag` | `os_release.os_tag` |
| `cortex` | `os_release.cortex` |
| `cortex_codename` | `os_release.cortex_codename` |
| `os_release` | Full `os_release` block |

If the system cabinet is unavailable, these fields default to `'unknown'` or `{}` — the manifest still renders.

---

## Output Format

The macro returns a single-line JSON string. After `| from_json`, the structure is:

```json
{
  "status": "success",
  "tool": "zen_dojotools_servicedesk",
  "display_name": "Service Desk / Zammad",
  "tier": "dojotools",
  "version": "0.3.1",
  "schema_version": "1.0.0",
  "health": {
    "configured": true,
    "status": "ok",
    "notes": []
  },
  "required_labels": ["service_desk"],
  "optional_labels": ["zammad", "tickets", "fulfillment"],
  "missing_required_labels": [],
  "missing_optional_labels": [],
  "icon": "mdi:ticket-outline",
  "color": "indigo",
  "os_version": "2026.8.0",
  "os_tag": "Neo",
  "cortex": "claude-sonnet-4-5",
  "cortex_codename": "Nyx",
  "os_release": { "..." : "..." },
  "caller_token": "",
  "consumes": ["label", "person"],
  "returns": ["ticket_evidence"]
}
```

Lens fields (`consumes`, `returns`, `failure_policy`, etc.) are only present when passed. Absent fields are not emitted as `null` — they are omitted entirely.

---

## Usage Example

Minimal Level 1 call from a tool's `tool_manifest` mode branch:

```jinja
{%- import 'zenos_ai/zenos_manifest.jinja' as MF -%}
{{ MF.tool_manifest(
    tool='zen_dojotools_mynewtool',
    display_name='My New Tool',
    tier='dojotools',
    version='1.0.0'
) }}
```

Full Level 3 call:

```jinja
{%- import 'zenos_ai/zenos_manifest.jinja' as MF -%}
{{ MF.tool_manifest(
    tool='zen_dojotools_servicedesk',
    display_name='Service Desk / Zammad',
    tier='dojotools',
    version='0.3.1',
    health={'configured': true, 'status': 'ok', 'notes': []},
    required_labels=['service_desk'],
    optional_labels=['zammad', 'tickets', 'fulfillment'],
    consumes=['label', 'person'],
    returns=['ticket_evidence'],
    icon='mdi:ticket-outline',
    color='indigo'
) }}
```

In practice, a tool exposes a `tool_manifest` mode branch in its `choose` block:

```yaml
- conditions:
    - condition: template
      value_template: "{{ mode == 'tool_manifest' }}"
  sequence:
    - variables:
        manifest: >
          {%- import 'zenos_ai/zenos_manifest.jinja' as MF -%}
          {{ MF.tool_manifest(
              tool='zen_dojotools_mynewtool',
              display_name='My New Tool',
              tier='dojotools',
              version='1.0.0',
              health=health_check,
              required_labels=['my_label'],
              consumes=['label'],
              returns=['my_evidence']
          ) }}
    - stop: "manifest"
      response_variable: manifest
```

---

## Notes and Constraints

- The macro always returns `"status": "success"` — it is a descriptor, not an executor. Health failures are reported inside the `health` block, not via a top-level error status.
- `required_labels` and `optional_labels` accept a sequence only. Strings are silently replaced with `[]`.
- `children` accepts a sequence only. Non-sequence values are ignored.
- `icon` and `color` must be strings. Non-string values are emitted as `null`.
- The macro does not validate `tier` values. By convention: `dojotools`, `sutra`, `root`, `admintools`.
- OS release fields are best-effort. If `sensor.zen_system_cabinet_resolved` is unavailable, fields degrade to `'unknown'` — do not treat them as authoritative outside the prompt.

---

→ **[Documentation Hub](../readme.md)**
→ **[zen_os_1.jinja reference](zen_os1_jinja.md)**
→ **[zenos_cabinets.jinja reference](zenos_cabinets_jinja.md)**
