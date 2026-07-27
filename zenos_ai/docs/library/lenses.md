# ZenOS-AI Lens Provider Contract

**Version:** 0.2.0  
**Surface:** `+lenses`  
**Dispatcher:** `zen_dojotools_lens_dispatch`  
**Registry:** household cabinet drawer `lens_registry`  
**Owner:** Zen DojoTools Library

---

## What Lenses Really Are

Lenses are the admission contract for ZenOS knowledge sources.

If a tool, plugin, or external system can provide useful evidence about a person, place, thing, label, concept, product, ticket, document, room, zone, or household primitive, it can opt into the lens bus. In return, ZenOS will register it as a knowledge source and allow consumer tools to ask for its context without hard-coding that provider.

The contract is intentionally strict:

* Providers consume typed anchors, not vague strings.
* Providers emit bounded evidence, not raw corpus dumps.
* Providers register their own capabilities, health, security act, and failure policy.
* Consumers call the lens dispatcher, not individual providers.
* Missing or broken providers fail soft and do not break the primary tool.
* Mutation is not part of lens lookup. Lenses are evidence projection.

In plain terms: use our contract and we will let your tool play in the shared knowledge graph. Break the contract and the dispatcher should treat you as unavailable.

---

## System Role

The Library owns the lens bus. Provider tools own their own data.

The bus does four jobs:

1. Reads `lens_registry` from the household cabinet.
2. Accepts typed anchors from consumers.
3. Routes each anchor to registered providers that declared support for that anchor type and return class.
4. Merges the provider evidence into a stable, redacted envelope.

This keeps `Inspect`, `Service Desk`, `Inventory`, `Rolodex`, Room Manager, and future tools from needing custom knowledge-source code for Paperless, Wiki.js, Radar, Grocy, CRM, or any later stack.

---

## Definitions

| Term | Meaning |
|------|---------|
| Consumer | A tool asking for enrichment, such as Inspect or Service Desk. |
| Provider | A knowledge source registered in `lens_registry`. |
| Anchor | A typed reference to something already known, such as `person.member_name`, `area_id: office`, or `label: medical`. |
| Evidence | A redacted provider result with provenance and stable IDs. |
| Lens | The projection from anchor to evidence through a provider. |
| Registry | The household cabinet drawer that declares active providers and their capabilities. |
| Security act | Provider declaration using the `rwxda` model. |

---

## Anchor Contract

Consumers MUST send typed anchors.

```json
[
  {"type": "label", "id": "medical"},
  {"type": "person", "id": "person.member_name"},
  {"type": "area_id", "id": "office"},
  {"type": "zone", "id": "home"}
]
```

Supported and planned anchor types:

| Type | Meaning | Authority |
|------|---------|-----------|
| `label` | Home Assistant label ID | HA label registry |
| `person` | Home Assistant `person.*` entity ID | HA person entity |
| `area_id` | Home Assistant area ID | HA area registry |
| `zone` | Home Assistant zone object ID | HA `zone.*` entity |
| `concept` | Controlled concept anchor | Future concept registry |
| `company` | Company/account anchor | Rolodex/CRM authority |
| `product` | Inventory product anchor | Inventory/Grocy authority |
| `ticket` | Service ticket anchor | Service Desk/Radar authority |
| `document` | Document/evidence anchor | Stack provider authority |
| `household` | Household cabinet or GUID | Household authority |

Rules:

* `person` means a resolved `person.*` entity. A label named `nathan` is still only a label.
* `area_id` means an HA area primitive. A label named `office` is not an area.
* `zone` means an HA zone primitive, such as `home` for `zone.home`.
* `label` is valid, but it is never silently promoted into another primitive type.
* If the same text exists in multiple namespaces, the explicit `type` wins.
* Do not guess entity IDs, areas, zones, people, labels, or cabinet names.

---

## Consumer Contract

Consumers MAY opt into lenses when they already hold authoritative anchors.

Consumer responsibilities:

* Build anchors from local authoritative context.
* Call `zen_dojotools_lens_dispatch`.
* Treat missing providers as empty enrichment.
* Read `domain_context.lenses` for shared provider output.
* Read per-result `lens_context` for local matches.
* Avoid depending on provider internals unless the provider declares them in `returns`.
* Never block the primary workflow on lens enrichment.

Inspect example:

