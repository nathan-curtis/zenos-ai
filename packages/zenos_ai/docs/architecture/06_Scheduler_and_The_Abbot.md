# 6. The Scheduler (The Abbot)

*A deterministic cognitive governor for Friday’s House*
"Hayyyy Abbot!" ( He really hates that...)

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

# 6.5 Trigger Resolution Process

The Abbot runs a deterministic resolution algorithm:

1. **Collect relevant triggers** occurring within a small time window.
2. **Score them** based on relevance to each cognitive component.
3. **Filter** by component policy (ignore/noise thresholds/quiet hours).
4. **Enforce cooldown** periods (no component may run twice too quickly).
5. **Select components** eligible for summarization.
6. **Run Ninja Summarizer** or SuperSummary in strict order.
7. **Validate and commit** new Katas.
8. **Emit telemetry** for both successful and failed runs.

This ensures that even in bursts (e.g., someone opening multiple doors rapidly), only the earliest meaningful event triggers reasoning.

---

# 6.6 Component Policies and Scheduling Logic

Each cognitive component (e.g., Security Manager, Water Manager) defines its **policy block** in its corresponding Dojo drawer.

Policy blocks include:

### 6.6.1 Triggers of Interest

A declarative list of entity domains, labels, or attributes that should initiate summarization.

Example (Security Manager):

```
triggers_of_interest:
  - device_class: door
  - device_class: window
  - label: security_attention
```

### 6.6.2 Trigger Cooldowns

A cooldown preventing repeated computation:

```
cooldown_seconds: 30
```

When combined with door sensors, this prevents summarization storms.

### 6.6.3 Quiet Hours

Policies that override or suppress summarization during defined intervals:

```
quiet_hours:
  start: "22:00"
  end: "06:00"
  mode: evaluate_only
```

### 6.6.4 Sensitivity Settings

For example, for Water Manager:

```
flow_loss_threshold_seconds: 20
stabilization_period_seconds: 120
```

These allow domain experts to tune sensitivity.

---

# 6.7 Abbot’s Execution Pipeline

The Abbot dispatches summarization through a strict pipeline:

### Step 1: Trigger Ingestion

HA events and state changes are ingested into an internal “trigger context” object.

### Step 2: Policy Matching

Using DojoTools’ label and index functions, the Abbot resolves:

* which components are impacted
* which Katas require recalculation
* which rules override triggers (e.g., quiet hours)

### Step 3: Reasoning Dispatch

The Abbot fires one or more tasks:

* Ninja Summarizer tasks for components
* SuperSummary task for the system

The Abbot ensures:

* no two tasks run for the same component concurrently
* tasks honor minimum spacing
* no task is run without a valid reason

### Step 4: Validation and Write

Summaries are validated and written into the Kata cabinet.

### Step 5: Telemetry

The Abbot publishes a structured event for observability.

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
