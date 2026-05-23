# Zen DojoTools Scribe — v1.8.0

*Guided authoring and lifecycle management for KF4 artifacts — MCP-exposed*

---

## What Scribe Does

Scribe is the authoring tool for the KF4 ecosystem. It manages the full lifecycle of
**thoughts**, **scrolls**, and **KFCs** — from blank draft to published, running component.

It is fully MCP-exposed. Friday can use it directly. A capable LLM with access to Scribe
can design, scope, draft, refine, formalize, and publish a working KFC component without any
operator intervention beyond initial label design.

---

## The Artifact Lifecycle

Scribe treats artifacts as living documents with three formal states:

```
thought (draft)  →  formalize_scroll  →  scroll (formal, RO)  →  publish_kfc  →  KFC (live in Dojo)
```

| State | Class | What It Is |
|---|---|---|
| `draft` | `thought` | Mutable working document. Patch freely. |
| `formal` | `scroll` | Locked for review. Read-only. Can be promoted. |
| Published | `kfc` | Live component definition in the Dojo. Scheduler picks it up. |

**Drawer name = primary index key.** The artifact class is semantics — not a key prefix.
A `thought` named `water_manager` and a `kfc` named `water_manager` are the same drawer
in different states.

---

## Modes

| Mode | What It Does |
|---|---|
| `help` | Return full doctrine, contract, and examples. Start here. |
| `read` | Read an existing thought, scroll, or KFC. |
| `new_thought` | Scaffold a new mutable draft — auto-creates the label, runs Index to capture scope, writes the full component payload. |
| `patch` | Update only the fields you specify. Existing content is preserved. |
| `replace` | Replace the full payload explicitly. Requires `replace_confirm: true`. |
| `clear_field` | Remove named fields from the payload. Requires explicit field list. |
| `delete` | Delete the drawer entirely. Requires `delete_confirm: true`. |
| `formalize_scroll` | Lock the artifact as formal and read-only (`artifact_state: formal`). Applies `zen_scroll` label — hard RO via FileCabinet. |
| `publish_kfc` | Press a formal scroll to the Dojo as a live KFC. Requires `artifact_state == formal` and `publish_confirm: true`. |
| `republish_kfc` | Edit-and-republish an existing KFC without going through the formal scroll gate. Reads the current KFC from the Dojo, merges any supplied field changes, runs the full trim block (label create-or-update, `icon` sync, chonk warning). Use after `patch` when label sync is needed, or standalone to sync only the label description. |
| `dry_run` | Preview scope discovery and authoring guidance without writing anything. |
| `list_triggers` | Return all available scheduler trigger IDs. |
| `enable` | Shorthand — patch `meta.enabled: true` without a full rewrite. |
| `disable` | Shorthand — patch `meta.enabled: false` without a full rewrite. |
| `repair` | Detect and flatten wrapper-accumulated drawer corruption. `dry_run: true` by default — set `false` to write. |

**Default:** No input → `mode: help`.

---

## The Scope Model

Every KFC component has two identity concepts that often differ:

| Concept | Field | Purpose |
|---|---|---|
| **Corpus identity** | `label` / `primary_index_key` | The durable label and drawer key that defines what this artifact *is*. |
| **Index scope** | `label_1` + `label_2` + `operator` | The entity set the component reasons over when it runs. |

**Simple case:** Set `label` only. Scribe uses it as both corpus identity and index scope.

**Advanced case:** Set `label_1` + `label_2` + `operator` to bound a more precise entity set — cross-label AND/OR/NOT/XOR queries, or `*` for hypergraph traversal.

Scribe calls `zen_dojotools_index` automatically during `new_thought` to capture a live scope snapshot. The result becomes part of the thought payload — the LLM knows what the component will see before it writes a single line of `component_summary`.

---

## Component Group Model

One corpus may host a parent context plus multiple component views — all sharing a clock,
trigger set, and labelset:

| Field | What It Does |
|---|---|
| `primary_index_key` | Shared corpus identity — the parent thought drawer key |
| `component_key` | Published component view key (what goes into the Dojo) |
| `parent_component_key` | Links subcomponents to a parent |
| `shared_schedule_group` | One clock, one group name |
| `shared_trigger_subscriptions` | One trigger list for parent and all subcomponents |
| `shared_delay_seconds` | One stagger delay for the whole group |
| `shared_labelset` | One label set applied across the group |

Use this when one domain (e.g., `water_manager`) needs multiple summarizer views
(`water_manager_flow`, `water_manager_pressure`) under one shared schedule.

---

## Key Fields Reference

### Core Identity

