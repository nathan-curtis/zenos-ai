# Zen DojoTools Library — v6.12.0 (ZenOS-AI 2026.7.0 'Neo')

*Knowledge broker and Lens owner for the Monastery*

---

## Overview

`zen_dojotools_library` is the **knowledge broker** for ZenOS-AI. It owns and coordinates all Lens provider interfaces, routes knowledge requests to registered stack providers, normalizes evidence output, and fails soft when a provider is absent or misconfigured.

The Library is **MCP-exposed**. Consumers — Inspect, Index, Query, Room Manager — call the Library directly. They never call provider scripts (`zen_stack_paperless`, `zen_stack_radar`) themselves. The Library is the single entry point for all knowledge surfaces.

Content is **redacted by default**. Pass `include_content=true` in `input_json` to opt in to full document body.

---

## Lens Bus — `stack=` Routing

v5.5.0 introduces the Lens Bus architecture. Generic knowledge verbs (`get`, `find`, `list`, `configure`, `by_anchor`) are dispatched through the Library to whichever registered stack provider the caller selects via the `stack=` field.

```
caller
  └─ zen_dojotools_library (section=stacks, stack=radar, mode=find)
       └─ zen_stack_radar  →  zen_dojotools_servicedesk (ticket_find)
```

The Library resolves the provider's config from the household FileCabinet on each call (`integrations_config.<provider>`). If the provider is unconfigured or unreachable, the Library returns `status=error` — it never raises.

### `section=stacks` vs `section=radar`

| Section | Purpose |
|---|---|
| `section=stacks` | Browse registered stack providers, or route to a knowledge stack via `stack=`. |
| `section=radar` | Shorthand entry point for Zammad/Radar service desk access. Equivalent to `section=stacks, stack=radar`. |

---

## Generic Verbs

These are the universal Lens verbs. Any registered stack provider must handle all of them. Pass them as `mode=`.

| Verb | What It Does |
|---|---|
| `get` | Return one record by ID. Requires `input_json.ticket_id` (radar) or `input_json.document_id` (paperless). |
| `find` | Search records by query string. Requires `input_json.query` or the `query` field. |
| `list` | List open/active records. No anchor required. |
| `configure` | Inspect the stack provider's config state. Read-only. |
| `by_anchor` | Find records linked to a typed anchor (label, person, area, ticket). Requires `input_json.anchor_type` and `input_json.anchor_value`. |

Generic verbs are forwarded to the provider, which maps them to its own internal calls. The caller does not need to know the provider's native API surface.

---

## Registered Stack Providers

| `stack=` | Provider Script | Security | Description |
|---|---|---|---|
| `paperless` | `zen_stack_paperless` | read-redacted + write | Paperless-NGX document archive. Content redacted by default. Also supports correspondent management (`stacks_correspondents_list`, `stacks_correspondent_get`, `stacks_correspondent_create`, `stacks_correspondent_update`), bulk document edits (`split`, `reprocess`, `set_correspondent`, `set_document_type`, `set_storage_path`, `add_tag`, `remove_tag`, `delete`), and `stacks_flag_for_review` to flag a document and notify. |
| `wiki` | `zen_dojotools_filecabinet` (stack=wiki) | read-redacted | Wiki.js pages via `zen_sutra_wikijs`. Requires Wiki.js integration installed and registered. |
| `radar` | `zen_stack_radar` | r-only | Zammad service desk tickets. See section below. |
| `media` | `zen_dojotools_media_manager` | read-only | Music Assistant media — tracks, albums, playlists, artists. Returns evidence leaves with `playback_hint`. |

Providers register themselves via their own `mode=register` call (writes to `lens_registry` in the household cabinet). Run `mode=configure` against any stack to inspect its config state. To register a provider: call `zen_dojotools_<provider> mode=register`. To list all registered providers with live status: `section=stacks mode=stacks_list`.

To list all registered providers with live status, use:

```yaml
section: stacks
mode: stacks_list
```

---

## `zen_stack_radar` — Zammad Service Desk Stack

**Plugin:** `plugins/zammad/zammad.yaml`
**Version:** 1.0.0 (codename: Radar)
**Security:** `r-only` — no write operations

`zen_stack_radar` is the Lens stack provider for Zammad (service desk / help tickets). It proxies generic Lens verbs to `zen_dojotools_servicedesk` modes:

| Generic Verb | Maps To (`zen_dojotools_servicedesk`) |
|---|---|
| `get` | `ticket_get` |
| `find` | `ticket_find` |
| `list` | `ticket_list` |
| `configure` | Reads `integrations_config.fulfillment` from household cabinet |
| `by_anchor` | `tickets_by_anchor` |

**Config location:** `integrations_config.fulfillment.url` in the household cabinet. Run `zen_dojotools_servicedesk case=configure` to bootstrap.