```json
{
  "output_fields": "+lenses,-device,-timestamps,-volume"
}
```

Expected per-result shape:

```json
{
  "lens_context": {
    "status": "matched",
    "evidence_count": 2,
    "anchors": [{"type": "label", "id": "medical"}],
    "evidence_ids": ["paperless_ngx:document:12"],
    "providers": ["paperless_ngx"]
  }
}
```

No matching evidence is still success:

```json
{
  "lens_context": {
    "status": "success",
    "evidence_count": 0,
    "anchors": [{"type": "zone", "id": "home"}]
  }
}
```

---

## Provider Admission Rules

A provider is eligible for registration only if it can:

* Accept typed anchor lookup using the standard input shape.
* Return the standard response envelope.
* Declare its consumed anchors and returned evidence classes.
* Declare its security act using `rwxda`.
* Provide setup/configuration status.
* Provide health and audit information.
* Provide help that an agent can read to troubleshoot the provider.
* Fail soft when unavailable.
* Keep lens lookup read-only.

Providers SHOULD be internal stack tools where possible:

* `zen_stack_paperless`
* `zen_stack_wikijs`
* `zen_stack_radar`
* `zen_stack_grocy`

Public consumer-facing tools may exist, but the lens bus should route through a provider-owned stack surface so each knowledge source can manage its own health, audit, and setup.

---

## Provider Setup Shape

Every lens provider MUST expose a setup/config mode that can be called manually during installation or upgrade.

Minimum setup actions:

| Action | Purpose |
|--------|---------|
| `inspect` | Report current configuration and registration state. |
| `register` | Write or refresh provider declaration in `lens_registry`. |
| `unregister` | Remove provider declaration. |
| `health` | Return provider health, dependency status, and last known error. |
| `audit` | Return recent provider activity and policy-relevant facts. |
| `help` | Return agent-readable usage, modes, inputs, and failure notes. |

The exact mode name may be provider-specific, but the action vocabulary should remain stable.

Example setup input:

```json
{
  "action": "register",
  "provider": "paperless_ngx",
  "enabled": true,
  "caller_token": "provider_setup"
}
```

Example setup response:

```json
{
  "status": "success",
  "provider": "paperless_ngx",
  "action": "register",
  "registry_key": "lens_registry",
  "registered": true,
  "health": {
    "status": "available",
    "configured": true,
    "last_error": null
  },
  "audit": {
    "last_registered_at": "2026-06-05T11:19:00-05:00",
    "registered_by": "paperless_ngx"
  }
}
```

---

## Registry Declaration

Lens providers register in the household cabinet drawer `lens_registry`.

The registry is maintained through `zen_admintools_prompt_loader mode=lens_registry` or a provider setup routine that calls the same governed path.

Minimum provider declaration:

```json
{
  "provider": "paperless_ngx",
  "configured_by": "paperless_ngx",
  "department": "stacks",
  "enabled": true,
  "status": "available",
  "tool": "zen_stack_paperless",
  "mode": "stacks_by_anchor",
  "consumes": ["label", "person", "area_id", "zone"],
  "returns": ["document_evidence"],
  "timeout_ms": 1200,
  "failure_policy": "soft",
  "content_policy": "redacted_by_default",
  "risk_class": "read_redacted",
  "security_act": {
    "model": "rwxda",
    "required": "r",
    "allowed": ["r"],
    "denied": ["w", "x", "d", "a"],
    "note": "Lens provider returns redacted read evidence only; CRUD modes are outside the lens bus."
  }
}
```

Required fields:

| Field | Required | Notes |
|-------|----------|-------|
| `provider` | yes | Stable provider key. |
| `configured_by` | yes | Tool/plugin that owns registration. |
| `tool` | yes | Internal callable provider tool. |
| `mode` | yes | Provider lens lookup mode. |
| `enabled` | yes | Dispatcher skips disabled providers. |
| `status` | yes | `available`, `unavailable`, `degraded`, or `disabled`. |
| `consumes` | yes | Anchor types accepted. |
| `returns` | yes | Evidence classes returned. |
| `failure_policy` | yes | Usually `soft`. |
| `content_policy` | yes | Usually `redacted_by_default`. |
| `risk_class` | yes | Usually `read_redacted`. |
| `security_act` | yes | `rwxda` declaration. |
| `timeout_ms` | no | Provider budget hint. |

