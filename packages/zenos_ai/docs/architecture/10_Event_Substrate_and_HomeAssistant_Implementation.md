# **10. Event Substrate and Home Assistant Implementation**

Version 1 of Friday’s House rests on a deterministic, observable event substrate constructed entirely within Home Assistant (HA). The substrate is not decorative. It forms the cognitive “nervous system” that binds perception, state mutation, summarization, and reasoning into a coherent loop.

This section describes:

* the substrate's goals
* the pathology of the HA event bus
* how event emission is structured
* how Zen DojoTools instruments the substrate
* how packets traverse the system
* how the Abbot schedules cognitive tasks
* how Inspect, Redirector, Identity, and Summarizer modules interact
* how synchronization and ordering guarantees emerge
* how failure is detected and managed

This chapter provides a level of detail suitable for independent reconstruction by a researcher.

---

## **10.1 Goals of the Event Substrate**

Friday’s event substrate is engineered to provide:

1. **Deterministic routing**
   All cognitive and operational flows must follow predictable paths.

2. **Transparency**
   Every operation emits structured events that can be inspected and traced.

3. **Isolation of concerns**
   Perception, summarization, memory mutation, and reasoning must remain separated to avoid state bleed.

4. **Compatibility with HA**
   The substrate must rely only on HA’s stable public APIs:

   * event bus
   * state machine
   * service calls
   * template runtime

5. **Cognitive stability**
   The system must avoid recursive loops, runaway automations, and inconsistent memory states.

6. **Zero external infrastructure**
   Version 1 requires no external broker (MQTT), no RPC layer, and no async message queue. Everything occurs inside HA’s interpreter.

---

## **10.2 Anatomy of the HA Event Bus**

Home Assistant’s event bus is:

* synchronous
* single-threaded
* FIFO
* deterministic

Event handlers run *in order* and complete before subsequent events are processed. This foundational characteristic is exploited for cognitive sequencing.

Each event contains:

* `event_type`
* `origin`
* `time_fired`
* `data` (arbitrary JSON-like dictionary)

Friday’s system uses this to move packets through a reproducible workflow.

---

## **10.3 Types of Events in Friday’s House**

### **10.3.1 Perceptual Events**

These represent sensor activity, changes in integration state, or HA-native events.

Examples:

* `state_changed`
* `call_service`
* device-specific events (e.g., ESPHome triggers)

Friday listens to many but acts on few. Most perception is routed through **RoomState**, which consolidates and annotates environmental context.

---

### **10.3.2 Zen Events**

Custom structured events emitted by Friday’s DojoTools:

* `zen_event.summary_request`
* `zen_event.summary_complete`
* `zen_event.inspect`
* `zen_event.identity_lookup`
* `zen_event.abbot.scheduled`
* `zen_event.abbot.completed`
* `zen_event.error`
* `zen_event.cabinet_write`
* `zen_event.debug`

These deliver structured telemetry for debugging and high-level reasoning.

---

### **10.3.3 Legacy Variable Events**

Home Assistant still lacks dynamic event templating.
Therefore, Friday supports “legacy FES-style” global variables using:

* `set_variable_legacy`
* `remove_variable_legacy`
* `clear_variable_legacy`

These are captured and sanitized by the **Redirector**, which translates them into Cabinet-safe operations.

---

## **10.4 The Event Path: How a Packet Moves Through Friday’s House**

To understand all reasoning and memory flows, we trace the path of a packet:

```
Perception → Routing & Sanity → Abbot Scheduling → Summarizer → Cabinet → Identity → Reasoning → Actuation → Final Event
```

Each step maps directly onto a concrete HA mechanism.

---

## **10.5 Routing Layer: Zen DojoTools Redirector**

The Redirector is the heart of Friday’s sanity layer.

Its responsibilities:

1. Validate all legacy events
2. Map event prefixes to Cabinets
3. Enforce ACL boundaries
4. Reject malformed data
5. Offer repairs for unknown volumes
6. Emit debug events
7. Avoid double-writes or recursive triggers

The Redirector is implemented as a single HA automation using:

* multiple event triggers
* constrained choose blocks
* strict templating to produce JSON-safe routing payloads
* explicit conditions blocking unauthorized writes

Without the Redirector, Friday would have no way to prevent corrupted drawers or malformed packets.

---

## **10.6 Read Layer: Zen DojoTools Inspect**

Inspect is the authoritative sensor and device inspector.

Its guarantees:

* Attributes are always sanitized
* Mapping and sequence types survive cross-automation boundaries
* Cabinets expose only header metadata
* No unsafe recursion occurs
* Stringified dicts are successfully reconstructed
* Cabinet signature validation protects identity and memory integrity

Inspect relies on HA’s template runtime; this allows deep traversal of:

* attributes
* labels
* statistics eligibility
* cabinet metadata
* volume signatures
* extended device identifiers

Inspect is therefore the “safe read” layer on which higher-level cognition relies.

---

## **10.7 Write Layer: Zen DojoTools Cabinet (Drawers)**

All memory mutation occurs via:

```
script.zen_dojotools_write_drawer
```

