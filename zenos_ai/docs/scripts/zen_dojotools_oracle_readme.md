# Zen DojoTools Oracle — v0.2.1

> **2026.12.0 (0.1.0 to 0.2.1), prototype, off by default:** Oracle is the decision service (KF6.5). It asks a small, typed question through Home Assistant's `ai_task.evaluate` and returns a typed answer. It judges and never acts. Nothing turns it on but the operator's switch, and nothing in the background pipeline calls it yet: in this release an agent asks it, interactively. Room Manager is the first host.

**File:** `packages/zenos_ai/dojotools/dojotools_oracle.yaml`
**Script:** `zen_dojotools_oracle` (parallel, max 10)
**Switch:** `input_boolean.zen_decision_enabled` (operator only, off on a new install)
**Needs:** Home Assistant 2026.11 or later, and an `ai_task` entity that supports evaluation (`AITaskEntityFeature.EVALUATE`).

---

## Overview

A background summary is expensive: the Ninja runs the component's seed and then a full model call, every time a trigger fires, whether anything changed or not. KF6.5 adds a cheaper step that can come first: a bounded question, *does this change matter enough to summarize?* Oracle is where that question is asked.

- **The host** (a component's tool, for example Room Manager) owns the question. It declares its decision templates: the questions, the policy thresholds and what a full summary would refresh. It also binds its own evidence.
- **Oracle** checks the gates, checks the size, calls `ai_task.evaluate` once, validates every answer and applies the host's policy. It returns a typed decision.
- **The consumer** (an agent now, the Abbot later) decides what to do with it. A decision never authorizes an action, and it never requests or forces a summary by itself.

Evaluation is cheap. On the reference install one question over about 500 to 700 tokens of evidence took about 0.7 s. Most of a decision's time is the host's own seed, which a full summary would run anyway.

Evaluation and inference are separate branches. Oracle depends on none of the summarizer switches or certs, and holding a decision cert grants no inference.

---

## Gates

Every gate must pass, or nothing is sent to a model.

| Gate | What it checks | Who controls it |
|---|---|---|
| Decision switch | `input_boolean.zen_decision_enabled` is on | The operator only (below) |
| `zenos.decision.use` | The agent may use decisions: level 1 for `decide`, level 2 for `ask` | Cert grant |
| `zenos.decision.component.<host>` | The agent may use this host's templates (`decide` only) | Cert grant |
| Readiness | An evaluation-capable `ai_task` entity is available | Install configuration |
| The host's seed cert | The agent may read the evidence (`decide` only; the host checks it when it binds) | Cert grant |

**The switch is the operator's.** It is defined in YAML like the summarizer kill switches, carries `zen_never_expose` (Flynn applies it, so Ectoplasm refuses to expose it), and no agent tool writes it. It keeps its setting across restarts. Turning it off stops decisions at once and changes no schedule. Configuring an evaluator, granting a cert or a host publishing templates never turns decisions on.

**Certs sit on one tree,** `zenos.decision`, beside the inference tree, `zenos.inference`. Prefix inheritance applies (see [dotted certification names](../getting_started/security_certification_manual.md)): granting `zenos.decision` covers use and every host's templates, while `zenos.decision.use` plus one `zenos.decision.component.<key>` covers a single host. The checks are independent: holding use grants no host's templates, and a host cert grants no use. A host must declare its cert as `zenos.decision.component.<its own key>`; Oracle refuses any other name.

---

## Modes

| Mode | What it does |
|---|---|
| `help` (default) | The reference, with the question structure and examples. A bare call never acts. |
| `tool_manifest` | Identity, the `zenos.decision.use` cert, and the inference declaration (`ai_task_type: evaluate`). |
| `templates` | Every host's decision templates (`host=` filters). Zero inference, schedules nothing. |
| `decide` | `host=<key> decision_template=<id> [area=<area>]`: one judgment over the host's evidence. |
| `ask` | `questions={...} evidence={...} [task_name=...]`: the agent's own questions over evidence it supplies. Advisory only. |

### `decide`

1. **Switch and use cert** (`zenos.decision.use` level 1).
2. **Host and template.** The host is found through its Dojo drawer, and its templates through its `kfc_manifest` (`component.decision`). Unknown host: `host_not_found`. Unknown template: `unknown_template`.
3. **Host cert** (`zenos.decision.component.<host>`, at the level the host declares).
4. **Bind.** Oracle calls the host with `mode=decision_bind decision_template=<id> area=<area>`. The host checks the template, the scope and its seed cert, runs its seed, and returns the evidence with an `evidence_revision` and `as_of`. An area the host doesn't know is `unsupported_scope`: a decision is never widened to the whole house.
5. **Deterministic first.** If the host can settle the case without a model (Room Manager: there is no prior summary to compare with), the answer is `basis: deterministic` and nothing is sent.
6. **Size.** The questions and evidence together are estimated at characters ÷ 4. Over `max_evaluation_tokens`, nothing is sent: `evaluation_context_overflow`, with the estimate, the cap and the three largest evidence keys. It never truncates.
7. **Evaluate once.** `ai_task.evaluate` with the template's questions and the bound evidence as `state`.
8. **Validate.** Every declared question must be answered with the right type: a `noul` between 0 and 1, a `choice` from the allowed options, a numeric `score`. Anything missing or malformed is `evaluator_error`, never a keep.
9. **Policy.** The template names one `noul` question and two thresholds. At or above `refresh_at` the outcome is `refresh`; below `keep_below` it is `keep`; in between, or with no usable answer, it is `insufficient_evidence`, never a confident keep.

```yaml
decision:
  host: room_manager
  template: resummary_warranted
  template_version: 1
  outcome: refresh              # keep | refresh | insufficient_evidence
  basis: evaluated              # deterministic | evaluated
  full_summary_required: true   # only on a completed refresh
  evidence_revision: "..."
  as_of: "..."
  answers: {material_change: {noul: 0.82}}
  thresholds: {refresh_at: 0.6, keep_below: 0.35}
  tokens_est: 640
```

`full_summary_required` is set only on a completed decision. A refusal or failure is its own answer and never reads as false.

### `ask`

The agent writes its own questions over evidence it supplies. It needs `zenos.decision.use` **level 2**, plus the switch and readiness; no host is involved.

- `questions`: 1 to 5, as `{question_id: {type, instructions, criteria}}`.
  - `noul`: `{type: noul, instructions}`. The answer is `noul`, the probability of yes.
  - `choice`: `{type: choice, instructions, criteria: {option: description, ...}}`, 2 or more options. The answer is `choice` plus `probabilities`.
  - `score`: `{type: score, instructions, criteria: [lowest, ..., highest]}`, 2 or more levels. The answer is `score` plus `probabilities`.
- `evidence`: an object with only the facts needed. It is all the model sees, and it counts against the size cap.

The structure is checked before anything is sent; a bad one is `invalid_questions` with a list of `problems`. The answer is advisory: the raw `answers`, never keep or refresh. It requests nothing, acts on nothing, and never overrides a deterministic or life-safety rule.

```yaml
mode: ask
task_name: laundry_urgency
questions:
  urgency:
    type: choice
    instructions: How soon should someone move the laundry?
    criteria:
      now: it will sour or wrinkle badly soon
      today: it can wait a few hours
      whenever: no harm in leaving it
evidence: {washer: done, minutes_since_done: 95, load: towels, outdoor_temp_f: 88}
```

`mode=help` carries a `noul`, a `choice` and a `score` example.

---

## Outcomes

| Outcome | Meaning |
|---|---|
| `keep`, `refresh`, `insufficient_evidence` | A completed `decide` |
| answers, `advisory: true` | A completed `ask` |
| `decision_disabled` | The switch is off |
| `denied` | A cert gate failed; `gate` names which |
| `host_not_found`, `unknown_template` | No such host, or no such template (or a template with no questions) |
| `needs_input`, `unsupported_scope` | The host refused the binding (a missing field, an area it doesn't know) |
| `invalid_questions` | An `ask` with a bad structure; nothing was sent |
| `evaluation_context_overflow` | Over `max_evaluation_tokens`; nothing was sent |
| `evaluator_unavailable` | No evaluation-capable entity |
| `evaluator_error` | No answers (a provider error or refusal), or answers that failed validation |

Every answer comes in the standard DojoTool envelope.

---

## Configuration

An optional `zen_decision_config` drawer in the household cabinet:

| Key | Default | Purpose |
|---|---|---|
| `max_evaluation_tokens` | `6000` | The size cap, estimated as characters ÷ 4, over questions and evidence |
| `evaluate_entity_id` | (none) | Pin the evaluation entity. Without it, Home Assistant's preferred evaluation entity is used, and at least one `ai_task` entity must support evaluation |
| `thresholds` | (none) | Per-template tuning, keyed `<host>.<template>`, for example `{"room_manager.resummary_warranted": {"refresh_at": 0.7, "keep_below": 0.3}}`. A template's own policy is the default |

The entity pin is temporary. It moves to the mechanism inference uses for engine selection once that is updated.

---

## Writing a host

A host is any component tool whose `kfc_manifest` component carries a `decision` block. The host always defines its templates; a caller only picks a template and an area.

```yaml
decision:
  requires: {cert: zenos.decision.component.room_manager, level: 1}
  templates:
    - id: resummary_warranted
      version: 1
      purpose: Does the house's room picture differ enough from the last room summary that a fresh one is warranted?
      scope: [house, area]
      fields: {required: [], optional: [area]}
      summary_target: room_manager
      outcomes: [keep, refresh, insufficient_evidence]
      questions:
        material_change: {type: noul, instructions: "..."}
      policy: {question: material_change, refresh_at: 0.6, keep_below: 0.35}
```

The host also:
- answers `mode=decision` with its templates (zero inference), and advertises `decision: true` in its `tool_manifest`, with the component cert in `certs_required`;
- answers `mode=decision_bind` for Oracle: it checks the template, the scope and its seed cert, and returns `status: ok` with `evidence`, `evidence_revision` and `as_of`, or `deterministic: keep|refresh` with a `reason` when no model is needed. A refusal returns `status: error` with an `outcome` and `message`.

What `decision_bind` must honor:
- **Its inputs.** Oracle always passes `decision_template`, `area` and `caller_token`. `area` may be an empty string, which means the whole house: treat it as house scope and never fail on it.
- **The caller's identity.** The bind runs as the asking agent, so the host's seed-cert check is against that agent, not the host.
- **Read only, no inference.** The bind changes nothing and calls no model. It returns only the evidence the template needs, because that evidence counts against `max_evaluation_tokens` (Room Manager's is about 540 to 740 tokens).
- **A stable `evidence_revision`.** The host defines it, and the same evidence must give the same revision. It is what repeated decisions will be recognized by. Room Manager's is urgency, reasons, each room's occupancy, the open sensors and the area.

**The prototype's policy reads one `noul` question,** named by `policy.question`. A template may declare `choice` or `score` questions too: they are validated and returned in `answers`, but they don't drive keep or refresh yet.

A host must not call Oracle from inside itself: Oracle calls back into the host to bind, and a queued host would wait on itself. That is why Room Manager's `mode=decision` only lists templates.

### Room Manager

Room Manager 5.21.0 is the first host. See [Room Manager: decisions](../components/room_manager.md#decisions-kf65).

---

## Known limits

- **A provider's error text stays in Home Assistant's log.** A script can't capture an action's exception, so a provider failure (a wrong model name, for example) answers `evaluator_error` with "no answers", and the reason is only in the log.
- **A host's seed sets the time.** Room Manager's evidence is its full summary seed, about 35 s on the reference install. The evaluation itself takes under a second.
- **New fields reach MCP clients only after a restart, then a client reconnect.** That applies to `ask`'s fields after an upgrade.

---

## Cross-References

- [KF6 and the Ninja](zen_dojotools_summarizers_readme.md): the summary pipeline a decision would sit in front of.
- [Scheduler: requesting a run](zen_dojotools_scheduler_readme.md#requesting-a-run): how a refresh is requested (never forced).
- [Security certification manual](../getting_started/security_certification_manual.md): granting certs, levels, prefix inheritance.