| Field | Purpose |
|---|---|
| `key` | Prime index key / drawer name — the artifact's canonical identifier |
| `friendly_name` | Human-readable title |
| `artifact_class` | `thought`, `scroll`, or `kfc` |
| `cabinet_entity_id` | Target cabinet (defaults to Dojo) |
| `label` | Base label — corpus identity; auto-created if `auto_create_label: true` |
| `version` | Semantic version string |

### Scheduling

| Field | Purpose |
|---|---|
| `trigger_subscriptions` | Comma-separated scheduler trigger IDs |
| `delay_seconds` | Summarizer stagger delay (0–300s) |
| `custom_trigger_override` | Explicit trigger override for this run |
| `drift_threshold` | Urgency score at or above this value triggers forced supersummary. Default 5. |
| `emission_cooldown_minutes` | Minimum minutes between urgency/action emissions. Default 60. |
| `suggested_act_event` | Static suggested_act_event slug for this KFC — must be a registered dojotool slug or blank. |
| `master_switch` | `input_boolean` entity ID that gates this component (optional). Preserved for backward compat. |

### Component Content

| Field | Purpose |
|---|---|
| `component_summary` | One-line operational description (what Friday sees) |
| `icon` | MDI icon for the KFC label (e.g. `mdi:water`). Set on first publish via label create; preserved on subsequent updates. |
| `component_instructions` | Monk guidance for the Ninja Summarizer |
| `more_info` | Extra rationale, references, design notes |
| `what_it_does` | Plain-English description of the collection's purpose |
| `what_good_looks_like` | Nominal behavior guide |
| `what_bad_looks_like` | Non-nominal behavior guide |
| `urgency_rules` | What elevates urgency in this component |

### Policy Overlays

| Field | Purpose |
|---|---|
| `home_policy` | Home-level behavioral overlay text |
| `target_household` | Household cabinet entity or GUID — scopes `home_policy` to a specific household |
| `family_customizations` | Family-level overlay text |
| `target_family` | Family cabinet entity or GUID — scopes `family_customizations` to a specific family |
| `user_customizations` | User-level overlay text |
| `target_person` | `person.*` entity — scopes `user_customizations` to a specific person. Resolved via identity (covers user and ai_user cabinet types). |

**Targeting:** Each overlay field has an optional target. If no target is supplied the overlay is anonymous — the monk applies it as a general policy without identity context. When a target is supplied, the resolved GUID, display name, and cabinet type are embedded in `policy_overlays` alongside the overlay text. The monk knows exactly who the customization applies to.

`target_person` is a `person.*` entity — never a cabinet entity. The identity resolver determines whether the person backs a `user` or `ai_user` cabinet.

### Scope

| Field | Purpose |
|---|---|
| `label_1` / `label_2` | Index scope labels |
| `operator` | `AND`, `OR`, `NOT`, `XOR`, `*` |
| `filter_json` | ZQ-1 post-filter |
| `expand_scope` | Pass scope through Inspect for entity-level detail |
| `output_fields` | Inspect field shaping when `expand_scope: true` |
| `domain_inspect_domains` | Comma-separated HA domains to restrict step-4b domain inspect (e.g. `"todo,calendar"`). Empty = all entities from index result (default). |
| `domain_inspect_limit` | Max entity count for step-4b domain inspect. Default 25. |

### Seed (v1.4.0+)

`seed` and `area_seed` let a KFC define its context source as a direct tool call instead of a label-based index query.

| Field | Purpose |
|---|---|
| `seed` | JSON string. Concept-first tool call: `{"tool": "zen_dojotools_room_manager", "params": {...}}`. When present, Ninja Summarizer calls this tool instead of running HyperIndex. |
| `area_seed` | JSON string. Location-first variant. `{{area_id}}` slot in params is filled at runtime when `area_id` is passed to the Ninja Summarizer. Used for per-area rollup patterns. |

When publishing a KFC, `seed` and `area_seed` are preserved in the Dojo drawer. When inputs are blank on patch/publish, existing seed values are left untouched.

**Choosing between index_call and seed:**
- `label_1`/`label_2`/`index_call` — concept-first, entity set is the context. Right for most components.
- `seed` — tool-first, tool output is the context. Right when Room Manager, Plant, or a domain tool returns richer context than a label query.
- `area_seed` — like `seed` but spatial. One KFC definition fires per-room with `{{area_id}}` injected. Right for room-scoped ambient awareness.

### Safety

| Field | Default | Purpose |
|---|---|---|
| `prevent_self_indexing` | `true` | Exclude this artifact's drawer from scope rediscovery |
| `suppress_drawer_feedback` | `true` | Use drawer blurbs as advisory only — prevent feedback loops |
| `auto_create_label` | `true` | Create the base label automatically if missing |

### Audit

| Field | Purpose |
|---|---|
| `caller_id` | Durable audit identity — persisted on every write |
| `caller_token` | Governance token — echoed on every response |

---

