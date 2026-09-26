# 14. Katas

A Kata is a summary in a fixed shape. The name comes from martial arts: a kata is a codified form, the same pattern repeated until it is reliable. In ZenOS a Kata is the codified form a domain's state is reduced to, so every consumer can read every domain the same way.

There are two kinds: the component Kata each Ninja run produces, and `zen_summary`, the whole-house Kata SuperSummary produces.

## 14.1 The component Kata

Every component fills the same schema, stored as `kata_template` in the Kata cabinet:

| Field | Meaning |
|---|---|
| `subjective` | How the situation reads, in a sentence |
| `general_inference` | What the data implies |
| `attention` | What deserves attention, if anything |
| `objective` | What should happen |
| `events` | Timestamped notable events |
| `watchlist` | Things to keep an eye on |
| `error` | A problem producing the summary, if any |
| `urgency` | 0 to 10 |
| `confidence` | 0.0 to 1.0 |
| `action_required` | Whether something needs to be done |
| `suggested_act_desc`, `suggested_act_event` | What, and which tool call would do it (Chapter 13) |
| `period` | The period the Kata covers |
| `last_run_at` | When a run last wrote this Kata |
| `last_emission_at` | When this component last escalated |

One shape for every domain is the point. SuperSummary, the prompt, and any tool can read the security Kata and the pantry Kata the same way, and compare their urgency directly.

`last_run_at` and `last_emission_at` answer different questions. `last_emission_at` only changes when urgency crosses the escalation threshold, so a quiet, healthy component can go a long time without one. `last_run_at` is stamped on every successful write, so "did this actually run?" always has an answer.

## 14.2 zen_summary

SuperSummary fills `zen_template`:

```text
system:            cortex, os_version, build_version
home_overview:     status, risks[], events[]
active_components: { component_id: its summary and attention }
zen_summary:       summary, attention, objective, subjective,
                   urgency, confidence, criticality,
                   emotion_queue[], errors[],
                   timestamp, stale_after
```

`home_overview` is the coarse situation: overall status, active risks, recent notable events. `active_components` is the per-domain view of whatever SuperSummary decided to include (Chapter 13). The `zen_summary` block is the synthesis: what matters, what should happen, how urgent it is, and how sure the system is.

## 14.3 The write guard

A Kata is only written if the model's answer is real. The summarizer extracts the JSON object from the response by its first `{` and last `}`, which survives code fences and prose the model adds around it, and validates it with `from_json`. If the result is not a real, non-empty object, the failure is logged and nothing is written. The previous Kata stays in place.

This matters because the failure it prevents is silent. A write that stamps a fresh timestamp on an empty drawer looks healthy to every consumer: the Kata is recent, it just says nothing. Guarding on the parsed result instead of on "the model returned some text" keeps a bad run from impersonating a good one. SuperSummary applies the same guard to `zen_summary`.

## 14.4 Freshness

Consumers decide whether a Kata is current from its own fields: `last_run_at` on a component Kata, `timestamp` and `stale_after` on `zen_summary`. Each KFC declares a `staleness_minutes` ceiling. A component whose Kata passes that ceiling is dispatched for a refresh even when the scheduler would otherwise shed it under load (Chapter 21).

## 14.5 Acknowledgements

A household can tell the system "we know about this, it's fine" with `zen_dojotools_alertmanager mode=ack`. An acknowledgement is keyed by a `condition_key`, expires after `ttl_days` (default 7), and can carry a baseline and a note. `check_ack` reads it and `revoke_ack` removes it.

For KFC-summarized components, the summarizer checks for an acknowledgement whose `condition_key` equals the component's `kata_key` before escalating. If one exists, the escalation is suppressed and an `emission_suppressed` event records why. That suppression is whole-component and coarse: a genuinely new problem inside an acknowledged component is also held back until the acknowledgement expires or is revoked. The expiry is the backstop.

AlertManager stores the acknowledgement. It does not judge whether a condition has materially changed since, and cannot: it has no way to evaluate a stored rule at runtime. A domain tool that wants that finer judgment compares current values against the stored baseline in its own logic and revokes the acknowledgement itself.

<!-- nav -->
---

[← The Monastery](13_the_monastery.md) · [Contents](00_toc.md) · [Live State →](15_live_state.md)
<!-- /nav -->
