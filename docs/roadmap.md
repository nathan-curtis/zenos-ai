# ZenOS-AI Roadmap

*Local-first cognitive architecture for persona-scale AI on Home Assistant*

ZenOS-AI is a cabinet-centric AI framework for deterministic, inspectable household cognition.
This roadmap reflects the current state of the system and resets the milestone line to match reality:

- `RC1` is effectively complete
- current focus is `RC2`
- `RC2` is the deployment and bootstrap milestone
- `GA` remains the consolidation and lifecycle milestone

---

## Current Status

The project appears to have cleared the original `RC1` intent already, even if it was not formally called done at the time.

What exists now in practical terms:

- canonical Ring-0 cabinet package structure exists under `/packages/zenos_ai`
- core cabinet health and label health concepts exist
- Flynn onboarding/recovery exists in working form
- core DojoTools exist and are operational
- Monastery / summarizer pipeline exists
- HyperIndex exists in parallel evaluation form
- package migration has started and is materially underway
- deployability is not yet fully locked
- clean bootstrap from reset is not yet the primary proven path

That means the project is no longer in "can this architecture work?" territory.
It is in "can this architecture be deployed, initialized, recovered, and imported cleanly?" territory.

---

# 1.0 RC1 - Achieved Foundations

`RC1` established the architecture and proved the substrate.

## Completed or Functionally Present

### Identity and Local Selfhood Authority

Stable GUID-oriented identity structures for household, family, person, AI persona, and system roles.

### Ring-0 Cabinet Model

Canonical deployable cabinets have been defined as package-rooted storage modules, with `packages/zenos_ai/zenos_cabinets.yaml` serving as the core storage substrate.

### Cabinet Health and Structural Validation

Health sensors and validation logic exist to verify labels, cabinet presence, and expected slot structure.

### Label Framework

A consistent label-driven architecture exists and is already central to cabinet resolution, indexing, and tool behavior.

### FileCabinet Runtime

Zen DojoTools FileCabinet exists as the typed operational storage interface for deterministic drawer operations.

### Volume Redirector Pattern

Legacy flows can be translated into governed cabinet operations, establishing a bridge from older installs to the canonical storage model.

### HyperIndex RC Track

A parallel hypergraph-capable index exists for evaluation without making it a hard dependency of the core deployment path.

### Monastery and Kata Memory

The summarization and persistent memory loop exists in working form and supports structured reflective state.

### Flynn Bootstrap / Recovery Concept

Flynn exists as the infrastructure-layer bootstrap and failback agent for incomplete or unhealthy core states.

---

# 1.0 RC2 - Current Milestone

## Theme

`RC2` is the deployment, reset, bootstrap, and import milestone.

The goal is not merely to have working components in one reference install.
The goal is to prove that ZenOS-AI can be:

- deployed from script
- reset to canonical labels and cabinets
- bootstrapped from minimal state
- brought to a live default agent
- imported from a pre-existing install into the correct cabinet slots

## RC2 Core Outcome

The target state is:

`install -> labels -> Ring-0 cabinets -> default household/family/person/AI -> agent live -> import old memory -> regenerate derived state`

---

## RC2 Deliverables

### 1. Canonical Deployable Core

ZenOS-AI core must be package-rooted and deployable from `/packages/zenos_ai`.

Core package responsibilities:

- canonical Ring-0 cabinets
- core health validation
- core scripts
- core automations
- Flynn bootstrap and recovery support
- baseline support sensors

The canonical Ring-0 cabinet package remains the single source of truth for default deployable storage.

### 2. Legacy Compatibility as a Temporary Layer

Any non-core or legacy cabinets from the reference install are not part of Ring-0.
They are treated as:

- optional compatibility mappings
- import sources
- temporary redirector targets during migration

This separation is now a hard architectural rule.

### 3. Resettable Label Install

RC2 must support a real reset path:

- remove legacy `zen*` labels
- install canonical label set
- restore structural health
- continue bootstrap safely

The system must be able to survive a deliberate label reset and recover through canonical package logic.

### 4. Deterministic Bootstrap of Default Slots

The system must be able to come up from near-empty canonical state and create:

- default household
- default family
- default person/user
- default AI user
- required system/dojo/kata/history structures

This bootstrap must be deterministic enough to support scripted deployment.

### 5. First-Live Agent Path

The system must prove that a default agent can load and answer coherent baseline questions from clean state before historical import occurs.

This is the critical boundary between "architecture exists" and "deployable operating system".

### 6. Legacy Memory Import

After clean bootstrap succeeds, RC2 must support controlled import of prior state into the correct canonical cabinet slots.

Import order should favor:

- raw identity and relationship facts
- durable memory
- essential persistent context
- optional historical material

Derived artifacts should be rebuilt where possible rather than blindly copied forward.

### 7. Post-Import Regeneration

After import, the system should regenerate and reconcile:

- manifests
- indexes
- summaries
- health state
- prompt-ready derived context

This is how the system becomes a native canonical install rather than a legacy shell.

---

## RC2 Workstreams

### A. Package Consolidation

Move active primary ZenOS runtime logic out of root monoliths and into package-owned files.

Focus on active definitions, not backups.

Priority order:

1. cabinet-adjacent support
2. FileCabinet / Manifest / Identity / Index / Query / Inspect
3. labels / library / history / admin tools
4. core automations and redirectors
5. Flynn and bootstrap support
6. secondary plugins and non-critical stragglers

### B. Bootstrap Contract

Define the minimum drawers and relationships required to produce a first-live default system.

This includes at least:

- who this household is
- who the default family is
- who the default user is
- who the default AI is
- what matters now
- enough time/place/system context to reason

### C. Import Mapping

Map legacy cabinets, drawers, and persona memory into canonical slot destinations.

Every importable source should be classified as:

- direct import
- transform on import
- regenerate
- discard

### D. Verification and Smoke Testing

Each migration batch needs a repeatable test gate.

Minimum checks:

- config loads
- labels validate
- cabinets validate
- FileCabinet read/write works
- manifest works
- identity works
- index works
- agent answers baseline prompts
- import lands in the right slots
- summarizer path remains healthy enough to operate

---

## RC2 Exit Criteria

ZenOS-AI RC2 is complete when the following are true:

- core package tree is authoritative for deployable ZenOS behavior
- canonical labels can be installed after a reset
- Ring-0 cabinets validate cleanly
- default household, family, person, and AI can be initialized
- a default agent can start from clean canonical state
- historical state can be imported into correct canonical slots
- derived state can be regenerated successfully
- the full path is script-deployable on a fresh or resettable install

---

# 1.0 GA - Planned Release Target

## Theme

`GA` is the consolidation, lifecycle, and operator experience milestone.

Where RC2 proves deployability, GA proves polish, boundaries, and maintainability.

## GA Priorities

### Cabinet Loader v1 GA

A hardened, deterministic cabinet loader with slot enforcement, diagnostics, and clear resolution boundaries.

### Governance Model v1

Declarative operational boundaries for what agents can read, do, ask, and modify.

### Persona Importer v1

A fully defined persona lifecycle loader covering:

- identity
- proprioception
- governance
- kata
- essence
- prompt structure

### Flynn Finalization

Flynn becomes the official first-run and failback system agent with stronger health shunts, onboarding flow, and repair tooling.

### HyperIndex Consolidation

HyperIndex capabilities are folded into the canonical DojoTools Index path without leaving parallel RC dependencies exposed as architecture requirements.

### Proprioception v1 Finalization

Component reasoning becomes more graph-aware and memory-aware rather than relying primarily on static structure assumptions.

### Baseline KFC Set

GA guarantees a stable baseline set of label-driven Kung Fu Components needed for a compliant deployment.

---

# v.next

Following GA, ZenOS-AI can evolve toward:

- dynamic extension mapping
- richer KFC ecosystem
- deeper long-horizon memory
- stronger governance modules
- improved portability templates
- multi-agent or persona coordination improvements
- architectural simplification and cleanup

---

# Immediate Next Focus

The immediate objective is not new capability expansion.

The immediate objective is to finish the path from:

`reference install with mixed legacy state`

to:

`canonical package deployable system with deterministic bootstrap and import`

That is the real `RC2` line.