## Authoring Flow (LLM-Driven)

This is where Scribe gets interesting. A capable LLM with MCP access can run the full
component authoring loop conversationally:

**Step 1 — Understand available triggers**
```yaml
mode: list_triggers
```
Returns all scheduler trigger IDs. The LLM picks the right ones for the domain.

**Step 2 — Preview scope (optional but recommended)**
```yaml
mode: dry_run
artifact_class: thought
key: water_manager
label: Water Manager
label_1: Water Manager
label_2: Critical Spaces
operator: AND
```
Returns scope guidance and a live index snapshot without writing anything.

**Step 3 — Draft the component**
```yaml
mode: new_thought
artifact_class: thought
key: water_manager
friendly_name: Water Manager
label: Water Manager
label_1: Water Manager
label_2: Critical Spaces
operator: AND
component_summary: "Monitors all water flow sensors and valves. Alerts on active flow during Away mode or at unusual hours."
component_instructions: "Focus on: (1) is there active flow right now? (2) is flow expected given home mode and time of day? (3) any recent anomalies?"
what_good_looks_like: "Flow only during expected times. No continuous drips. Valves closed when away."
what_bad_looks_like: "Flow active in Away mode. Sustained flow >30 minutes. Unexpected middle-of-night readings."
urgency_rules: "Elevate to high if flow detected in Away mode. Elevate to critical if flow sustained >2 hours."
trigger_subscriptions: "water_flow_stop,home_mode_updates,hourly_trigger"
delay_seconds: 30
```

**Step 4 — Refine iteratively**
```yaml
mode: patch
artifact_class: thought
key: water_manager
changes:
  delay_seconds: 60
  more_info: "CPS alert threshold is 3 gallons/hour continuous."
```

**Step 5 — Lock for review**
```yaml
mode: formalize_scroll
artifact_class: scroll
key: water_manager
```
Artifact state moves to `formal`. `zen_scroll` label applied — FileCabinet hard RO from this point.

**Step 6 — Publish to Dojo**
```yaml
mode: publish_kfc
artifact_class: kfc
key: water_manager
publish_confirm: true
```
KFC pressed to the Dojo. Scheduler picks it up on the next cycle. Component is live.

---

## Where Artifacts Live

| Artifact | Cabinet | Why |
|---|---|---|
| `thought` | Any — defaults to Dojo | Working draft; often starts in Dojo for LLM visibility |
| `scroll` | Any | Formal but not yet published |
| `kfc` | **Dojo** | Must be in Dojo — Monastery and Scheduler read from there |

Thoughts and scrolls can live in any cabinet. KFCs must be pressed to the Dojo.
See [Cabinet Placement Guide](../getting_started/cabinet_placement.md) for the full placement model.

---

## Safety Model

Scribe is non-destructive by default:

- **`patch`** — only changes what you specify. Everything else is preserved.
- **`replace`** requires `replace_confirm: true`
- **`delete`** requires `delete_confirm: true`
- **`publish_kfc`** requires `artifact_state == formal` and `publish_confirm: true` — cannot publish a draft
- **Self-indexing guard** — prevents a component from including its own drawer in scope discovery
- **Drawer feedback loop guard** — drawer blurbs are advisory; they don't re-enter authoring context

---

## Integration

| Dependency | Purpose |
|---|---|
| `script.zen_dojotools_filecabinet` | All drawer reads and writes |
| `script.zen_dojotools_labels` | Label create / auto-create |
| `script.zen_dojotools_index` | Scope discovery on `new_thought` and `dry_run` |
| `sensor.zen_dojo_cabinet_resolved` | Default target cabinet |
| `kfc_template` drawer in Dojo | KF4 contract schema — validated on `publish_kfc` |

---

## Notes

- `zen_dojotools_kungfu_writer` has been retired. Scribe is the replacement — richer schema, full lifecycle management, and LLM-native authoring flow.
- **Scribe is fully MCP-exposed.** Friday can use it as a first-class tool.
- For admin-only cabinet repair and system-level operations, see [AdminTools](zen_dojotools_admintools_readme.md). AdminTools is not MCP-exposed.

## Version History

| Version | Change |
|---------|--------|
| v1.8.0 | `republish_kfc` mode — edit-and-republish path that bypasses the formal scroll gate; merges changes and runs full trim block including label sync. |
| v1.7.0 | `icon` field added to KFC schema; set on first publish (label create), preserved on update. `publish_kfc` label upsert — creates label if absent, always updates description. Component summary chonk warning (>150 chars). |
| v1.4.0 | `seed` and `area_seed` input fields; parsed into draft and publish payloads. |
| v1.3.0 | Initial release as Scribe. Replaces `zen_dojotools_kungfu_writer`. Full lifecycle management, LLM-native authoring, component group model. |
