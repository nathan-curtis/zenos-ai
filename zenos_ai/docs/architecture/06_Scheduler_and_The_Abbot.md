# 6. The Scheduler (The Abbot)

*A deterministic cognitive governor for Friday’s House*
"Hayyyy Abbot!" ( He really hates that...)

**Current implementation status (2026-08-04):** "The Abbot" is not a single
component — it's the name for the scheduling+dispatch function, realized today
jointly by two real, separate files: `dojotools_scheduler.yaml`
(`zen_dojotools_scheduler` — trigger dispatch, debounce/shed logic) and
`dojotools_dispatcher.yaml` (`zen_dojotool_dispatcher` — event-driven routing,
decouples callers from direct script calls). Most of this chapter — the
per-component scoring/cooldown/quiet-hours policy model in particular — describes
a design target, not current line-by-line behavior; §6.5-6.7 below have been
corrected to describe what's actually implemented today. The rest of the chapter
(§6.1-6.4, §6.8-6.11) remains architecture-direction material, same status as the
chapter's own closing author's note already partially acknowledges. Future work
(per direct confirmation, not inferred): tagging and evaluation of each job so it
can be routed to the correct inference core with the right metadata — this is
real stated direction, not yet built.

The Scheduler—referred to internally as **The Abbot**—is the command and metronome of Friday’s cognitive architecture. It is not merely a cron-like time-based trigger but a **policy-driven orchestrator** of reasoning, summarization, and state transitions within the Monastery. Every Kata, every summarizer run, and every system-level reflection originates in the Abbot’s discretion.

This chapter describes the design, operation, guarantees, and user-customizable behavior of the Abbot, with enough depth that an engineer can reconstruct the entire governance mechanism directly from first principles and Home Assistant primitives.

---

# 6.1 Purpose and Conceptual Role

The Abbot exists to answer a deceptively simple question:

**“When should the system think?”**

Reasoning in ZenOS-AI is *expensive, stateful, and consequential*. Continuous reasoning is undesirable, free-running thought is unsafe, and user interactions alone do not provide reliable coverage of all system needs. Therefore, the system requires a centralized authority that determines:

* **when to generate new Katas**
* **which components need fresh summaries**
* **how to interpret sensor and cabinet triggers**
* **how to enforce cognitive invariants and pacing**
* **how to quarantine or retry failed summarization tasks**

The Abbot enforces these constraints while preserving a predictable cadence of reflective thought across the House.

---

# 6.2 Architectural Boundaries

Version 1 of the Abbot operates entirely within Home Assistant’s native constructs:

* event triggers
* state triggers
* time triggers
* scripts
* helpers (input booleans, selectors, sensors)
* templated variables and guards

The architecture avoids external schedulers, cloud brokers, queues, websockets, or custom inference servers. Every cognitive action must be reducible to:

1. a Home Assistant event,
2. a DojoTools script invocation, and
3. a write into a cabinet drawer.

This ensures that the Abbot is fully local, resilient to connectivity issues, observable, and debuggable through standard Home Assistant tools.

---

# 6.3 Abbot Design Goals

The Abbot is engineered to guarantee:

### 6.3.1 Determinism

Given identical environmental state and identical triggers, summarizer execution must be identical. No silent branching, no randomization, no concurrency-driven race conditions. Determinism allows for forensic reconstruction and rigorous debugging.

### 6.3.2 Bounded Reasoning

No component may trigger a summarizer run unless explicitly allowed. No summarizer may preempt another. The Abbot enforces:

* a maximum frequency for each component
* a maximum total concurrency (typically 1 per reasoning tier)
* explicit cooldown intervals
* reason-based invocation (not raw random events)

### 6.3.3 User Customizability

Every home is different. Users can override:

* which triggers matter,
* how often summarization should occur,
* which components participate,
* when summarization is suppressed,
* how errors are escalated.

The Abbot is meant to be tuned.

### 6.3.4 Safety

If a cognitive module stalls, errors, or produces invalid output, the Abbot:

* freezes further runs for that component,
* preserves last known good Katas,
* emits telemetry events,
* logs structured errors for review.

