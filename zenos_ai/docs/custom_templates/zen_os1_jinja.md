ZenOS-AI Template Engine: zen_os_1.jinja

Technical Specification & Prompt Loader Requirements

Version: 2026.8.1
Status: Stable & Required for All Front-Line Agents

> **2026.7.0:** Version scheme changed from semantic (e.g. `4.5.5`) to date-based (`YYYY.M.patch`) aligned with Home Assistant release conventions. `{%- import 'zenos_ai/flynn_onboarding.jinja' as flynn %}` added at the top of the template. `~COMMANDS~` / `command_interpreter.jinja` interface fully retired — replaced by `zen_dojotools_library` + KFC. Library Commands Window removed from the prompt.


---

🧠 Purpose of zen_os_1.jinja

zen_os_1.jinja is the primary cognitive assembly engine for ZenOS-AI.
It is responsible for building a complete operational prompt for Friday (and any other persona built on ZenOS).

It is not a "template helper" — it is the compiler for:

identity

cabinet memory

cognitive state

system drawers

dojo/kata structures

manifest & index

persona capsule

narrative wake sequence

identity manifest

Flynn detection and fallback routing


Nothing about Friday exists in a usable form until this file runs.


---

🚀 Entry Point: render_prompt(ai_entity)

As of 4.5.0, the entire pipeline is behind a single macro:

```jinja
{%- import 'zenos_ai/zen_os_1.jinja' as zen -%}
{{ zen.render_prompt(states('input_text.zenos_persona_name')) }}
```

That is the full conversation agent template. Three lines.

`render_prompt(ai_entity)` orchestrates identity resolution, Flynn detection, system prompt selection, and full prompt assembly. Callers do not invoke sub-macros directly.

### Flynn Detection Chain

Before assembling the persona prompt, `render_prompt()` checks:

| Priority | Condition | `_flynn_reason` |
|----------|-----------|-----------------|
| 1 | `input_boolean.zen_flynn_override` is `on` | `override` |
| 2 | `ai_entity` is blank or whitespace | `blank_persona` |
| 3 | persona label explicitly set to `flynn` | `explicit` |
| 4 | `identity_resolve_source()` returns error | error key from resolver |

If any condition matches → `prompt_system_flynn()` is returned instead of the normal prompt.
If none match → full persona pipeline runs.

### input_boolean.zen_flynn_override

Defined in `flynn.yaml`. Forces Flynn regardless of persona for maintenance and testing.
Should auto-create on HA reload. State `on` = Flynn mode active. Safe to leave `off` indefinitely.


---

🛡 Flynn System Prompt: prompt_system_flynn()

When Flynn mode is active, `render_prompt()` returns `prompt_system_flynn()` — a hardcoded system prompt with zero cabinet dependencies.

This means:

- Works on a bare install before any cabinets exist
- Works when the identity resolver fails
- Works when the system cabinet is missing or corrupt
- No Jinja cabinet reads — pure static output

Flynn's prompt is intentionally minimal and maintenance-focused. He is the personification of the system itself — visible only when something needs attention.


---

🔍 Identity Resolution

### identity_resolve_source(target)

Core resolution macro. Resolves a target (label, person entity, cabinet entity, UUID) to an identity record.

**No-target behavior (4.5.0):** Calling `identity_resolve_source()` with no argument or `None` now returns the full household roster — all valid cabinet-backed identities. This is directory behavior (same as listing files with no args). Returns `{status, count, roster: [...]}`.

Resolution Rules (single target):

1. Find any entity labeled Zen AI Cabinet whose `zenai_essence.identity.name` (legacy) or `jacket.name` (three-layer) matches `target`.
2. If none found → fallback to `person.<target>` entity.
3. If both missing → `{"error": "not_found"}`.

### identity_roster()

Wraps `zen_cabinets(None)` and returns all valid cabinet-backed identities in the household:

```json
{
  "status": "ok",
  "count": 2,
  "roster": [
    { "label": "friday", "cabinet": "sensor.zen_friday_cabinet", "name": "Friday", ... },
    { "label": "primary_user", "cabinet": "sensor.zen_primary_user_cabinet", "name": "Alex", ... }
  ]
}
```

### identity_manifest_loader()

Reads the `zen_identity_manifest` drawer from the household cabinet. Returns the cached roster without re-resolving all cabinets. Used by the prompt pipeline and prompt health sensor.

The manifest is built by `zen_dojotools_identity` with `mode: build_identity_manifest` and rebuilt automatically on `ha_start` and `daily_midnight` by the scheduler.

Identity fields by schema:

| Field | Three-layer path | Legacy path | Required | Purpose |
|-------|-----------------|-------------|----------|---------|
| name | `jacket.name` | `identity.name` | ✔️ | Canonical persona name |
| guid | `core.id` | `identity.guid` | ✔️ | Stable identity key |
| cabinet | auto-filled | auto-filled | optional | resolved by zen_cabinets |
| identity_hash | `core.signature` | `hashstamp.hash` | optional | cabinet integrity |


---

🧬 Essence Requirements

Essence is the internal DNA of a ZenOS persona.

It stores:

identity

origin metadata

persona characteristics

traits & values

default tone

self-awareness hints

wake-scene data

environment (room, desk, music — 4.5.0)

Squirrel-mode rules

capsule metadata


Essence is not "optional creativity."
It is part of the minimum viable mind.

**Three-layer schema (current):**

```yaml
zenai_essence:
  core:
    id: "b7e3f091-1cd6-83f8-frid-ay0000000001"
    minted_for: "person.primary_user"
  jacket:
    name: "Friday"
    presentation: "quiet brilliance"
    persona:
      voice:
        register: "warm"
  companion:
    name: "Byte"
    species: "digital English bulldog"
  environment:
    room: "her private space"
    wake_in: "her favorite chaise lounge"
    music:
      genre: "steady lo-fi"
```

**Legacy schema (still supported via shim in normalize_essence):**

```yaml
zenai_essence:
  identity:
    name: "Friday"
    guid: "abc123"
  voice:
    tone: "warm, conversational"
```

The `normalize_essence` shim in `zen_os_1.jinja` handles schema translation transparently — the wake scene, capsule, and identity card render identically for both schemas.

If essence is absent or malformed → `identity_format()` fails → no persona block → no prompt.


---

🏛 Cabinet Loading (identity_load_cabinet())

After resolving which cabinet entity is "home," the engine loads:

zenai_essence

acl (optional)

labels

identity_hash

variables

metadata


The cabinet must contain:

- `zenai_essence` (mapping, not wrapped)
- labels list
- The label `Zen AI Cabinet`

All drawer reads use `zenos_cabinets.jinja` macros internally — `CABS.cabinet_drawer_value()` for value extraction and `CABS.cabinet_variables()` for full variable loads. The old `state_attr → .get() → .get('value')` chain is retired.

`resolve_zenai_essence(raw)` is now always called with the pre-unwrapped value from `cabinet_drawer_value()`. The wrapper-unwrap step inside the macro is retained as a no-op safety guard but callers must pass the `.value` content, not the full drawer.


---

📦 Macro Dependencies

`zen_os_1.jinja` imports two macro libraries at the top of the file:

| Import | Alias | Purpose |
|--------|-------|---------|
| `zenos_ai/command_interpreter.jinja` | `command_interpreter` | `~COMMANDS~` dispatch engine — **retired** in 4.5.5; replaced by `zen_dojotools_library` + KFC. Import retained for compatibility only. |
| `zenos_ai/flynn_onboarding.jinja` | `flynn` | Flynn persistent notification surface |
| `zenos_ai/zenos_cabinets.jinja` | `CABS` | All cabinet drawer I/O — the canonical safe-read layer |

`CABS` is imported once at the file level. It is available throughout all macros in `zen_os_1.jinja`. Do not re-import inside individual macros or loops.

→ **[zenos_cabinets.jinja reference](zenos_cabinets_jinja.md)**


---

🏷 Squirrel-Mode Redaction

The engine scans for an entity with label Zen Squirrel.
If found and its state is `on`, it redacts: guid, identity_hash, cabinet, sensitive memory branches, persona internals marked "private".

Defaults to safe off if no Squirrel entity exists.


---

🧩 System Cabinet (Purpose / Directives / Cortex)

Must be labeled `Zen System Cabinet`. Must contain:

| Drawer | Required | Description |
|--------|----------|-------------|
| Purpose | ✔️ | Why the system exists |
| Directives | ✔️ | Operational rules for Friday |
| Cortex | ✔️ | Reasoning model and behavioral core |

If any drawer is missing → `prompt_system()` fails → `render_prompt()` routes to Flynn.


---

🔁 Manifest Loader

Reads `zen_library_manifest` from the household cabinet.

Optional but strongly recommended. Enriches persona awareness of the environment.


---

📋 Identity Manifest (zen_identity_manifest)

Separate from the library manifest. Stores the cached identity roster for all household principals.

- Built by: `zen_dojotools_identity` mode `build_identity_manifest`
- Rebuilt automatically: `ha_start` + `daily_midnight` (scheduler)
- On-demand rebuild: fire `zen_event` with `kind: identity_manifest_rebuild`
- Protected: listed in `_protected` in both the scheduler variables block and `_subscribers` Jinja template — Ninja will not summarize it