**Secrets:** `zammad_token` — full Authorization header value (`Token token=YOUR_ZAMMAD_API_TOKEN`).

**Never call `zen_stack_radar` or `zen_dojotools_servicedesk` directly.** Route through the Library:

```yaml
section: stacks
stack: radar
mode: find
input_json: '{"query": "heating system"}'
```

Or use the `section=radar` shorthand:

```yaml
section: radar
mode: find
input_json: '{"query": "heating system"}'
```

---

## Legacy `tool=` Surface

The original utility surface still works and is still MCP-exposed. These are **not** Lens routes — they are standalone utility functions. Use `tool=` to invoke them, not `stack=`.

| Tool | What It Does |
|---|---|
| `library` | Legacy command dispatch — `command_interpreter.jinja` removed in 2026.7.0. This tool mode is a stub. Use `stack=` routing instead. |
| `hash_md5` | Computes MD5 hash of the input string. Returns `{tool, query, output}`. |
| `slugify` | Applies HA's `slugify()` filter to the input string. Returns `{tool, query, output}`. |

**Default tool:** `library`

### When to use `tool=` vs `stack=`

- Use `stack=` when accessing a knowledge surface (documents, tickets, wiki pages).
- Use `tool=` when computing a value (hash, slug) or running a library command.

### Examples

```yaml
# Hash a string
tool: hash_md5
query: "my-string-to-hash"
```

```yaml
# Slugify a name
tool: slugify
query: "Security Manager"
# output: "security_manager"
```

---

## `~commands~` Retirement Notice

> **Retiring at GA.** The `~COMMANDS~` interface (`command_interpreter.jinja`) is being retired. Individual commands are migrating to index-supported constructs. No new commands should be added to `command_interpreter.jinja`.

The `library` tool currently routes queries through `command_interpreter.jinja`. Kung Fu components register their library command via the `command` field in their Dojo drawer. The Ninja Summarizer calls the Library automatically before building the monk prompt — the output lands in `library_console` in the review data.

Individual command tokens (`~SECURITY~`, `~MEDIA~`, etc.) are not documented here — this interface is retiring at GA.

---

## Input Fields

| Field | Required | Description |
|---|---|---|
| `section` | No | Library department. Default: `stacks`. Values: `stacks`, `radar`, `catalog`. |
| `stack` | No | Stack provider to route to: `paperless`, `radar`, `wiki`, `media`. |
| `mode` | No | Generic verb or legacy mode. Default: `help`. |
| `item_type` | No | Works catalog item type: `book`, `game`, `music_recording`, `video`, `periodical`, `art_print`. Required when `section=catalog`. |
| `query` | No | Text input or search query. |
| `input_json` | No | Structured JSON payload for modes that require it. |
| `caller_token` | No | Opaque pass-through token for correlation. Not interpreted. |

---

## Capability Tiers — What You Unlock

Library is independently useful at every tier. Each addition compounds the previous.

| Tier | Requires | What you get |
|------|----------|-------------|
| **1 — Knowledge Broker** | Library only | Lens Bus routing, `stack=` dispatch to registered providers, evidence envelopes, generic verbs |
| **2 — Circulation Desk** | + Grocy (Inventory plugin) | Full physical catalog: browse/find/search/add/loan/return across books and games. Library science enforced — location tracking, dedup guards, loan lifecycle |
| **3 — Media-Aware Library** | + Media Manager | Music evidence via `stacks_by_anchor`. Evidence envelopes carry `playback_hint` — caller gets both the item and the call shape to play it |
| **4 — Room-Context Evidence** | + Room Manager `+media` | Anchor searches know what's already playing in the target room. Evidence confidence boosted when provider matches room's current source |

The Library **silently degrades** when a tier dependency is absent. If Grocy is unavailable, catalog modes skip the call and return empty. If Media Manager is unreachable, `stack=media` returns `status: degraded` with empty evidence — no error raised.

---

## `section=catalog` — Unified Works Catalog

v6.x+ replaces the separate `section=books` and `section=games` surfaces with a unified catalog keyed by `item_type`. The catalog is backed by Grocy; Library enforces library science (ISBN dedup, loan lifecycle, location tracking) behind the scenes.

```yaml
zen_dojotools_library:
  section: catalog
  item_type: book   # book | game | music_recording | video | periodical | art_print
  mode: browse
```

### Catalog Modes

