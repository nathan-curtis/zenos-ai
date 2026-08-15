# KF5 — Self-Registering KFC Components (2026.7.0)
**File:** `zen_kf5_pattern_readme.md`
**Type:** Architecture Guide (cross-cutting pattern, not a single script)

---

## Overview

KF5 is the fifth generation of the Kung Fu Component (KFC) registration pattern. A tool that owns a domain also owns its KFC registration — no hand-provisioning, no drift between what a tool does and what the Dojo cabinet says it does.

Before KF5, a component's summarizer instructions lived as a hand-authored blob directly in the Dojo cabinet drawer, maintained separately from the tool that actually implements the behavior. Any change to the tool required a matching manual edit to the drawer, and the two silently drifted apart over time.

Under KF5, the tool script itself exposes a `mode=kfc_manifest` that returns its own KFC definition. `zen_dojotools_manifest mode=bootstrap_kfc` discovers all such tools and writes a **live drawer mount** — a pointer, not a copy — into the Dojo cabinet. Add a tool, run bootstrap, done.

---

## The three Dojo cabinet readers

The Dojo cabinet (`sensor.zenos_dojo_cabinet`) has exactly three consumers, each reading a different subset of fields, through two different access paths:

| Reader | Access path | Fields it needs |
|---|---|---|
| **Ninja Summarizer** (`zen_dojotools_ninja_summarizer`) | `zen_dojotools_filecabinet action_type=get` — resolves the live mount by calling the tool script fresh | Everything, but fetched live from the tool's own `kfc_manifest` response — never from the stored drawer copy |
| **Prompt loader** (`dojo_loader()` macro, `zen_os_1.jinja`) | Raw `state_attr(dojo_cab, 'variables')` — a Jinja macro rendered into the live conversation prompt; cannot make service calls, so it can never resolve a mount | `friendly_name`, `label`, `meta.enabled`, `component_summary` (as a description fallback) |
| **Scheduler dispatcher** (`zen_dojotools_scheduler`) | Raw `state_attr(dojo_cab, 'variables')` — a hot dispatch loop, same constraint as the prompt loader | `trigger_subscriptions`, `delay_seconds`, `pipeline_tier`, `schedules` |

This split matters for drawer design: only the Ninja Summarizer resolves the live mount, so anything large (like full monk instructions) never needs to be duplicated into the stored drawer — it only needs to reach the tool script, which the live mount already calls on demand. The prompt loader and scheduler read the drawer raw, so whatever they need must be physically present in storage.

---

## Drawer shape

A KF5 live-mount drawer contains only the fields the raw readers need:

```yaml
mount_point: true
fc_args:
  tool: zen_dojotools_<name>
  params: {mode: kfc_manifest, component: <kata_key>}
friendly_name: <str>                  # prompt loader
label: <str>                          # prompt loader
seed: {...} | null
trigger_subscriptions: <comma-str>    # scheduler dispatch
drift_threshold: <int>
emission_cooldown_minutes: <int>
delay_seconds: <int>                  # scheduler dispatch
kata_key: <str>
meta: {enabled: bool, requires: {cert: str, level: int}}
```

**Deliberately absent:** `component_instructions` — the full monk-facing instructions text lives only in the tool script, fetched fresh by the Ninja Summarizer's live-mount resolution. Storing it in the drawer would be pure duplication (nothing reads it there) and was the single largest contributor to Dojo cabinet bloat before this was corrected.

Also deliberately absent (as of 2026-07-04): `component_summary`. The prompt loader resolves an empty `component_summary` from the HA label's own description at read time — a 1:1 mapping with the tool's declared value — so persisting it in the drawer was pure duplication too. A `kfc_manifest` response may still return `component_summary` for callers that want it directly (e.g. index/report listings); it just never gets written into the cabinet drawer.

Also absent: legacy per-component metadata that nothing raw-reads (`area_seed`, `artifact_class`, `component_group`, `index_call`, `scope_summary`, `interpretation`, `policy_overlays`, `scope_safety`, `schema_version`). Tools that need this data for their own internal logic keep it in their own `_kfc_defs`; it simply isn't copied into the cabinet.

> **Known gap:** the current bootstrap write does not include a `schedules` key. Any component whose own `kfc_manifest` declares per-schedule sub-runs (the pattern `room_manager` uses) will not get that expansion once migrated to a live mount, until `schedules` is added to the write payload. Audit this before migrating any schedule-bearing component.

---

## Building a new KF5 component

**1. File and naming.** New file at `packages/zenos_ai/dojotools/dojotools_<name>.yaml` (core), or an equivalent path under your own non-core package directory for a personal/custom component. Script entity: `zen_dojotools_<name>`.

