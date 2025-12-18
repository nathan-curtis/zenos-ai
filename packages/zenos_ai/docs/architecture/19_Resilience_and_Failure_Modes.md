# **19_Resilience_and_Failure_Modes.md**

# 19. Resilience, Failure Modes, and Degradation Strategies

Friday’s architecture treats failure as a *normal operating condition*, not an exception. Consumer IoT networks are noisy. Wi-Fi drops packets. ESPHome devices reboot. Vendors throttle APIs. LLM calls timeout. Home Assistant’s event bus occasionally races state transitions. A cognitive system that assumes ideal conditions will misbehave in real conditions.

This section outlines the resilience model that underpins Version 1 of Friday’s House, including:

* the failure surfaces present in Home Assistant and the local substrate
* the guardrails applied across sensing, triggering, and summarisation
* the cabinet’s durability guarantees
* the summariser’s error-handling approach
* how the system behaves under degraded conditions
* how human operators can observe and debug system failures

The objective is not perfection, but *predictability*: a system that degrades gracefully, fails loudly, and never corrupts its long-term cognitive state.

---

## **19.1 Primary Failure Surfaces in the Substrate**

The architecture recognises several recurring classes of instability at the HA layer:

### **(1) State Unavailability**

Entities often report:

* `unknown`
* `unavailable`
* absent attributes
  These cannot be treated as meaningful data.

### **(2) Event Loss or Duplication**

Triggers may be:

* dropped during HA restarts
* duplicated by chatty integrations
* fired in unpredictable order

### **(3) Timing Skew**

Automation execution order can drift:

* delayed triggers after supervisor load
* rapid bursts after reconnect
* interleaving across unrelated domains

### **(4) External Integration Failures**

Calendars, cloud APIs, and notifications can:

* timeout
* rate limit
* return malformed payloads

### **(5) LLM Pipeline Errors**

* timeout
* malformed JSON
* incomplete summariser results
* retry loops under load

### **(6) Cabinet Integrity Risks**

* mismatched schemas
* missing drawers
* malformed write payloads
* stale timestamps

The architecture assumes all these can happen and designs for them explicitly.

---

## **19.2 Defensive Sensing and Guarded State Access**

Home Assistant is an eventually-consistent, best-effort state machine. As such, Friday never reads raw state without validation.

The system uses:

### **Guard 1: Explicit Validation for `unknown`/`unavailable`**

Any entity reporting these states is:

* treated as absent
* ignored for summarisation
* logged for Resilience Manager review

### **Guard 2: Safe Attribute Access**

Attributes are accessed with patterns such as:

```jinja
{{ state_attr(entity, key) | default(None) }}
```

If:

* the attribute is missing
* the integration stringifies a dictionary
* the attribute is null or empty

…the system falls back to safe defaults.

### **Guard 3: Robust Cabinet Detection**

A sensor is only treated as a cabinet if:

* the `variables` structure exists
* validation markers match expected signatures
* GUIDs align with the manifest

This prevents false positives from misconfigured template sensors.

---

## **19.3 Scheduler Resilience and Load Management**

The **Zen DojoTools Scheduler** must remain stable even under heavy event load.

It uses:

### **• Trigger Bucketing**

Events are grouped by component domain:

* security events
* water manager updates
* room manager changes
  This prevents a single chaotic integration from cascoding through the entire system.

### **• Concurrency Limiting**

Automations run in queued mode with explicit caps.
Duplicate invocations do not spawn parallel summariser runs.

### **• Delayed Noncritical Summaries**

Low-priority summaries wait 60–120 seconds before execution.
This lets the system absorb bursty HA state churn.

### **• Context-Bound Triggers**

Certain components execute only during meaningful time windows, reducing unnecessary load.

---

## **19.4 Summariser Resilience and Error Handling**

The summariser is the cognitive engine. It is also the most failure-prone component due to network conditions, model load, or payload size.

The architecture addresses this via:

### **19.4.1 Strict Timeout Behaviour**

A summariser call that exceeds its timeout:

* returns an explicit failure object
* triggers no write to cabinets
* leaves existing katas untouched
* does not block the scheduler

### **19.4.2 Strict JSON Parsing**

No heuristic recovery.
If JSON fails to parse:

* the event is logged
* the write is cancelled
* the summariser returns an `errors` field in the kata

Strictness protects the long-term cognitive state.

### **19.4.3 Kata Self-Description**

Every kata encodes:

* `confidence`
* `urgency`
* `criticality`
* `errors`

This allows Friday to treat degraded katas as advisory rather than authoritative.

---

## **19.5 Cabinet Integrity and Write Safety**

The cabinet is the long-term cognitive memory of the system.
Corruption here distorts Friday’s worldview.

To prevent this:

### **19.5.1 Schema Validation**

Each cabinet volume includes:

* a schema version
* a signature marker
* required drawer structure

Write attempts that violate schema:

* log errors
* reject the write
* maintain existing content

### **19.5.2 Timestamped Writes**

Every update includes:

* `timestamp`
* `stale_after` markers
  Consumers can detect stale content and downgrade confidence.

### **19.5.3 Idempotent Writes**

Summarisers write *whole drawers*, not patches, to prevent partial merge corruption.

### **19.5.4 Controlled Overrides**

“Force” write flags are allowed only in:

* bootstrap
* migration
* cabinet repair

Normal summariser paths never use these.

---

## **19.6 Staleness Detection and Graceful Degradation**

The system explicitly models staleness.

When:

* the LLM is unreachable
* summariser calls repeatedly fail
* HA is unstable after a restart

Then:

* old katas remain readable
* confidence is lowered
* urgency is reduced
* advisory mode activates
* Friday refrains from high-confidence claims

The system behaves like a careful operator acting under uncertainty.

---

## **19.7 Observability and Human Recovery Path**

Failures must be observable.

Friday’s House exposes:

### **• Scheduler Events**

Every trigger written into the `zen_scheduler` drawer.

### **• Component Katas**

Each contains:

* what triggered it
* how confident it is
* any internal error notes

### **• Cabinet Drawers**

Readable historical state for every cognitive component.

### **• Zen Inspect**

A consistent view of:

* the underlying HA entity
* its attributes
* its state history
* how it maps into RoomState
* how it fed into the summariser

**Human debugging path:**

1. Inspect the failing entity
2. Check its RoomState representation
3. Read the last kata for the component
4. Check scheduler events for that run
5. Inspect cabinet drawer for corruption or staleness

This yields a deterministic workflow for understanding any misbehaviour.

---

## **19.8 Architectural Principles for Handling Failure**

These patterns embody the core principles of Friday’s architecture:

1. **Never trust raw state without validation.**
2. **Keep sensing and cognition separate.**
3. **Favor explicit, timestamped structures.**
4. **Reject unsafe writes instead of salvaging them.**
5. **Maintain observable transformation lineage.**
6. **Degrade into advisory mode when signals are noisy.**
7. **Ensure humans can always reconstruct the reasoning chain.**

---

## **19.9 Summary**

Resilience is not an optional extension. It is foundational.

Friday’s House handles:

* unstable sensors
* incomplete state
* flapping integrations
* LLM inconsistencies
* cabinet schema mismatch
* load spikes
* restart turbulence
* partial kata generation

…without compromising long-term cognitive integrity or user trust.

This establishes Version 1 as a stable platform on which advanced features—identity, session tokens, visa onboarding, tool shunting, and the MCP channel—can be layered in future releases without jeopardizing the core cognitive architecture.an immediately draft Section 20 or move ahead into the future-work reserved sections.