| Mode | What It Does |
|------|-------------|
| `browse` | List all items of the given `item_type`. Optional `query` for title/author/platform filter. Optional `limit`/`offset` (or `per_page`) in `input_json` page the result — response reports `total_count` separately from the (possibly page-limited) `count`/`items`. Omitting `limit` returns every matching item, as before. |
| `find` | Search by title, author/artist, ISBN, platform, or tag. Returns `{items[], count}`. |
| `search` | Full-text search across all item fields including notes. Accepts `q` or `query`. |
| `add` | Add an item. Requires `isbn` or `title`. ISBN triggers Grocy dedup guard — returns existing record if found. |
| `update` | Retroactively update an existing catalog item (v6.12.0). Requires `item_id`. Safe partial update: reads current userentity values, `combine()`s the caller's partial `input_json` onto them, then writes the merged dict — never a blind overwrite. Same field set as `add` (including `mealie_cookbook_slug`), so an already-shelved item can be edited without recreating it. `title` renames the underlying Grocy product directly (it lives on the product's `name` field, not as a userentity value, so it can't be merged like the other fields) — previously there was no `update` mode at all, so title changes after add were not possible. |
| `move` | Reshelve a catalog item to a named location. Resolves the location by a real targeted `locations_find` query (not a paginated `locations_list`), so it isn't capped by Grocy's ~250-per-page listing and can find locations created after the first page. |
| `loan` | Check out an item to a person. Requires `item` and `person` (HA person entity or name). Writes loan record to Grocy userfields. Uses `inventory_root` from borrower profile as destination location. |
| `return` | Return a loaned item. Requires `item`. Clears loan fields, restores item to home location. |

### Works Schema

All catalog items share the unified userfield schema stored in `library_meta` (JSON blob in Grocy):

| Field | Description |
|-------|-------------|
| `library_meta` | JSON blob: `platform`, `digital`, `hardware_note` (games); `author`, `isbn`, `series`, `genre` (books) |
| `library_loan_borrower` | Person entity ID or name of current borrower |
| `library_loan_date` | ISO date loan was checked out |
| `library_loan_due` | ISO date loan is due back |
| `library_loan_home_location_id` | Grocy location ID to restore to on return |

### Location Discovery

Grocy locations tagged with the HA label `bookshelf` are auto-discovered for browse/find queries. Tag once, the whole shelf tree joins the catalog.

### Legacy: `section=books` (v5.6.0–v5.9.0)

The original `section=books` surface (separate from games, flat-key userfields) was retired in v6.1.0. The unified `section=catalog item_type=book` is the replacement. v5.x loan fields (`on_loan_to`, `loan_date`, `loan_due`, `loan_notes`) are read via dual-source fallback — the Library checks both flat keys and `library_meta` on decode, so existing book records continue to resolve without migration.

---

## MCP Exposure

The MCP-facing script is `zen_dojotools_library`. Consumers and agents call this. Provider scripts (`zen_stack_paperless`, `zen_stack_radar`, `zen_dojotools_servicedesk`) are internal and should not be MCP-exposed or called directly by agents.

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `zen_stack_paperless` | Paperless-NGX document Lens provider |
| `zen_stack_radar` | Zammad service desk Lens provider (v1.0.0) |
| `zen_dojotools_servicedesk` | Radar's internal Zammad call surface |
| `zen_sutra_filecabinet` | Provider config registration and config reads |
| ~~`command_interpreter.jinja`~~ | Removed in 2026.7.0. |
| HA `md5` filter | MD5 hash computation (legacy `tool=` surface) |
| HA `slugify()` filter | String slugification (legacy `tool=` surface) |

---

## Version History

| Version | Change |
|---------|--------|
| v6.12.0 | `section=catalog mode=update` (safe partial update via `combine()`, never a blind overwrite; `title` renames the Grocy product directly). Paperless correspondent management (`stacks_correspondents_list/get/create/update`), bulk document edit `split`/`reprocess` methods, and `stacks_flag_for_review`. `catalog move`/`update`'s location lookups use a targeted `locations_find` query instead of paginated `locations_list`, which was silently missing locations past Grocy's ~250-item page cap. |
| v5.9.0 | `books_loan`, `books_return`, `books_configure` modes. KFC schema v1.4.0 loan fields: `on_loan_to`, `loan_date`, `loan_due`, `loan_notes`. Loan uses `inventory_root` from borrower's profile as destination location. |
| v5.8.0 | `move` mode (relocate a book by title/ISBN to a new Grocy location). `stock_transfer_location` (bulk move). |
| v5.7.0 | `add` mode with ISBN dedup guard (checks existing Grocy products before creating). |
| v5.6.0 | `section=books` introduced. `browse`, `find`, `search` modes. Bookshelf discovery via `bookshelf` HA label on Grocy locations. |
| v5.5.0 | Lens Bus architecture (`stack=` routing). `zen_stack_radar` registered as Radar provider. Generic verbs (`get/find/list/configure/by_anchor`). `zen_dojotools_wikijs` retired; wiki access via `stack=wiki`. |
