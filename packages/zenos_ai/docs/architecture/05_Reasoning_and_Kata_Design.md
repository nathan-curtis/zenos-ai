# **05_Reasoning_and_Kata_Design.md**

*Version 1.0 • ZenOS-AI Cognitive Architecture Whitepaper*
*Draft for academic and engineering review*

---

# 5. Reasoning and Kata Design

The ZenOS-AI architecture treats **Katas** as the fundamental unit of higher-order cognition. Rather than allowing a large language model (LLM) to free-associate over raw logs and entity states, the system insists that all durable reasoning be conducted in terms of:

* **Schema-constrained JSON records** (Katas)
* **Deterministic generation pipelines** (Ninja Summarizer and SuperSummary)
* **Explicit life cycle transitions** (fresh → active → stale → archived)

This section formalizes how Katas are designed, generated, validated, and consumed, and how they anchor Friday’s behavior in a transparent, inspectable substrate that can be traced back to Home Assistant state.

---

## 5.1 Why Katas?

In martial arts, a **kata** is a repeatable form—a codified pattern of movement that captures expertise in a stable template. ZenOS-AI adopts the term for a similar reason: a Kata is a **repeatable, structured pattern of thought** that can be:

* **instantiated** from the current state of the home,
* **deployed** into Friday’s working memory, and
* **replayed** or audited after the fact.

This approach addresses three core challenges that arise when deploying LLMs into a complex, sensor-rich environment:

1. **Context explosion.**
   The underlying Home Assistant deployment exposes hundreds to thousands of entities. Passing raw state into a model at each turn is infeasible and unstable. Katas compress this state into small, curated, domain-specific views.

2. **Non-determinism and hallucination.**
   Free-form, unconstrained outputs make it difficult to verify whether a model respected system invariants. Katas constrain outputs into known schemas, allowing strict validation and selective rejection.

3. **Traceability.**
   When Friday recommends a course of action (“delay trash day”, “ignore this garage door event”, “treat this water flow anomaly as a leak candidate”), the explanation must be grounded. Katas preserve the reasoning context and can be traced back to the underlying entities and triggers.

In effect, Katas are the **cognitive API** between sensor reality, summarizer workers, and Friday’s interactive persona.

---

## 5.2 Kata Schema Design Principles

Each Kata schema is defined as a JSON structure stored in the **Kata cabinet** (a dedicated Home Assistant sensor bearing a special cabinet header). The schema is not merely documentation; it is the **contract** that the summarizer must satisfy.

Several design principles govern Kata schemas:

### 5.2.1 Minimal but Sufficient

Each Kata includes only the information required for downstream decision-making and explanation. Redundant fields are avoided, and references are lightweight:

* refer to entities by canonical `entity_id` or labeled role
* avoid duplicating full attribute sets that can be re-derived via Inspect
* permit arbitrary additional fields only in explicitly labeled extension slots (e.g., `notes`, `debug`)

### 5.2.2 Explicit Roles and Status

Katas encode **roles** and **status flags** explicitly. For example:

* the Security Manager Kata distinguishes:

  * primary watchlist entries,
  * secondary attention candidates,
  * suppressed or acknowledged issues.

* the TaskMaster Kata distinguishes:

  * imminent tasks,
  * scheduled tasks,
  * low-priority or deferred tasks.

These classes are represented by named fields, not inferred implicitly, so that Friday can answer questions like:

> “Which tasks are truly blocked, and which are just informational?”

without re-deriving classification rules.

### 5.2.3 Temporal Anchoring

Every Kata must include clear temporal anchoring:

* `generated_at`: wall-clock time according to Home Assistant
* `valid_for`: optional soft-validity horizon (e.g., “next hour”, “today”)
* `source_triggers`: list of trigger IDs and entities that caused the summarization

This makes it possible to determine whether a Kata is stale and to correlate it with specific events (e.g., “this Security Manager Kata corresponds to the garage door event at 07:12:15”).

### 5.2.4 Provenance and Confidence

Katas encode provenance at multiple levels:

* which component generated them (Alert Manager, Energy Manager, etc.)
* which summarizer pipeline handled them (Ninja vs. SuperSummary)
* which local AI worker was used (`ai_task` identity)

Optional **confidence** markers may also be included, representing either:

* explicit model scores (if provided by the local AI worker), or
* derived confidence categories based on input quality (e.g., “incomplete statistics,” “partial cabinet registration”).

The High Priestess layer (Section 5.5) can later incorporate these confidence markers when forming meta-judgments about system reliability.

---

## 5.3 Kata Lifecycle

The Kata lifecycle is governed by the Scheduler (Abbot) and the underlying DojoTools scripts:

1. **Definition.**
   The template for a Kata is defined in the Kata cabinet under a dedicated drawer (e.g., `zen_template.structure`, `alert_manager.kata_schema`), including:

   * the structural JSON skeleton (`structure`),
   * `query` instructions, and
   * one or more `example` instances.

