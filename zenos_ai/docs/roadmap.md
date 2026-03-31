# ZenOS-AI Roadmap

*Local-first cognitive architecture for persona-scale AI on Home Assistant*

ZenOS-AI is a cabinet-centric AI framework for deterministic, inspectable household cognition.

---

## Current Status

> **Versioning:** Public ZenOS releases follow Home Assistant's `YYYY.M.patch` convention. Each new month starts at `.0`. Internal architecture versioning (`4.5.x` series) is retained in commit history and internal tooling.

**2026.3.1 — Shipped (2026-03-27)**

Identity graph, alertmanager, and GUID correctness batch. `unlink_partners` variable name fix (ACL
removal now persists). Cold bootstrap wires default family into household graph. Identity tree
name resolution extended (friendly_name → name → entity_id). GUID id-preference fix (23 sites —
`id` over legacy `guid`). Household root profile-first name resolution. Home mode `input_datetime`
helpers ship without `initial:`; Flynn seeds defaults on first boot. AlertManager deduplication
fixed. `run_repair` moved from SystemTools to AdminTools behind confirm gate; two new maint scripts
(`stamp_cab_guid`, `roster_guid_repair`) for pre-provisioner installs. Family cabinet VolumeInfo
rogue key repair script added (`maint/repair_fam_cab_volumeinfo.yaml`). User cabinet essence
seeding gap fixed — provisioner now seeds `zenai_essence` for `user` type at provision time;
repair script added for existing installs (`maint/repair_user_cab_essence.yaml`).

See: [2026.3.1 Patch Notes](releases/patch_4_5_6.md)

---

**2026.3.0 'Ready Player Two' — Shipped (2026-03-26)**

Identity and lifecycle release. Cabinet provisioning system (mount-aware state machine,
RP2 provisioner, expansion slots, scroll ceremony). Warmup as first-class sensor state.
Identity model v4.5.0 — full household/family group management, 12 new modes, depth-2
security resolution, enriched manifest. Profile editor GA-hardened. Cortex 32 (True Voice)
now the default. UAT passed 21/21 on live install 2026-03-26.

See: [Release Notes — Ready Player Two](releases/ready_player_two.md)

---

**Previous: Meridian — Shipped (2026-03-21)**

Hardening and governance batch. `caller_token` SP1 §4a stub wired to all 15 AI-accessible
DojoTools scripts. Cortex 31 cabinet state visibility. KFC schema 1.2.0. Labels UPDATE
patch semantics (pre-read, confirm gate, no data-loss path). Scheduler, summarizer, and
history bug fixes.

See: [Release Notes — Meridian](releases/meridian.md)

---

**Previous: 4.1.0 RC2 — Complete**

KF4 RC2 has shipped. The Kung Fu Component architecture is fully Dojo-driven — drawer IS the spec, label IS the scope, HyperIndex IS the data layer. The Scheduler is data-driven; no hardcoded branches. 12 components stamped and running. UAT passed on live install 2026-03-11. The next milestone is GA.

Pulled forward from GA deliverable §7 (Baseline KFC Set). The 4.0.0 Scheduler was not freely editable — each new component required a hardcoded choose branch, creating a shipping conflict on every update. KF4's Dojo-driven dispatch made that constraint moot entirely, so we elected to pull it forward rather than carry the architectural debt to GA.