---

# 6.4 Trigger Taxonomy

The Abbot responds to three categories of triggers. Understanding these is essential for reproducing or auditing Friday’s behavior.

## 6.4.1 Structural Triggers

These originate from high-level system structure:

* cabinet updates
* identity changes
* persona capsule changes
* registration or de-registration of components
* modifications to templates or schemas

Structural triggers force recalculation of components whose interpretations depend on identity, security rules, or available drawers.

## 6.4.2 Environmental Triggers

These originate from Home Assistant’s entity graph:

* device_class changes
* room occupancy transitions
* water flow anomalies
* security sensor spikes
* hot tub state changes
* environmental stat updates (humidity, power draw)

Environmental triggers fire often but only lead to reasoning if the affected component declares interest.

## 6.4.3 Temporal Triggers

Clear time-based triggers:

* hourly or daily cadence
* nighttime quiet hours
* sunrise or sunset based on local solar position
* rolling window timers for energy, water, or presence-based Katas

Temporal triggers serve as a catch-all to reaffirm Katas even when sensors remain silent.

---

# 6.5 Trigger Resolution Process — as actually implemented

*(Updated 2026-08-04: this section previously described a scoring/cooldown/quiet-hours
pipeline that doesn't exist in code. Rewritten to match `dojotools_scheduler.yaml`.)*

The real dispatch mechanism has no generic "score triggers" step. Each cognitive
component declares its own `trigger_subscriptions` (a list of Scheduler event
IDs it cares about — e.g. `daily_midnight`, `water_flow_stop`,
`force_summary`) and a `pipeline_tier` in its own KFC (Kung Fu Component) drawer.
When a trigger fires:

1. The Scheduler checks which components have that trigger's ID in their own `trigger_subscriptions`.
2. Each matching component is dispatched or shed based on its `pipeline_tier` and the current queue depth — not a relevance score.
3. **`pipeline_tier: keeper`** (the default) — dispatched on all standard triggers; deferred when queue depth ≥ `shed_keeper_at` (default 8, configurable in `zen_scheduler_config`).
4. **`pipeline_tier: ambient`** — dispatched only on slower triggers (e.g. hourly); shed on fast triggers (`quarter_hour`, `every_10_minutes`) and when queue depth ≥ `shed_ambient_at` (default 4).
5. **`pipeline_tier: super`** — reserved for SuperSummary; **`pipeline_tier: system`** — infrastructure components.
6. Shed work isn't lost — a drain router recovers the most-stale shed-eligible component once queue depth falls back below `drain_below` (default 3), and a component whose Kata exceeds its own `staleness_minutes` ceiling fires regardless of queue depth (a starvation guard).
7. Selected components run Ninja Summarizer or SuperSummary.
8. Successful/failed runs emit real `zen_event` kinds — see §10.3.2 for the actual vocabulary (`summarizer_start`, `ninja_failure`, `kata_emit`, etc.) — not a generic "telemetry" event.

There is no `cooldown_seconds` field or `quiet_hours` policy block anywhere in
the current scheduler or summarizer code — the shed-at-queue-depth mechanism
above is what actually prevents burst storms (e.g. rapid door events), not a
per-component cooldown timer. `delay_seconds` (documented per-KFC) sets dispatch
priority ordering, not a cooldown.

---

# 6.6 Component Policies and Scheduling Logic — as actually implemented

Each cognitive component defines its policy directly in its own KFC (Kung Fu
Component) drawer in the Dojo Cabinet — not a separate "policy block."

### 6.6.1 Trigger Subscriptions

A declarative list of Scheduler event IDs that cause this component to dispatch:

```yaml
trigger_subscriptions:
  - daily_midnight
  - water_flow_stop
  - force_summary
```

### 6.6.2 Pipeline Tier and Delay

Dispatch priority and shed-eligibility, not a cooldown timer:

```yaml
pipeline_tier: keeper   # keeper (default) | ambient | super | system
delay_seconds: 120      # how long to wait after trigger before dispatching — lower = higher priority
```

### 6.6.3 Staleness Ceiling

The age at which a shed-eligible component's Kata gets a forced recovery dispatch regardless of queue depth:

```yaml
staleness_minutes: 1440   # default 24h
```

### 6.6.4 Sensitivity Settings

Domain-specific thresholds a component's own `component_summary` prose references — these aren't a distinct Scheduler mechanism, just data the component's prompt reads:

```
flow_loss_threshold_seconds: 20
stabilization_period_seconds: 120
```

Custom hardware triggers (e.g. a specific sensor going critical) that need
instant dispatch use a dedicated per-KFC trigger file firing a `zen_event` with
`kind: summary_force` and a `component:` field — not a Scheduler-level policy
block.

---

# 6.7 Abbot’s Execution Pipeline — as actually implemented

### Step 1: Trigger Ingestion

HA events and state changes fire the Scheduler's own trigger set.

### Step 2: Subscription Matching

For the fired trigger, the Scheduler resolves which components declared that
trigger ID in their own `trigger_subscriptions` — no separate quiet-hours
override step exists.

### Step 3: Dispatch or Shed

Matched components are dispatched or shed per §6.5's `pipeline_tier`/queue-depth
logic — not a generic "reasoning dispatch" step:

* Ninja Summarizer tasks for individual components
* SuperSummary task for the whole-home consolidation

Concurrency is bounded (no two tasks for the same component run at once), and a
component only dispatches when its subscription genuinely matched — but there is
no separate "valid reason" check beyond the subscription match itself.

### Step 4: Validation and Write

Summaries are validated and written into the Kata cabinet via the Cabinet package (see §10.5/§10.7).

### Step 5: Telemetry

Real `zen_event` kinds are emitted for success/failure — see §10.3.2, not a single generic "structured event."

---

# 6.8 Error Handling and Recovery

The Abbot implements strict safety mechanisms:

### 6.8.1 Summarization Failures

If a model returns malformed JSON or invalid fields:

* the Abbot logs a failure event
* the component is marked “cooldown-locked”
* the previous Kata is retained
* optional “retry window” or exponential backoff may be used

### 6.8.2 Entity Anomalies

If Inspect detects malformed attributes, null states, or corrupt cabinet registrations:

* the Abbot refuses summarization
* the Summarizer receives a note about corrupted state
* user-facing warnings may be generated depending on policy

### 6.8.3 Global Lockdown

If identity stability or cabinet validation is compromised:

* the Abbot suspends all summarization
* the High Priestess layer enters safe mode
* Friday informs the user that identity needs repair

---

# 6.9 User Customization and Extensibility

The Abbot is intentionally modular:

* components can opt-in or opt-out
* users can tune thresholds
* summarization frequency can be changed
* suppression rules can be added (e.g., ignore garage door during mowing)
* future modules can define their own Katas and policy blocks
* advanced users can attach additional listeners to Abbot telemetry

In this way, the Abbot is not only the governor but also the **maintainer of cognitive culture** in the household.

---

# 6.10 Relationship to Classical Scheduling and Cognitive Architectures

The Abbot borrows ideas from:

* **blackboard systems**, where multiple agents write structured summaries
* **rule-based expert systems**, where triggers activate modules only when necessary
* **attention mechanisms**, where noisy sensor spikes must be filtered
* **homeostatic regulators**, where stability matters more than immediacy

However, it diverges in key ways:

* it operates fully on top of Home Assistant’s event graph
* it controls not actuators but reasoning itself
* its outputs are not actions but **Katas**
* its role is not temporal control but **cognitive pacing**

---

# 6.11 Outlook: Toward Identity-Aware Scheduling

Future versions of the Abbot will incorporate the ZenAI Identity Manifest:

* identity-based permissions
* persona capsules
* visas
* session-token-guarded tool invocation
* household and family access control
* intent routing based on authenticated users

This will allow:

* reasoning on behalf of different users
* multiple assistants with differing privileges
* identity-confirmed model actions
* user-customizable “thinking styles” per household member

Version 1 lays the deterministic foundation on which these features will be built.

Author's Note: The Abbot is a work in progress and some features may be held for future release. eg. Quiet Hours.