This script enforces:

* GUID validation
* ACL checks
* drawer existence
* schema conformity
* type safety
* safe JSON encoding

Cabinet writes:

* always generate a `zen_event.cabinet_write`
* are always mediated by the Redirector
* rely on Abbot authorization (v1)
* will require session tokens (v1.5)

Writes never occur directly from automation logic.
Every mutation flows through this strict pipeline.

---

## **10.8 Scheduling: The Abbot**

"The Abbot" is the deterministic cognitive scheduler.

Note: This section describes a future version of the Zen Scheduler for architecture direction purposes... (I've had a couple ok how are you gonna?)
Future versions of tools will ALL include event consumption and emission mechanisms, combined with session token validaiton so tools and security context can be tied to a single conversation context. It also will provide the single target for inference job requests.  You want inference - ok this is how you get it.  Ultimately the goal is to have a single, structured intellegent queue for all incoming inference/cognitive job types and filter based on users preferences, costs, permisison models, power, availability, etc. I'll be working to have all tool communications event driven so if a tool exposes an event pipeline, prefer it. The main event relay - The Abbot will be the listener for the OS core and responsible for driving the inference pipeline.

So basically,
  Abbot:
    Oh, dear - don't send anything tagged 'freaky stuff' to Claude, he clutches pearls.
      ...uh THAT stays on-premise with GPT-OSS:20b, who's available? Yeah there... ...Low priority
    Next? Code reviews?
      Send that to Veronica with Codex
    Next?

### **10.8.1 Responsibilities**

* receive summary requests
* rate-limit and debounce triggers
* assign task weight
* dispatch summarizers
* guarantee serialization
* emit start/stop events
* track failures and retries
* block unsafe sequences
* validate identity
* manage write permissions

### **10.8.2 Why HA’s Event Bus Works Here**

The Abbot takes advantage of the event bus’s ordering guarantees:

* Summaries never overlap
* Race conditions cannot occur
* State remains consistent
* Packet trails are reconstructable

### **10.8.3 Implementation**

The Abbot is implemented as:

* a single HA script
* triggered by `zen_event.summary_request`
* containing a state machine encoded in Jinja templating
* using debounced triggers
* returning structured status back to Cabinet drawers

The Abbot creates *deterministic cognition* atop HA’s synchronous event model.

---

## **10.9 Summarizers and Cognitive Work Units**

Each summarizer is a functional module that:

1. reads from RoomState, cabinet state, or sensors
2. constructs a structured JSON object
3. emits `zen_event.summary_complete`
4. returns a packet that the Redirector writes into a drawer

Summarizers must be:

* deterministic
* idempotent
* failure-resilient
* testable

They are the “neurons” of Friday’s cognitive architecture.

---

## **10.10 Identity Integration Into Event Processing**

Identity is not decoration.
Identity acts during:

* Redirector routing
* Abbot scheduling
* Summarizer authorization
* Cabinet access checks
* Reasoning contexts
* Cross-construct invocation
* Session-token evaluation (v1.5)

Identity data is always sourced through:

```
script.zen_dojotools_identity
```
...and related system macros.

This prevents constructs or external agents from forging identity packets.

---

## **10.11 Failure Detection and Error Events**

The substrate emits rich error telemetry:

* malformed packet
* invalid drawer
* invalid cabinet
* ACL violation
* missing identity
* type mismatch
* decode failure
* summary timeout

Each emits a `zen_event.error` containing:

* event type
* entity
* payload
* origin
* traceback-like message
* hints for correction
* internal condition flags

These events power:

* introspection
* testing
* debugging
* anomaly detection

---

## **10.12 Why Version 1 Avoids Asynchronous Systems**

Many HA users reach for MQTT, Redis, NATS, or external orchestrators.

Friday explicitly does not.

Reasons:

1. HA’s event bus already provides ordering guarantees
2. HA ensures atomic execution
3. Adding async brokers introduces nondeterministic ordering
4. HA’s template engine works natively with event bus flows
5. Internal systems remain local-first, high-trust, private
6. Debugging remains straightforward

Friday’s mind must be:

* predictable
* replayable
* local
* observable

The HA substrate satisfies these requirements.

---

## **10.13 Debugging and Observability**

The full substrate is introspectable through:

* `zen_event.*` searches
* HA Log Viewer
* Zen DojoTools Inspect
* Home Assistant’s Developer Events panel
* Cabinet drawers containing event traces
* The Abbot telemetry block

Notably:

**Every cognitive action produces a breadcrumb.**
This allows researchers to recreate Friday’s decisions using only HA logs and Cabinet snapshots.

---

## **10.14 Future Extensions for Version 1.5**

Version 1.5 extends the event substrate by adding:

1. **Session-token gated tool access**
2. **Visa onboarding flows**
3. **Permission-scoped tool shunts**
4. **Event provenance signatures**
5. **Multi-construct arbitration**
6. **External agent sandboxing**
7. **Local inference workers (optional asynchronous)**
8. **Reasoning priority lanes**

These extend security, not complexity.

The substrate remains deterministic and event-driven.