Unknown provider declarations should not become executable dispatch routes.

---

## Provider Lookup Input

The dispatcher calls providers using this shape:

```json
{
  "anchor_type": "label",
  "anchor_ids": ["medical", "alex"],
  "page_size": 50,
  "include_suggestions": false,
  "consumer": "inspect",
  "caller_token": "inspect-lenses"
}
```

Provider rules:

* Accept a single `anchor_type` per call.
* Accept one or more `anchor_ids`.
* Respect explicit page size/result limits.
* Return only evidence matching the requested anchor namespace.
* Do not treat a label match as a person/area/zone match.
* Return an empty success when nothing matches.

---

## Provider Evidence Output

Evidence MUST include stable provenance and anchor context.

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
  "content_redacted": true
}
```

Provider evidence responsibilities:

* Include a stable `stack.evidence_id`.
* Include `stack.provider`.
* Include `stack.object_type`.
* Include `anchor_contexts`.
* Mark redacted output with `content_redacted: true`.
* Keep suggestions separate from explicit matches.
* Deduplicate provider-local evidence where possible.
* Do not return raw OCR, full note bodies, private ticket bodies, or full document content by default.

---

## Dispatcher Envelope

`zen_dojotools_lens_dispatch` returns:

```json
{
  "status": "success",
  "tool": "Zen DojoTools Lens Dispatch",
  "consumer": "inspect",
  "anchors": [
    {"type": "label", "id": "medical"}
  ],
  "providers": {
    "paperless_ngx": {
      "status": "success",
      "department": "stacks",
      "consumes": ["label", "person", "area_id", "zone"],
      "returns": ["document_evidence"],
      "count": 1,
      "documents": []
    }
  },
  "count": 1,
  "documents": [],
  "content_redacted": true
}
```

Dispatcher responsibilities:

* Accept native HA lists and JSON strings for anchors.
* Group anchors by type.
* Read active provider declarations from `lens_registry`.
* Call only providers whose declarations match the anchor type and return class.
* Use soft failure semantics for provider calls.
* Deduplicate evidence by provider evidence ID.
* Preserve provider-specific lists under provider keys.
* Keep a provider-neutral top-level envelope.

---

## Event Contract

Lenses are read projection, but knowledge sources still participate in the event substrate.

Providers SHOULD consume these event families when relevant:

| Event | Purpose |
|-------|---------|
| `lens_registry.changed` | Refresh local provider cache or registration status. |
| `anchor.changed` | Re-evaluate provider-local mappings for an anchor. |
| `label.changed` | Reconcile label-backed mappings. |
| `person.changed` | Reconcile person-backed mappings. |
| `area.changed` | Reconcile area-backed mappings. |
| `zone.changed` | Reconcile zone-backed mappings. |
| `provider.health_probe` | Return health without performing mutation. |
| `provider.audit_probe` | Return audit facts without performing mutation. |

Providers SHOULD emit these event families when relevant:

| Event | Purpose |
|-------|---------|
| `lens.provider_registered` | Provider registration was written or refreshed. |
| `lens.provider_unregistered` | Provider registration was removed. |
| `lens.provider_health` | Provider health changed or was probed. |
| `lens.evidence_changed` | Provider evidence changed for one or more anchors. |
| `lens.anchor_linked` | Provider linked evidence to an anchor. |
| `lens.anchor_unlinked` | Provider removed an evidence-to-anchor link. |
| `lens.lookup_completed` | Optional audit receipt for lookup completion. |
| `lens.lookup_failed` | Optional audit receipt for lookup failure. |

Event payloads SHOULD include:

```json
{
  "provider": "paperless_ngx",
  "anchor_type": "label",
  "anchor_ids": ["medical"],
  "evidence_ids": ["paperless_ngx:document:12"],
  "status": "success",
  "risk_class": "read_redacted",
  "content_redacted": true,
  "caller_token": "..."
}
```

Event rules:

* Events are fire-and-forget.
* Events must use `continue_on_error: true` when emitted from workflow hooks.
* Consumers must not wait indefinitely for event responses.
* Events may trigger reindexing, audit receipts, or provider cache refreshes.
* Event emission must not bypass the provider security act.

---

## Health, Audit, And Help

Every provider needs agent-readable operational surfaces.

Health report SHOULD include:

```json
{
  "provider": "paperless_ngx",
  "status": "available",
  "configured": true,
  "registry": "registered",
  "dependencies": {
    "api": "available",
    "token": "configured"
  },
  "last_success_at": "2026-06-05T11:32:44-05:00",
  "last_error": null
}
```

Audit report SHOULD include:

```json
{
  "provider": "paperless_ngx",
  "recent_activity": [
    {
      "event": "lens.lookup_completed",
      "anchor_type": "label",
      "count": 6,
      "content_redacted": true
    }
  ],
  "policy": {
    "security_act": "r",
    "risk_class": "read_redacted",
    "failure_policy": "soft"
  }
}
```

Help SHOULD include:

* Provider purpose.
* Setup actions.
* Lens lookup mode.
* Accepted anchor types.
* Returned evidence classes.
* Security act.
* Failure behavior.
* Known limitations.
* Example calls.

This is not decoration. Agents need these surfaces so they can inspect a broken provider, confirm whether it is registered, read its limits, and explain what is going on without guessing.

---

## Security

Default lens lookup is read-only and redacted.

Security rules:

* `security_act` is a declaration, not permission by itself.
* Policy decides whether a caller can use the declared act.
* Missing caller authority must not unlock raw evidence.
* Raw document text, OCR, private notes, full ticket bodies, and PII-bearing details require explicit read modes and policy gates.
* Mutating actions are not lenses.
* CRUD modes may exist beside a provider, but CRUD must not run through the lens lookup path.
* Bulk edits are allowed only when they pass the same checks as equivalent individual edits.

`rwxda` flags:

| Flag | Meaning |
|------|---------|
| `r` | Read redacted evidence |
| `w` | Write/update provider records |
| `x` | Execute side-effecting workflow |
| `d` | Delete/destructive operation |
| `a` | Administrative/provider configuration |

Most providers should register lenses with only `r`.

---

## Failure Semantics

Lens providers are optional.

Expected behavior:

* Provider unavailable: dispatcher returns success with unavailable/empty provider status.
* Provider timeout: caller still receives its base response.
* Provider error: error remains contained under that provider.
* No anchors: consumer result remains valid.
* No matching evidence: `evidence_count: 0`.
* Registry missing: dispatcher returns empty lens context, not a broken consumer.

Consumers MUST NOT fail the primary user workflow just because lens enrichment failed.

---

## Current Implementation

### Inspect

`zen_dojotools_inspect` supports `+lenses`.

It builds anchors from:

* Entity labels -> `label`
* Entity ID beginning `person.` -> `person`
* Device/room area -> `area_id`
* Entity ID beginning `zone.` -> `zone`

It injects:

* `domain_context.lenses`
* per-result `lens_context`

### Stacks / Paperless-NGX

Paperless registers as provider `paperless_ngx`.

It uses internal provider tool:

* `zen_stack_paperless`

It consumes:

* `label`
* `person`
* `area_id`
* `zone`

It returns:

* `document_evidence`

Paperless lens output is sanitized:

* Raw content/OCR is not returned in lens output.
* `content_redacted: true` is set.
* Tags and metadata are mapped into `anchor_contexts`.
* Suggestions remain separate from explicit anchors.

---

## Future Provider Checklist

Before a new knowledge source is registered:

* Create an internal provider surface, such as `zen_stack_wikijs`.
* Add a setup/config mode with `inspect`, `register`, `unregister`, `health`, `audit`, and `help`.
* Register in `lens_registry`.
* Declare `consumes`, `returns`, `risk_class`, `content_policy`, and `security_act`.
* Support standard lookup input.
* Return standard evidence envelopes.
* Emit provider registration and evidence-change events.
* Consume registry/anchor-change events if local cache or mappings exist.
* Keep lookup read-only.
* Prove unavailable providers fail soft.

---

## HALMark Notes

* Documentation is ground truth.
* Do not guess entity IDs, area IDs, zones, people, labels, or cabinet names.
* Do not treat a label as a primitive anchor unless the anchor type is `label`.
* Keep edits surgical when adding providers.
* Keep provider failure soft for read enrichment.
* Keep raw content out of default lens responses.
* Avoid unbounded provider expansion; page sizes and result limits must be explicit.
* Do not let event hooks become hidden mutation paths.
* Do not let provider registration bypass policy or audit.