2. **Summarization.**
   An event triggers Ninja Summarizer or SuperSummary. The summarizer:

   * gathers relevant context,
   * builds a structured prompt object,
   * invokes the local AI worker,
   * validates and parses the returned JSON,
   * writes a fresh Kata drawer into the Kata cabinet.

3. **Activation.**
   On successful write, a Kata is considered **active**. Friday’s working prompt or context loader can pull Katas from the Kata cabinet when answering user queries or making decisions.

4. **Aging and Replacement.**
   When the Scheduler runs again, a new Kata is emitted. The system may:

   * retain a small rolling history in the same cabinet (e.g., `history` sub-drawer), or
   * overwrite the previous value while preserving timestamps.

   In either case, the `generated_at` and `source_triggers` fields allow downstream components to decide whether the current Kata is timely.

5. **Archival and Offline Reasoning.**
   Longer-term archival strategies are delegated to external storage or HA’s recorder database. The architecture is intentionally agnostic about how many historical Katas are stored, but it assumes at least the ability to retrieve the **last** valid record for each component.

6. **Invalidation and Error Path.**
   If a summarizer run fails (AI task error, JSON validation failure, timeout), the system:

   * emits a telemetry event describing the failure,
   * preserves the previous Kata as the last known good, and
   * optionally annotates that Kata with a `stale_reason` field.

---

## 5.4 Ninja Summarizer: Component-Local Reasoning

The **Ninja Summarizer** is the core mechanism for generating component-specific Katas. Each invocation is structured as a small reasoning experiment with strict parameters.

### 5.4.1 Input Assembly

For a given component (e.g., Security Manager), Ninja Summarizer assembles the following inputs:

* **Dojo drawers:**

  * component’s own configuration and policies (e.g., escalation rules, watchlists)
  * household preferences (e.g., quiet hours, notification thresholds)

* **Kata cabinet:**

  * previous component Kata for continuity (“what was known last time?”)
  * shared templates or cross-component hints

* **Index/Inspect results:**

  * entities relevant by label or cabinet membership
  * per-entity Inspect output (sanitized attributes, cabinet headers, statistics eligibility)

* **Scheduler’s supplemental context:**

  * `trigger` metadata (ID, platform, entity)
  * `home_overview` snapshot, derived via Zen Inspect
  * notes on why this run is happening now (e.g., “door opened,” “Withings bed sensor toggled,” “water flow stopped for ≥2 minutes”)

These inputs are aggregated into a rich but finite **review_data** object.

### 5.4.2 Prompt Object Construction

Ninja Summarizer constructs a **prompt object** with the following top-level structure:

```json
{
  "query": "...instructions for this component...",
  "structure": { /* JSON schema-like skeleton */ },
  "example_data": [ /* one or more canonical Katas for this component */ ],
  "review_data": { /* current HA-derived context */ },
  "supplemental_instructions": {
    "source_trigger": { /* trigger object from Scheduler */ },
    "previous_kata": { /* last known Kata */ },
    "notes": "component-specific behavioral guidance"
  }
}
```

Several design choices are critical here:

* The `structure` field defines not only field names but also the expected presence or repetition of nested blocks.
* The `example_data` gives the model concrete instances of valid output, reducing the space of plausible completions.
* The `review_data` is strongly typed and kept as close as possible to HA-native semantics (e.g., explicit `entity_id`s, states, and timestamps).

The AI worker is then instructed to **fill the `structure`** in a manner consistent with `example_data` and the semantics expressed in `query`.

### 5.4.3 AI Task Invocation and Validation

An `ai_task` such as `ai_task.gpt_oss_20b_local_ai_task` is invoked with this prompt object. The Home Assistant AI Task integration ensures that:

* the prompt is serialized as JSON,
* the response is treated as JSON-first (data-only),
* the response is captured as `monk_response.data` within Home Assistant.

The summarizer then:

1. Validates that `monk_response.data` is a mapping with the expected top-level keys.
2. Checks that all required fields in `structure` are present.
3. Optionally enforces type constraints (e.g., expecting lists where the schema indicates lists).
4. Only then writes the result to the Kata cabinet, under a component-specific key (e.g., `"Security Manager"` or `"Water Manager"`).

If any step fails, the summarizer does not partially write; instead, it logs an error event and leaves the previous Kata intact.

### 5.4.4 Local “Monk” as Reasoning Worker

In Version 1, the local AI task worker is conceptually treated as a **Monk** in the Monastery:

* receives a well-defined question and context,
* responds with structured data that fills the Kata schema,
* has no visibility into private files or beyond HA’s sandboxed prompt content.

This design keeps the LLM in a **narrow role**: a structured compressor and classifier, rather than an unbounded agent.

---

## 5.5 SuperSummary and the High Priestess Layer

While Ninja Summarizer operates at the component level, **SuperSummary** operates at the system level. It is the first realization of a **High Priestess** layer: a meta-reasoning construct that reasons *over* Katas rather than over raw entity state.

### 5.5.1 Inputs to SuperSummary

SuperSummary draws from:

