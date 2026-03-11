# ZenOS-AI Roadmap

*Local-first cognitive architecture for persona-scale AI on Home Assistant*

ZenOS-AI is a cabinet-centric AI framework for deterministic, inspectable household cognition.

---

## Current Status

**4.1.0 RC2 — Complete**

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

# v.next

Following GA, ZenOS-AI can evolve toward:

- Dynamic extension mapping
- Richer KFC ecosystem
- Deeper long-horizon memory
- Stronger governance modules (multi-level ACL, audit log)
- Improved portability templates
- Multi-agent or persona coordination improvements
- Architectural simplification and cleanup passes
- Public persona registry

---

# Immediate Next Focus

The RC2 baseline is done. The system deploys, runs, and maintains itself.

The GA focus is proving the full path from **zero to live** and from **legacy to canonical** — cleanly, deterministically, and repeatably.

`fresh install → canonical state → first live agent → legacy import → regenerated derived state`

That is the real GA line.