See: [#56 feat: Kung Fu Components KF4 RC2](https://github.com/nathan-curtis/zenos-ai/issues/56)

---

# 1.0 RC1 — Achieved

`RC1` established the architecture and proved the substrate.

### Delivered

- Ring-0 canonical package structure under `packages/zenos_ai/`
- Cabinet model with health sensors and structural validation
- Label-driven architecture for cabinet resolution, indexing, and tool behavior
- FileCabinet v1 — typed drawer operations
- Volume redirector pattern
- Monastery and Kata memory loop
- Flynn bootstrap and recovery concept
- DojoTools v1 operational baseline

---

# 1.0 RC2 — Achieved

`RC2` was the deployment, bootstrap, consolidation, and lifecycle milestone.

### Delivered

#### Package Consolidation

All active ZenOS runtime logic is package-owned under `packages/zenos_ai/`. No root monoliths. No `/scripts/` or `/automations/` directories. The package tree is the authoritative deployment artifact.

#### Version Rationalization

All files rationalized to `4.0.0 RC2`. Eliminated bogus version strings (`1.5.x`, `3.9.5+`), mismatched internal versions, stale inline versions, missing headers, and compound version strings. Every file in the repo now reports the same coherent baseline.

#### FileCabinet v4

Drawer metadata extended with five new fields:

- `description` — human-readable purpose; surfaces as `_hint` on direct read
- `expires_after` — ISO-8601 timestamp; time-bombs a drawer
- `no_autoexpire` — opt out of GC expiry path
- `no_autorecycle` — opt out of recycle path
- `acl` — access control mapping

Label-mode reads return blurbs only (description or truncated value), not full drawer content. Token-efficient scan view for AI consumption.

Direct reads: described drawers return full content; undescribed drawers return value truncated to 64 chars. Architectural incentive for good cabinet hygiene.

#### Drawer Lifecycle

Unix-style visibility model:

- `foo` — visible, active
- `.foo` — hidden, archived
- `_foo` — system-protected, never touched

#### FileCabinet GC

Renamed from `kata_gc` to `filecabinet_gc`. Upgraded from single-cabinet stamp-based eviction to:

- Full hide/delete/recycle/unhide state machine
- Hits all registered `AI Data Storage Cabinet` entities in series
- Resolves cabinet list from `zenos_ai/library_index.jinja` at runtime
- Single `volume_entity_id` override to target one cabinet
- `AI_Cabinet_VolumeInfo` always protected in all paths
- System drawers (`_` prefix) always protected
- Protection guards on all manual operations (recycle, hide, unhide)
- Runs every 15 minutes via scheduler
- Manual trigger via `zen_event` / `gc_force`

#### Scheduler

- FileCabinet GC on 15-minute pattern
- `force_gc` event trigger added

#### Flynn

Updated to v4.0.0 RC2. Remains the infrastructure-layer bootstrap and failback agent.

---

# 1.0 GA — Current Target

## Theme

`GA` is the hardening, operator experience, and lifecycle polish milestone.

Where RC2 proves deployability, GA proves durability, clean boundaries, and maintainability at scale.

---

## GA Stability Gates — Delivered

### Prompt Finalization — Final GA Feature (planned)

Final polish pass before gold candidate lock. Three work streams:

**1. Template structure pass + rename**

`zen_os_1rc.jinja` renamed to `zen_os_1.jinja` for release. Full ref sweep across
all packages, templates, and docs completed. `custom_templates/zenos_ai/` layout finalized
and locked.

**2. Prompt instrumentation + collapse**

Each major prompt section (cortex, home overview, active components, zen_summary,
etc.) gets a dedicated sensor — observable at runtime, traceable by Nyx, and
auditable by Flynn. Once all sections are instrumented and validated individually,
the full prompt collapses behind a single `zen.prompt('ai_user')` call. Persona-aware,
data-layer driven, no hardcoded structure in the surface API.

The "exploded" dev-era template layout disappears from the surface. The structure
is still there — it's just behind a clean interface.

**3. Flynn prompt shunt**

Pre-boot and degraded-state path: agent receives Flynn-only context (who Flynn is,
what tools he has) instead of an incomplete or errored full prompt. Shunt condition
derived from existing health sensors — resolver unsettled, essence missing, Gate 3
incomplete. No full prompt rendered until the system is ready to render it correctly.

**Stuffiness gauge:** token pressure per prompt section surfaced as a first-class
observable. Flynn can see which sections are consuming context budget, flag pressure
before it degrades agent quality, and recommend or trigger remediation (trim, collapse,
raise abstraction level). Context stuffiness goes from invisible to instrumented.

**Sequence:** instrument first → Nyx validates each sensor → collapse behind
`zen.prompt()` → Flynn shunt wired last.

---

### Highlander Resolver Architecture (2026-03-19)

Boss Add. Support team demand. Shipped in `fix/ga-batch-1`, UAT-passed on live install.

Cabinet entity ID resolution is now deterministic system-wide. There can be only one source of
truth per cabinet — resolver sensors (`sensor.zen_*_cabinet_resolved`). All OS code reads cabinet
entity IDs from these sensors exclusively. No `label_entities()` for single-cabinet resolution
anywhere in OS code.

**What shipped:**
- 7 always-live resolver sensors (dojo, kata, system, household, ai_user, family, user)
- Full resolver canonization sweep across all OS files (dojotools, sensors, flynn, templates)
- `zen_health_report` v4.3.0 — all 7 resolver states, 6 health sensors, FG-38-safe diagnosis
- `zen_resolver_refresh` — exposed MCP tool + ha_start cold-start retrigger
- FG-38 v3 (double-encoded JSON) — three-round normalization pattern, patched in all affected paths
- HALMark FG-38 v3 candidate filed for Vera's ratification review

---

## GA Deliverables

### 1. Clean Bootstrap from Scratch

Prove the full path from zero:

`fresh HA install → packages drop-in → Flynn init → labels → cabinets → default household/family/person/AI → agent live`

Must be script-reproducible without manual intervention after the initial package drop.

### 2. Legacy Memory Import

Controlled import of prior-install state into canonical cabinet slots. Every importable source classified as:

- direct import
- transform on import
- regenerate
- discard

Import order: identity facts → durable memory → essential context → optional historical material.

Post-import regeneration of manifests, indexes, summaries, health state, and prompt-ready context.

### 3. Cabinet Loader v1 GA

Hardened, deterministic cabinet loader with:

- Slot enforcement
- Clear resolution boundaries
- Diagnostics on mismatch or missing slots

### 4. Governance Model v1

Declarative operational boundaries for what agents can read, write, and do. Per-cabinet and per-drawer ACL enforcement wired to the `acl` meta field introduced in RC2.

### 5. Persona Importer v1

Fully defined persona lifecycle loader covering:

- identity
- proprioception
- governance
- kata
- essence
- prompt structure

Clean import path from persona export to live canonical cabinet state.

### 6. Flynn Finalization

Flynn becomes the official first-run and failback system agent with:

- Stronger health shunts and repair triggers
- Deterministic onboarding flow from clean state
- Better recovery tooling surface area

### 7. Baseline KFC Set

GA guarantees a stable, documented baseline set of label-driven Kung Fu Components required for a compliant deployment. Every required KFC is loadable from the package tree without external dependencies.

### 8. HyperIndex Consolidation

HyperIndex capabilities folded into the canonical DojoTools Index path. No parallel RC evaluation tracks exposed as architecture requirements.

### 9. Verification Gates

Each deployment batch has a repeatable test gate:

- config loads cleanly
- labels validate
- cabinets validate
- FileCabinet read/write round-trips correctly
- manifest works
- identity resolves
- index queries correctly
- agent answers baseline prompts from clean state
- import lands in correct slots
- summarizer path remains healthy

---

## GA Exit Criteria

GA is complete when:

- core package tree is the sole authoritative source for all ZenOS behavior
- canonical labels can be installed after a deliberate reset
- Ring-0 cabinets validate cleanly from fresh state
- default household, family, person, and AI initialize deterministically
- a default agent starts from clean canonical state and answers coherently
- historical state imports into correct canonical slots
- derived state regenerates successfully
- the full path is script-reproducible on a fresh or resettable install
- ACL enforcement is wired and tested
- Flynn onboarding path is documented and verified

---

# Known Limitations

These are documented gaps that are intentional non-fixes for the current release. Each is tracked for a future milestone.

---

### Conversation Agent Liveness Check (post-MVP)

**Scope:** `flynn_bootstrap_content` Gate 3

Flynn validates that the configured conversation agent entity exists and is not in an `unavailable` or `unknown` state. It does **not** perform an actual inference round-trip to confirm the model responds.

A misconfigured, offline, or rate-limited model will pass the boot gate silently. The failure will surface at runtime when a summarizer or OOBE call hits the `ai_task.generate_data` step — the script will error cleanly but the root cause may not be obvious.

**Acceptable for MVP because:** runtime failure is clean (script error, not state corruption). A full liveness ping at boot would require an inference round-trip which is too expensive for a gate that fires on every HA restart.

**Target:** GA — add a low-cost test call in `flynn_bootstrap_content` after entity validation, with a clear notification if the model doesn't respond.

---

# 1.0 SP1 — Queued

Post-GA service pack. Token-related items deferred from `fix/ga-batch-1`.

### Security Enforcement — `security_policy` Syscab Drawer

The System Cabinet carries a `security_policy` drawer at GA with the following
fields stubbed and documented:

- `secure: false` — GA active. Set to `true` at SP1 to activate full enforcement.
- `max_delegation_depth: 2` — GA active. Hard rule enforced now.
- `max_nesting_depth: 2` — GA active. Hard rule enforced now. Group nesting ceiling.
- `provider`, `token_endpoint`, `validation_mode`, `claims_cache_ttl` — SP1 fields.
  Plumbed and documented. Populating them at GA is a no-op — enforcement is not
  active until SP1.

**SP1 activates:** external auth provider integration (pluggable — ZenOS is the
policy plane, external stack is the auth plane). One drawer setting (`secure: true`)
enables full enforcement. The policy plane, HyperIndex claims engine, and caller_token
enforcement are all plumbed at GA. SP1 wires the external provider and flips the switch.

---

# v.next

Following GA, ZenOS-AI can evolve toward:

- Dynamic extension mapping
- Deeper long-horizon memory
- Stronger governance modules (multi-level ACL, audit log)
- Improved portability templates
- Multi-agent or persona coordination improvements
- Architectural simplification and cleanup passes
- Public persona registry

### KFC v1.1 — State Key + Master-Switch-Free Controller

Replace the `master_switch` field in KFC drawer schema with a `state` key stored directly
in drawer `meta`. Remove the kill-switch HA switch entity as the default lifecycle mechanism.
Component enabled/disabled state lives in the drawer itself — no external entity required.
Introduce a dedicated KFC controller that owns component lifecycle (enable, disable, reset)
as a first-class operation rather than a side-effect of master switch state.

**`meta` block additions:**

```yaml
meta:
  description: "Plain-English description of what this component does."
  enabled: true          # replaces master_switch — on/off stored here, not in HA switch
  requires:
    cert: "zen_level_2"  # machine-readable capability requirement — retrievable by code
    level: 2             # numeric level for range checks
```

`requires.cert` is checked at Monastery dispatch time. When zen_auth is live, TGT validation
gates against this field. A component declares its own scope of practice; the runtime
enforces it. `cert` is intentionally retrievable by code — it is not documentation, it
is a runtime predicate.

Scope: Dojo cabinet schema (KFC drawer meta), Scheduler dispatch loop, KungFu Writer,
Flynn gate-3 bootstrap validation, Monastery dispatch guard.

---

# Immediate Next Focus

GA is shipped. The system deploys, bootstraps, and maintains itself cleanly through RP2 and the 4.5.6 patch.

The next major work stream is **Prompt Finalization**:

1. **Template structure pass + rename** — `zen_os_1rc.jinja` → `zenos.jinja`, ref sweep, finalize `custom_templates/zenos_ai/` layout
2. **Prompt instrumentation + collapse** — sensor per major prompt section → validate → collapse behind `zen.prompt('ai_user')`. Persona-aware, data-layer driven.
3. **Flynn shunt** — pre-boot/degraded state routes to Flynn-only context. Wire last.
4. **Stuffiness gauge** — token pressure per section as observable metric.

Sequence: instrument → Nyx validates sensors → collapse → shunt.