**2. Required modes:** `status`, `help`, `kfc_manifest`, `tool_manifest`. `kfc_manifest` is the contract — it returns the full component list when called with no `component` field, or a single component dict when `component=<kata_key>` is passed (used by FC's live-mount resolution).

```yaml
script:
  zen_dojotools_<name>:
    alias: "Zen DojoTools <Friendly Name>"
    fields:
      mode:
        default: status
        selector:
          select: {options: [status, help, kfc_manifest, tool_manifest]}
      component:
        selector: {text:}
      caller_token:
        selector: {text:}
    sequence:
      - variables:
          _mode: "{{ mode | default('status') | lower | trim }}"
          _caller_token: "{{ caller_token | default('') | string | trim }}"

      # ... tool_manifest, help, status branches ...

      - variables:
          _kfc_defs:
            - kata_key: <name>
              friendly_name: <Friendly Name>
              label: <label>
              component_summary: >-
                <short summary — reaches the live prompt>
              component_instructions: >-
                <full monk instructions — large text is fine, never touches the cabinet>
              tool: ""
              seed: null
              trigger_subscriptions: "hourly,force_summary"   # comma STRING, never a YAML list
              drift_threshold: 5
              emission_cooldown_minutes: 60
              delay_seconds: 30
              suggested_act_event: zen_dojotools_ninja_summarizer
              meta: {enabled: true, requires: {cert: '', level: 0}}

      - variables:
          _component: "{{ component | default('') | lower | trim }}"
      - variables:
          result: >-
            {%- if _component == '' -%}
              {{ {'mode': 'kfc_manifest', 'result': 'ok', 'status': 'ok',
                  'caller_token': _caller_token, 'kfc': _kfc_defs} }}
            {%- else -%}
              {%- set match = _kfc_defs | selectattr('kata_key', 'equalto', _component) | list | first | default(none) -%}
              {{ {'mode': 'kfc_manifest', 'result': 'ok' if match is not none else 'not_found',
                  'status': 'ok' if match is not none else 'not_found',
                  'caller_token': _caller_token, 'kfc': match} }}
            {%- endif -%}
      - stop: "<name> kfc manifest"
        response_variable: result
```

> **Footgun:** `trigger_subscriptions` must be a comma-separated string, never a native YAML list. A list value causes `TypeError: can only concatenate str (not "list") to str` inside HA's whitespace-stripped Jinja templates.

**3. Deployment sequence.** Every step below matters — several failure modes are silent (no error raised) and look identical to success unless checked explicitly:

| Step | Action | What to verify |
|---|---|---|
| 1 | Write the script file | — |
| 2 | `zen_dojotools_systemtools mode=ha_config_check` | `result: go` |
| 3 | `zen_dojotools_systemtools mode=ha_reload_scripts` | Reload is deferred — returns immediately, doesn't mean it's done |
| 4 | Wait ~8s | Reload is async |
| 5 | `zen_dojotools_inspect entity_id=[script.zen_dojotools_<name>]` | `state: off`, real `friendly_name` — NOT `state: unknown` / `friendly_name: null` (a ghost stub means the reload hasn't landed yet) |
| 6 | `zen_dojotools_labels mode=tag label_list=[zen_kfc_provider] target_entities=[script.zen_dojotools_<name>] confirm=true` | `verification.confirmed: true` in the response, not just `status: ok` — tagging a not-yet-loaded entity returns `status: warning` and silently fails to apply |
| 7 | `zen_dojotools_manifest mode=bootstrap_kfc` | Your `kata_key` appears in `registered[]` with `written: true`; `errors: []` |
| 8 | `zen_dojotools_ninja_summarizer mode=run kung_fu_component_id=<kata_key> post_to_kata_cabinet=false force=true` | Response includes full `component_instructions` in `dojo_drawer` — confirms live-mount resolution is working end to end |

Discovery for bootstrap is driven entirely by the HA label `zen_kfc_provider` on the script entity — there is no YAML-side registration list.

---

## Write semantics caveat

`zen_dojotools_filecabinet action_type=upsert` (and `create`) default to **deep-merge** with the existing stored value — dropping a key from a write payload does not remove it from storage, the old value persists underneath. Only `action_type=update` defaults to replace, and it requires the key to already exist unless `force_action=true` is also passed. `bootstrap_kfc`'s live-mount write therefore uses:

```yaml
action_type: update
force_action: true
```

This is why the drawer shape above is achievable — the old fat KF4 blob is genuinely replaced, not merged with the new lean mount fields.

---

## Migrated components (2026.7.0)

`trapper_keeper`, `zen_system`, `camera_manager`, `room_manager`, `security_manager` (+ `security_perimeter`, `security_panel`, `security_cameras`), `alert_manager`, `autovac`, `finance_budget`, `finance_bills`, `finance_networth`, `finance_manager`, `battery_health`, `logistics_intake`, `logistics_volatile`, `dishwasher_manager`, `hot_tub_manager`, `trash_trakker`, `laundry_manager`, `energy_manager`, `water_manager`, `garage_freezer_thermal_model`.

Not yet migrated: `taskmaster` (large instruction set — pending a dedicated pass).