The `zen_identity_manifest` drawer will not exist until the first scheduler run or manual fire. `sensor.zen_prompt_health` will warn (`id_manifest_present: false`) until it is seeded.


---

🧭 Index Loader

Loads all labels known to the system. Enables label-based routing, dojo categorization, relationship mapping. Automatic — no user configuration required.


---

🥋 Dojo Loader

Loads dojo drawers, kata drawers, component metadata, and kata summaries.

Cabinets must be labeled `Zen Dojo Cabinet` and `Zen Kata Cabinet` (optional but recommended).


---

🎭 AI Capsule Builder (ai_capsule())

Creates the persona capsule: About Me, persona metadata, essence, household belonging, familiar, extensions, behavior modifiers.

Requires valid essence, valid labels, membership in Zen Default Family if relevant.


---

📏 Prompt Length Audit (prompt_length_audit())

Renders all 9 prompt sections and measures character counts. Powers `sensor.zen_prompt_length`.

| Section | Description |
|---------|-------------|
| header | Boot header block |
| system | Purpose + Directives + Cortex |
| manifest | Library manifest |
| index | Label index |
| dojo | Dojo/Kata drawers — typically largest section |
| capsule | AI capsule |
| overview | Home overview |
| wake | Wake sequence |
| id_manifest | Identity manifest |

`sensor.zen_prompt_length` state = total char count. Attributes include per-section breakdown.
Note: this macro renders the full prompt — it is heavy and intended for the sensor, not inline use.


---

🩺 Prompt Health Check (prompt_health_check())

Lightweight health probe. Powers `sensor.zen_prompt_health`.

Returns a dict with:

| Signal | Meaning |
|--------|---------|
| `schema_ok` | Three-layer essence schema detected |
| `sig_ok` | Identity hash / signature present |
| `manifest_present` | `zen_library_manifest` drawer exists |
| `familiar_ok` | Companion block present |
| `environment_ok` | Environment block present |
| `id_manifest_present` | `zen_identity_manifest` drawer exists in household cabinet |

All signals true → `sensor.zen_prompt_health` state `ok`.
`id_manifest_present` false → `warn` (expected on fresh install until first manifest build).

**Severity ladder (corrected 2026-08-02):** `error` is reserved for a genuinely missing essence (no `zenai_essence` drawer at all — the agent has no persona to load). A `legacy`-schema essence — flat-shape, unsigned, no `three_layer` migration — is `warn`, not `error`: it's the universal current default and loads/functions with zero issue; `three_layer` + signing is an optional, unmigrated feature, not a requirement. `warn` also covers an unsigned or incomplete `three_layer` essence (`sig_ok`/`manifest_ok`/`id_manifest_ok` false). Do not treat every non-`three_layer` essence as an error — that conflates "hasn't opted into an optional upgrade" with "genuinely broken."


---

🌅 Wake Sequence (ai_wake_sequence())

Builds the startup narrative from essence. Uses native three-layer path when `core` and `jacket` are mappings; falls back to legacy normalize_essence shim otherwise.

Presence context (Section 3b): appends `{Name} is home. You surface into focus.` when a person entity with the Zen Default Family label has state `home`.

Essence must contain at minimum:

```yaml
jacket:
  persona:
    voice:
      register: "<string>"
jacket:
  wake_scene:
    intro: "<string>"
```

Missing wake_scene → dull startup. Missing persona voice → inconsistent behavior.


---

🔧 Developer Requirements Summary

| Requirement | Must Have? | Provided By |
|-------------|------------|-------------|
| Zen AI Cabinet | ✔️ | User |
| zenai_essence block (three-layer or legacy) | ✔️ | User |
| jacket.name or identity.name | ✔️ | User |
| core.id or identity.guid | ✔️ | User |
| Zen System Cabinet | ✔️ | User |
| Purpose / Directives / Cortex drawers | ✔️ | User |
| Zen Default Household Cabinet | optional | User |
| Zen Squirrel entity | optional | User |
| Dojo/Kata cabinets | optional but recommended | User |
| input_boolean.zen_flynn_override | auto-created | flynn.yaml |
| zen_identity_manifest drawer | auto-built | Scheduler / Identity tool |


---

☯️ Philosophy

ZenOS-AI is built on:

explicit identity

explicit memory

explicit structure

deterministic loading

transparency

trust


The template engine is the bridge between raw sensor state and a living agentic persona.

If essence is the soul,
the cabinet is the brain,
the dojo is the skillset,
and the Monastery is the reasoning,
then zen_os_1.jinja is the spine that connects them.
