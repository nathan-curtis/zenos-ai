# ZenOS Knowledge Enrichment Layer — Frozen Contracts

**Version:** 0.2.0  
**Status:** BLESSED — do not change without version bump and migration note  
**Owner:** Zen DojoTools Library  
**Companion:** `lenses.md` (provider admission rules, registry, event contract)

---

## LensAnchor

Typed reference handed by a consumer to the dispatcher.

```json
{
  "type": "label",
  "id": "medical",
  "label": "Medical",
  "source": "ha_label_registry",
  "confidence": "explicit"
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `type` | yes | `label` `person` `area_id` `zone` `product` `ticket` `document` `company` `concept` `household` |
| `id` | yes | Stable ID within that type's namespace |
| `label` | no | Human-readable display name. Never used for routing. |
| `source` | no | Who produced this anchor: `ha_label_registry`, `ha_person_entity`, `ha_area_registry`, `grocy`, `radar`, `rolodex`, etc. |
| `confidence` | no | `explicit` (directly observed) or `inferred` (derived). Default `explicit`. |

Rules:
- `type` is the namespace authority. A label named `office` is `type: label`, never `type: area_id`.
- Never promote or coerce anchor types.
- `id` must be stable across renames. Use registry IDs, not display strings, for non-label types.
- Consumers build anchors from local authoritative context only. Do not guess.

---

## EvidenceItem

A single piece of evidence returned by a provider for one or more anchors.

```json
{
  "stack": {
    "provider": "paperless_ngx",
    "object_type": "document",
    "source_id": 12,
    "evidence_id": "paperless_ngx:document:12",
    "lens_mode": "stacks_document_lens"
  },
  "anchor_contexts": [
    {
      "type": "label",
      "id": "medical",
      "confidence": "explicit",
      "source": "paperless_tag",
      "provider": "paperless_ngx"
    }
  ],
  "summary": "1099-SA HSA distribution — PNC Bank, 2024.",
  "content_redacted": true
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `stack.provider` | yes | Stable provider key from registry |
| `stack.object_type` | yes | `document` `article` `ticket` `product` `contact` etc. |
| `stack.source_id` | yes | Integer or string ID in the provider's system |
| `stack.evidence_id` | yes | Globally stable: `{provider}:{object_type}:{source_id}` |
| `stack.lens_mode` | no | Provider mode used to produce this item |
| `anchor_contexts` | yes | Which anchors matched, how, and with what confidence |
| `summary` | no | One-sentence human-readable description. No PII. No raw content. |
| `content_redacted` | yes | Always `true` for lens output. Raw content requires separate explicit read mode. |

Rules:
- `evidence_id` format is `{provider}:{object_type}:{source_id}` — no exceptions.
- `content_redacted: true` is mandatory. Never return raw OCR, full note bodies, full ticket bodies, or PII.
- `summary` is optional but strongly recommended. Max ~120 chars.
- `anchor_contexts[]` must include every anchor that matched this item, with `confidence`.
- Deduplicate by `evidence_id` within a provider response.

---

## LensEnvelope

What `zen_dojotools_lens_dispatch` returns to the consumer.

```json
{
  "status": "success",
  "tool": "Zen DojoTools Lens Dispatch",
  "consumer": "inspect",
  "anchors": [
    {"type": "label", "id": "medical"},
    {"type": "person", "id": "person.member_name"}
  ],
  "evidence": [
    {
      "stack": {"provider": "paperless_ngx", "object_type": "document", "source_id": 12, "evidence_id": "paperless_ngx:document:12", "lens_mode": "stacks_document_lens"},
      "anchor_contexts": [{"type": "label", "id": "medical", "confidence": "explicit", "source": "paperless_tag", "provider": "paperless_ngx"}],
      "content_redacted": true
    }
  ],
  "evidence_count": 1,
  "providers": {
    "paperless_ngx": {
      "status": "success",
      "department": "stacks",
      "consumes": ["label", "person", "area_id", "zone"],
      "returns": ["document_evidence"],
      "count": 1,
      "documents": []
    },
    "wiki_js": {
      "status": "unavailable",
      "count": 0,
      "miss_reason": "provider_offline"
    }
  },
  "content_redacted": true,
  "caller_token": "inspect-lenses"
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `status` | yes | `success` — even if all providers returned empty. Only `error` if dispatcher itself failed. |
| `tool` | yes | Always `"Zen DojoTools Lens Dispatch"` |
| `consumer` | yes | Who called the dispatcher |
| `anchors` | yes | Typed anchors passed in |
| `evidence` | yes | Merged, deduplicated `EvidenceItem[]` from all providers |
| `evidence_count` | yes | `evidence \| length` |
| `providers.{key}.status` | yes | `success` `empty` `unavailable` `disabled` `error` |
| `providers.{key}.count` | yes | Items returned from this provider |
| `providers.{key}.miss_reason` | no | Why nothing matched: `no_match` `provider_offline` `not_configured` `anchor_type_unsupported` `provider_disabled` |
| `providers.{key}.documents` | no | Per-provider evidence list — provider-specific detail, mirrors items in `evidence[]` |
| `providers.{key}.department` | no | Provider department from registry |
| `providers.{key}.consumes` | no | Anchor types this provider accepts |
| `providers.{key}.returns` | no | Evidence classes this provider returns |
| `content_redacted` | yes | Always `true` |
| `caller_token` | no | Echoed from input |

Rules:
- Top-level `status` is `success` when the dispatcher ran, even if every provider was empty or unavailable.
- A dead provider populates `providers.{key}.status: unavailable` — it does not surface as a top-level error.
- `evidence[]` is the merged flat list. Per-provider `documents` is extra detail — same items, scoped to that provider.
- `disabled` status means `enabled: false` in registry. `unavailable` means `status != available`.
- Consumers MUST NOT fail their primary workflow when `evidence_count: 0` or a provider is unavailable.
- Dispatcher must cap total evidence items. Default page_size: 50 per provider.
- Consumers MUST NOT fail their primary workflow when `evidence_count: 0` or a provider is unavailable.
- Dispatcher must cap total evidence items. Default page_size: 50 per provider.

---

## RefReceipt

Result shape for fire-and-forget inventory reference hooks. Embedded in the calling tool's result, not returned standalone.

```json
{
  "inventory_notified": true,
  "ref_type": "ticket",
  "ref_id": 6,
  "status": "ok",
  "already_present": false
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `inventory_notified` | yes | `true` if hook fired and Grocy confirmed OK. `false` on any error or skip. |
| `ref_type` | no | `ticket` or `doc` — present when notified |
| `ref_id` | no | Integer ID that was registered — present when notified |
| `status` | no | `ok` `warn` `skip` — present when notified |
| `already_present` | no | `true` if ref was already in the list (dedup held) |

Rules:
- `inventory_notified: false` is never a fatal error to the caller. The ticket/note was still created.
- Hook fires with `continue_on_error: true`. Caller never waits.
- `product_name` field drives the hook. Empty `product_name` = hook skips silently.
- Hook only fires on confirmed success of the primary operation (ticket created, note posted, document patched).

---

## DispatchTrace

Observability payload. Emitted as `lens.lookup_completed` event by the dispatcher on every call.

```json
{
  "consumer": "inspect",
  "caller_token": "inspect-abc123",
  "anchors": [{"type": "label", "id": "medical"}],
  "anchor_count": 1,
  "anchor_types": ["label"],
  "providers_attempted": ["paperless_ngx", "wiki_js"],
  "providers_succeeded": ["paperless_ngx"],
  "providers_failed": [],
  "providers_unavailable": ["wiki_js"],
  "evidence_count": 1,
  "total_ms": 215,
  "content_redacted": true,
  "status": "success"
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `consumer` | yes | Calling tool |
| `caller_token` | yes | Passed through for correlation |
| `anchors` | yes | Input anchors |
| `anchor_types` | yes | Unique types in the anchor list |
| `providers_attempted` | yes | All providers consulted |
| `providers_succeeded` | yes | Returned ≥0 results without error |
| `providers_failed` | yes | Returned an error |
| `providers_unavailable` | yes | Skipped — registry status unavailable/disabled |
| `evidence_count` | yes | Total merged evidence items returned |
| `total_ms` | no | Wall clock from dispatch start to envelope ready |
| `content_redacted` | yes | Always `true` |

Rules:
- Emitted as HA event `lens.lookup_completed` via `event:` action step.
- `continue_on_error: true` — trace emission never breaks the dispatch result.
- Never include raw content, OCR, PII, or full ticket/note bodies in trace.
- `caller_token` threads through from consumer → dispatcher → providers → trace.

---

## Regression Suite — v0.2.0

Five cases that must pass before any new provider is registered.

| ID | Name | What it proves |
|----|------|---------------|
| R1 | missing-product | `product_ref_add item="not a real product"` → `status: error, product_id: null` — resolver fails gracefully, no panic |
| R2 | duplicate-ref | Same `product_ref_add` twice → second call: `already_present: true, status: ok` — dedup holds |
| R3 | dead-provider | Dispatcher with a registry entry marked unavailable → `LensEnvelope.providers.{key}.status: unavailable`, top-level `status: success` |
| R4 | bad-customer | `ticket_create customer="nobody@nowhere.invalid"` → Zammad 422 surfaced cleanly, tool returns `status: error`, no silent swallow |
| R5 | stale-doc-ref | `product_refs_get` for product with doc_ref pointing to deleted document → returns ref IDs without fetching doc content; IDs are opaque pointers, not validated |

R1, R2, R4, R5: runnable against live tools today.  
R3: requires dispatcher to exist. Gate: must pass before Wiki.js registers.
