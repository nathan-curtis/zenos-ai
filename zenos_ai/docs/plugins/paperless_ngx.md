# ZenOS-AI Paperless-NGX Plugin

**Version:** 0.7.4  
**Package:** `packages/zenos_ai/plugins/paperless_ngx/paperless_ngx.yaml`  
**Lens Bus stack provider:** `zen_stack_paperless` (provider_key: `paperless_ngx`, internal — not MCP-exposed)  
**Internal REST broker:** `zen_sutra_paperless` (not MCP-exposed)  
**Public surface:** `zen_dojotools_library` with `section=stacks`

> **Auto-registration (2026.7.0):** `zen_stack_paperless` now participates in Lens Bus bootstrap registration. On every HA boot, `zen_dojotools_manifest mode=bootstrap_stacks` discovers and registers it automatically — no manual `lens_registry` writes needed. See [Lens Bus Auto-Registration](lens_bus_autoreg.md).

---

## Overview

Paperless-NGX is the document archive surface for ZenOS-AI. The plugin connects ZenOS to a self-hosted Paperless-NGX instance and exposes its documents as Lens Bus evidence. It does not provide a standalone MCP tool. Document lookups reach it through the Library stack dispatcher — the same path used by Room Manager, Lens enrichment, and other context-aware tools.

**CRUD operations** (direct document create/update/delete) are plumbed through `zen_sutra_paperless` but are not part of the current public surface. The lens path is read-only.

---

## Architecture

```
zen_dojotools_library (section=stacks)  ← public surface, MCP-exposed
  → zen_stack_paperless                 ← Lens Bus stack provider (internal)
    → zen_sutra_paperless               ← REST broker
      → Paperless-NGX REST API (/api/v1/)
```

The REST broker supports GET, POST, PATCH, and DELETE verbs but the current stack surface is read-only. All endpoints must begin with `/api/`.

---

## Lens Bus Integration

`zen_stack_paperless` participates in the Lens Bus as a provider of `document_evidence`.

| Property | Value |
|----------|-------|
| Stack key | `paperless` (via Library `section=stacks`) |
| Consumes | `label`, `person`, `area_id`, `zone` |
| Returns | `document_evidence` |
| Security | `r-only` — document content is redacted by default |
| Failure policy | `soft` |
| Content policy | `redacted_by_default` |

### Using via Library

```yaml
zen_dojotools_library:
  section: stacks
  mode: stacks_by_anchor
  input_json: '{"anchor_ids": ["kitchen"], "anchor_type": "area_id"}'
```

### Document evidence shape

```json
{
  "id": "<Paperless document id>",
  "title": "<document title>",
  "tags": ["<tag name>", ...],
  "correspondent": "<correspondent name>",
  "created": "<ISO8601 date>"
}
```

Content is redacted by default. Pass `include_content: true` in `input_json` to opt in to full content (not recommended for large document sets in context-bound calls).

---

## Stack Modes

| Mode | Purpose |
|------|---------|
| `stacks_by_anchor` | Find documents by HA anchor (label, person, area_id, zone). Main Lens lookup path. |
| `stacks_configure` | Write Paperless-NGX URL (and optional `api_version`) to household cabinet and register the provider. |
| `register` | Register `paperless_ngx` in the household Lens registry. |
| `unregister` | Remove `paperless_ngx` from the Lens registry. |
| `inspect` | Read the current Lens registry entry for `paperless_ngx`. |
| `health` | Probe Paperless-NGX reachability and return a human-readable availability diagnosis. |
| `audit` | Return Lens security and policy metadata. |
| `tool_manifest` | Return the full Lens provider manifest. |
| `help` | List available modes. |

---

## First-Time Setup

1. Add the Paperless-NGX API token to `secrets.yaml`. The secret stores the full Authorization header value including the `Token ` prefix:

```yaml
paperless_ngx_token: "Token YOUR_PAPERLESS_API_TOKEN"
```

2. Run `stacks_configure` to write the URL to the household cabinet and register the Lens provider:

```yaml
zen_dojotools_library:
  section: stacks
  mode: stacks_configure
  input_json: '{"provider": "paperless_ngx", "url": "https://your-paperless-host"}'
```

This writes the URL to `integrations_config.paperless_ngx.url` in the household cabinet. Optionally include `"api_version": 9` (default is 9).

3. Expose `zen_dojotools_library` to the conversation agent (MCP). Do not expose `zen_stack_paperless` or `zen_sutra_paperless`.

---

## Troubleshooting

| Symptom | Likely cause | Check |
|---------|-------------|-------|
| `not_configured` error | URL not set | Run `stacks_configure` with the Paperless URL |
| `auth_failed` (HTTP 401) | Token rejected | Check `paperless_ngx_token` in `secrets.yaml`; confirm it includes the `Token ` prefix |
| `non_json_response` on a 200 | Proxy login page or wrong path | Confirm the base URL points directly to the Paperless API, not a login redirect |
| `not_found` (HTTP 404) | Wrong base URL or reverse proxy path | Check DNS, base URL trailing slash, and proxy path strip rules |
| `stacks_by_anchor` returns empty | No documents tagged with the anchor slug | Tag Paperless documents with the HA label or area slug |
| `server_error` (HTTP 5xx) | Paperless worker or DB issue | Check Paperless container logs and Redis/database health |
