# ZenOS-AI Wiki.js Plugin

**Version:** 0.4.1  
**Package:** `packages/zenos_ai/plugins/wiki_js/dojotools_wikijs.yaml`  
**Public surface:** `zen_dojotools_filecabinet` with `stack=wiki`  
**Internal sutra:** `zen_sutra_wikijs` (provider_key: `wiki_js`, not MCP-exposed)  
**Internal GraphQL broker:** `zen_dojotools_wikijs_root` (not MCP-exposed)

> **Auto-registration (2026.7.0):** `zen_sutra_wikijs` now participates in Lens Bus bootstrap registration. On every HA boot, `zen_dojotools_manifest mode=bootstrap_stacks` discovers and registers it automatically. See [Lens Bus Auto-Registration](lens_bus_autoreg.md).

---

## Overview

Wiki.js is the internal knowledge base for ZenOS-AI. The plugin bridges Home Assistant automations to the Wiki.js GraphQL API for reading, writing, organizing, and tagging wiki pages.

**There is no standalone MCP-facing wiki tool.** The public surface is `zen_dojotools_filecabinet` with `stack=wiki`. `zen_sutra_wikijs` is the internal terminus — all wiki CRUD and Lens Bus participation happen there, and it is not exposed to MCP.

---

## Architecture

```
zen_dojotools_filecabinet (stack=wiki)  ← public surface, MCP-exposed
  → zen_sutra_wikijs                    ← sutra: business logic, stewardship, Lens provider
    → zen_dojotools_wikijs_root         ← root: GraphQL transport
      → Wiki.js GraphQL API
```

The sutra layer enforces stewardship rules:

* **Read-modify-write on update** — existing content and tags are fetched before any mutation.
* **Surgical diff guard** — if a `merge` edit would change more than 50% of the document, the write is blocked and a warning is returned. Use `mode=replace` to confirm an intentional full overwrite.
* **Tag merging** — supplied tags are merged with existing tags on update; no existing tags are dropped unless `mode=replace` is used.

---

## Using Wiki via FileCabinet `stack=wiki`

All wiki operations go through `zen_dojotools_filecabinet` with `stack: wiki`. Pass `action` and any required fields in the payload.

```yaml
zen_dojotools_filecabinet:
  stack: wiki
  action: get
  page_id: 42
```

### Supported Actions

| Action | Required fields | Notes |
|--------|----------------|-------|
| `get` | `page_id` or `path` | Returns full page including content, tags, and `isPublished`. |
| `list` | — | Optional: `tags_json` (filter by tag), `path_prefix` (filter by path prefix). |
| `search` | `query` | Full-text search. Returns `{id, path, title}` per result. |
| `create` | `path`, `title`, `content` | Optional: `description`, `tags_json`, `is_published` (default true). |
| `update` | `page_id` or `path`, `content` | Optional: `mode` (replace/merge, default merge), `tags_json`, `is_published`. Surgical guard applies on merge. |
| `upsert` | `path`, `title`, `content` | Create if path not found, update if found. |
| `delete` | `page_id` | Irreversible. |
| `move` | `page_id` or `path`, `destination_path` | Moves page to a new path. |
| `copy` | `page_id` or `path`, `destination_path` | Duplicates page at new path. |
| `relabel` | `page_id` or `path`, `tags_json` | Update tags only — does not change content. |
| `set_published` | `page_id` or `path` | Pass `is_published: true/false/toggle`. |
| `health` | — | Probe Wiki.js reachability. |
| `inspect` | — | Read Lens registry entry for wiki_js. |
| `register` | — | Register wiki_js in the household Lens registry. |
| `unregister` | — | Remove wiki_js from the Lens registry. |
| `audit` | — | Return Lens security and policy metadata. |

**Not supported on the wiki stack** (no-ops that return `status: not_supported`): `mount_cabinet`, `dismount_cabinet`, `set_mount`, `remove_mount`, `set_cabinet_ro`, `clone`, `set_expiry`. These concepts have no meaning for wiki pages; the sutra returns a clear reason for each.

---

## zen_sutra_wikijs — Lens Bus Provider

`zen_sutra_wikijs` participates in the Lens Bus as a provider of `article_evidence`.

| Property | Value |
|----------|-------|
| Provider key | `wiki_js` |
| Consumes | `label`, `concept`, `area_id`, `zone`, `person` |
| Returns | `article_evidence` |
| Security | `r-only` — content is redacted in evidence output |
| Failure policy | `soft` |

### Room Page Tagging Convention

For a wiki page to surface via the Lens Bus when `anchor_type=area_id`, the page must:

1. Be tagged with the HA area_id slug (e.g. `office`, `kitchen`, `water_heater_closet`).
2. Be published (`isPublished=true`).

Tag a room page and publish it:

```yaml
# Tag the page with the area slug
zen_dojotools_filecabinet:
  stack: wiki
  action: relabel
  path: zenai/rooms/kitchen
  tags_json: '["kitchen"]'

# Publish the page
zen_dojotools_filecabinet:
  stack: wiki
  action: set_published
  path: zenai/rooms/kitchen
  is_published: "true"
```

Once tagged and published, Room Manager's `+wiki` context slice discovers the page automatically via `stacks_by_anchor`.

---

## Evidence Shape

```json
{
  "evidence_id": "wiki_js:article:<page_id>",
  "provider": "wiki_js",
  "object_type": "article",
  "title": "<page title>",
  "path": "<page path>",
  "anchor_contexts": [{"type": "<anchor_type>", "id": "<slug>", "confidence": "search_match"}],
  "content_redacted": true
}
```

---

## First-Time Setup

1. Add the Wiki.js API token to `secrets.yaml`. The secret stores the full Authorization header value including the `Bearer ` prefix:

```yaml
wikijs_token_bearer: "Bearer YOUR_WIKIJS_API_TOKEN"
```

2. Store the Wiki.js URL in the household cabinet. Run the configure action via the root tool (internal — do this once at setup):

```yaml
action.script/zen_dojotools_wikijs_root:
  action_type: configure
  base_url: "https://your-wikijs-host"
```

This writes the URL to `integrations_config.wikijs.url` in the household cabinet.

3. Register the Lens provider so other tools can discover wiki content:

```yaml
zen_dojotools_filecabinet:
  stack: wiki
  action: register
```

4. Expose `zen_dojotools_filecabinet` to the conversation agent (MCP). Do not expose `zen_sutra_wikijs` or `zen_dojotools_wikijs_root`.

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| All wiki calls return `error` | Run `action: health` and inspect `availability`. Confirm URL and token. |
| Any call (other than `help`/`configure`) returns `status: not_configured` | Wiki.js was never configured on this install — no `base_url` given and `integrations_config.wikijs.url` is empty. Run `action_type: configure base_url=<your Wiki.js URL>` first. This is a fast fail, not an error — no request was sent to Wiki.js. |
| Update blocked with `surgical_check: failed` | The merge diff ratio exceeded 50%. Use `mode: replace` if the full overwrite is intentional. |
| Room page not surfacing in Lens Bus | Confirm the page is tagged with the HA area slug and `isPublished=true`. Use `relabel` and `set_published`. |
| Tag on `list` returns no results | Wiki.js tag filter is exact-match. Confirm the tag string matches exactly. |
