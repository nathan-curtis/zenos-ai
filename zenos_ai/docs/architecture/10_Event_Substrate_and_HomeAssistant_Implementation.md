# **10. Event Substrate and Home Assistant Implementation**

*(Updated 2026-08-04: Redirector and Abbot sections corrected to describe the
shipped implementation — Redirector as the Cabinet package, Abbot as the
Scheduler+Dispatcher working jointly — and fabricated dotted event-type names
replaced with the real `event_type: zen_event` + `kind:` pattern throughout.)*

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

Custom structured events emitted by Friday’s DojoTools do not use per-purpose
dotted event types. Every one shares a single flat `event_type: zen_event`; the
specific occurrence is carried in a `kind` field nested inside
`event.data.event.kind`. Real `kind` values in current use include
`summary_force`, `ninja_force`, `supersummary_force`, `ninja_failure`,
`ninja_context_overflow`, `monk_failure`, `kata_emit`, `emission_suppressed`,
`cabinet_mounted`, `cabinet_dismounted`, and others — see
`zen_dojotools_event_emitter` (`mode=help`) for the canonical, current lexicon.
A consumer filters on `event_type: zen_event` and switches on
`event.data.event.kind`, not on distinct top-level event names.

These deliver structured telemetry for debugging and high-level reasoning.

---

### **10.3.3 Legacy Variable Events**

Home Assistant still lacks dynamic event templating.
Therefore, Friday supports “legacy FES-style” global variables using:

* `set_variable_legacy`
* `remove_variable_legacy`
* `clear_variable_legacy`

These are captured and sanitized by the **Redirector — realized in the shipped
system as the Cabinet package** (`zen_dojotools_filecabinet`, the MCP-facing
surface, backed by `zen_sutra_filecabinet` as the internal terminus). Verified:
`dojotools_filecabinet.yaml` is exactly where `set_variable_legacy`/
`remove_variable_legacy` are emitted and handled (14+ call sites), and it carries
real reject/repair mechanics matching the Redirector's original design
responsibilities (`raw_key` repair mode gated behind explicit `force_action`
confirmation, drawer-list reject filters, mount-traversal validation). §10.5
below covers this mapping in full.

---

## **10.4 The Event Path: How a Packet Moves Through Friday’s House**

To understand all reasoning and memory flows, we trace the path of a packet:

```
Perception → Routing & Sanity → Abbot Scheduling → Summarizer → Cabinet → Identity → Reasoning → Actuation → Final Event
```

Each step maps directly onto a concrete HA mechanism.

---

## **10.5 Routing Layer: Zen DojoTools Redirector**

*(Updated 2026-08-04: the Redirector, as originally speced in this chapter, shipped
— but as the Cabinet package, not a standalone automation. This section now
describes the shipped implementation.)*

The Redirector is the heart of Friday’s sanity layer. It shipped as the **Cabinet
package**: `zen_dojotools_filecabinet` (the MCP-facing surface) backed by
`zen_sutra_filecabinet` (the internal terminus that does the actual work), both
in `packages/zenos_ai/dojotools/dojotools_filecabinet.yaml`.

Of the originally-speced responsibilities, verified present in the shipped Cabinet package:

1. **Validate legacy events** — `set_variable_legacy`/`remove_variable_legacy` are emitted and handled directly in `dojotools_filecabinet.yaml` (14+ call sites).
2. **Reject malformed data** — confirmed reject/guard mechanics: `raw_key` repair mode gated behind explicit `force_action` confirmation (a deliberate "I hope you knew what you were doing" warning path), drawer-list `reject()` filters removing bad entries, mount-traversal validation on write.
3. **Offer repairs for unknown volumes** — the `raw_key=true` repair path exists specifically for this: recovering keys written outside normal FileCabinet flow (e.g. raw events with characters that break slugification) that would otherwise be unreachable.

Two of the originally-speced responsibilities I could not independently confirm
as implemented in the Cabinet package and did not want to assert without
verification — flagging rather than guessing:

* **"Map event prefixes to Cabinets"** and **"Enforce ACL boundaries"** as
  distinct routing/enforcement mechanisms (ACL enforcement generally is a
  design target not yet active anywhere in the codebase today — see §15.6).
* **"Emit debug events"** and **"avoid double-writes or recursive triggers"** as
  dedicated Cabinet-package mechanisms — no distinct implementation found for
  either under this name; if these exist under different terminology, that's a
  gap in this doc's mapping worth closing, not a claim to make blind.