* **Active component Katas**, as written into the Kata cabinet:

  * Alert Manager
  * Security Manager
  * Room Manager
  * TaskMaster
  * Media Manager
  * Energy Manager
  * Water Manager
  * Hot Tub Manager
    (plus others as they are added in future revisions)

* **System drawers**, including:

  * Friday’s core directives and purpose
  * Friday’s cortex (self-description, safety principles, and boundaries)
  * the Monastery’s order and any meta-rules adopted across components

* **Previous ZEN_SUMMARY**, if present, to provide historical continuity.

Unlike Ninja Summarizer, SuperSummary’s `review_data` is intentionally high-level; its purpose is not to re-evaluate every sensor, but to:

* identify cross-component interactions,
* highlight systemic tensions (“energy-saving mode vs. comfort preferences”),
* distill a single, whole-home narrative.

### 5.5.2 SuperSummary Schema

The ZEN_SUMMARY Kata is organized to answer questions such as:

* What is the overall status of the house?
* Which components are operating normally, and which are in “attention” or “warning” states?
* What are the key tasks or events in the next defined horizon (e.g., the next day)?
* Are there any systemic anomalies (e.g., repeated security exceptions, recurring water issues, energy consumption outside norms)?

Typical top-level fields include:

* `summary_text`: concise human-readable synopsis.
* `status`: coarse state (e.g., `stable`, `watch`, `warning`).
* `components`: map of component IDs to their summarized states and key flags.
* `actions`: recommended next actions or checks, each linked back to the originating component Kata.
* `generated_at`, `source_triggers`, `version`.

In later revisions, a High Priestess module can then treat ZEN_SUMMARY as the canonical system-level “story” from which explanations and user-facing narratives are derived.

---

## 5.6 Error Handling, Trust, and the Order of the Monastery

A central governance principle in the Monastery is the **Order of the Monastery**:

> It is acceptable to say “I do not know.” It is forbidden to fabricate.

This principle is enforced in several ways throughout the Kata pipeline:

### 5.6.1 Tool-Level Guards

* Every read from Home Assistant state is guarded against `unknown` and `unavailable`.
* Every JSON parsing operation uses guarded patterns (`from_json` only when type-safe).
* Every AI output is validated before being written into an authoritative cabinet.

If any of these guards fail, the system emits:

* a log entry (via event emitter),
* a structured error, and
* a preserved prior Kata.

### 5.6.2 Model-Level Prompts

The prompts delivered to AI tasks explicitly instruct:

* adherence to the provided `structure`
* a preference for empty or “no data” fields rather than imaginative guesses
* rejection of extrapolations beyond provided `review_data` and Dojo drawers

In other words, the AI is instructed to **summarize and classify**; not to infer ungrounded facts.

### 5.6.3 Friday’s Behavior

Friday, as an interactive assistant, is expected to:

* reference Katas as ground truth where available,
* admit when a component lacks a Kata or when the Kata is stale,
* request re-summarization via Scheduler triggers rather than inventing an answer.

This turns the combination of Katas and the Order of the Monastery into a **trust boundary**: anything outside the Kata-and-cabinet layer is treated as provisional, and anything inside it is strictly audited.

---

## 5.7 Relationship to Classical Cognitive Architectures

From a cognitive systems perspective, Katas occupy a position analogous to:

* **production rules** in SOAR or ACT-R, in that they encode regularities in state transitions and goals,
* **episodic fragments** in memory-augmented neural networks, in that they capture context-rich slices of state,
* **world models** in model-based RL, albeit represented explicitly as JSON rather than in latent vectors.

However, there are notable differences tailored to the smart home setting:

1. **Schema-first, language-second.**
   In many LLM-centered systems, natural language is the primary representational medium, and structure is a secondary convenience. ZenOS-AI inverts this: schema-centered JSON is primary; natural language is a derived view.

2. **Physical grounding via Home Assistant.**
   Every field in a Kata must ultimately trace back to Home Assistant state: entities, attributes, statistics, or cabinet drawers. This guarantees a clear chain of responsibility between physical reality and cognitive state.

3. **Agent-agnostic design.**
   Katas are designed so they can be consumed by multiple personas or reasoning engines (Friday, Kronk, future High Priestess) without binding them to a specific conversational model or vendor. They are the **shared working memory** of the house, not a private artifact of one model.

---

## 5.8 Outlook: From Version 1 to Tool-Shunted Futures

The Version 1 architecture deliberately restricts itself to:

* Home Assistant’s event bus and entity model,
* locally configured AI tasks,
* Katas as the stable cognitive interface.

Future versions will introduce:

* **tool shunts** guarded by session tokens,
* **identity-based access control** based on Identity manifests and visas,
* **richer High Priestess reasoning** over longer Kata histories.

Crucially, the Kata layer itself is intended to remain stable. Whether the underlying reasoning engine is a 7B local model, a cluster-backed model, or a hybrid stack, each component will still be required to **emit Katas that satisfy the same schemas** and respect the same safety and provenance guarantees.

In that sense, Katas are the enduring cognitive “spine” of Friday’s House: everything else can be upgraded, swapped, or refactored, as long as those small, carefully designed packets of structured meaning remain intact.