Without the Cabinet package, Friday would have no way to prevent corrupted drawers or malformed packets.

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

*(Updated 2026-08-04: corrected script name — see §10.5 for the Redirector/Cabinet-package mapping.)*

All memory mutation occurs via the Cabinet package:

```
script.zen_dojotools_filecabinet   (MCP-facing surface)
script.zen_sutra_filecabinet        (internal terminus)
```

`script.zen_dojotools_write_drawer` (the name previously documented here) does
not exist in the codebase.

Verified enforcement present in the shipped Cabinet package:

* drawer existence / mount-traversal validation on write
* reject/repair mechanics for malformed or unreachable keys (`raw_key` repair path, gated behind explicit `force_action` confirmation)

**Not verified as currently active** (flagging rather than asserting): GUID
validation, ACL checks, schema conformity, and type safety as dedicated
per-write enforcement steps — I did not find code confirming these run on every
write. ACL enforcement specifically is a known design target, not yet active
anywhere in the codebase today (see §15.6).

Cabinet writes are not tagged with a dedicated `cabinet_write` event kind today —
see §10.3.2 for the actual event vocabulary in use (`kata_emit`,
`cabinet_mounted`/`cabinet_dismounted`, etc.).

"Rely on Abbot authorization (v1)" and "will require session tokens (v1.5)" are
both roadmap statements, not current behavior — Abbot-level ACL enforcement is
not implemented yet (see §15.6, §10.8).

Writes never occur directly from automation logic — every real mutation flows
through the Cabinet package's own script, which remains true regardless of which
of the above enforcement steps are or aren't active yet.

---

## **10.8 Scheduling: The Abbot**

"The Abbot" is the deterministic cognitive scheduler — realized today as the Scheduler and Dispatcher working together (see §10.8.3).

**Note:** this section describes a future version of the Zen Scheduler, for architecture direction purposes, not current behavior. Future versions of tools will all include event consumption and emission mechanisms, combined with session token validation, so tools and security context can be tied to a single conversation context. The Abbot will also become the single target for inference job requests: a structured, intelligent queue for all incoming inference/cognitive job types, filtering by user preferences, cost, permission model, power, and availability. Tool communication is moving toward fully event-driven — where a tool exposes an event pipeline, prefer it. The Abbot becomes the main event relay: the listener for the OS core, responsible for driving the inference pipeline.

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

*(Updated 2026-08-04: "the Abbot" is not one script — it's a name for the
scheduling+dispatch function realized jointly by two real, separate components.)*

The Abbot is implemented as:

* **`dojotools_scheduler.yaml`** (`zen_dojotools_scheduler`) — trigger dispatch, debounce/shed logic via `pipeline_tier` and queue-depth thresholds (`shed_ambient_at`/`shed_keeper_at` from `zen_scheduler_config`), serialization guarantees.
* **`dojotools_dispatcher.yaml`** (`zen_dojotool_dispatcher`) — event-driven routing: listens for `zen_event(kind: dojotool_call)`, routes to the registered script, fires a correlated `zen_event(kind: dojotool_return)`. Decouples callers from direct script calls (an unknown tool returns a structured error on the bus instead of hard-faulting the calling sequence) and carries an `acl` field through its payload — plumbed, not yet enforced (see §15.6).
* triggered by real event kinds — `summary_force`, `ninja_force`, `supersummary_force`, etc. (not `zen_event.summary_request`, which doesn't exist — see §10.3.2)
* returning structured status back to Cabinet drawers

The Abbot creates *deterministic cognition* atop HA’s synchronous event model.

Stated future direction (not yet built): job tagging and evaluation so a
request can be routed to the correct inference core based on its content —
e.g. keeping sensitive content on a local model, routing code review to a
different provider — with the Abbot as the single listener/dispatch point for
all inference job requests platform-wide. §10.8's introductory note already
describes this target; this subsection just corrects which real files
implement the *current* (non-routing) scheduling/dispatch behavior.

---

## **10.9 Summarizers and Cognitive Work Units**

Each summarizer is a functional module that:

1. reads from RoomState, cabinet state, or sensors
2. constructs a structured JSON object
3. emits a `zen_event` with `kind: kata_emit` (see §10.3.2 for the real event vocabulary)
4. returns a packet that the Cabinet package writes into a drawer

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

Each emits a `zen_event` with a failure-specific `kind` (e.g. `ninja_failure`, `monk_failure` — see §10.3.2), containing:

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

* `event_type: zen_event` searches, filtered on `event.data.event.kind`
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
