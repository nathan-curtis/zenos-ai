# Zen DojoTools Library — v5.9.0 (ZenOS-AI 2026.7.0 'Neo')

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
| `paperless` | `zen_stack_paperless` | read-redacted | Paperless-NGX document archive. Content redacted by default. |
| `wiki` | `zen_stack_paperless` (stack=wiki) | read-redacted | Wiki pages via Paperless storage path partition. |
| `radar` | `zen_stack_radar` | r-only | Zammad service desk tickets. See section below. |

Providers register themselves by writing their config to `integrations_config.<provider>` in the household FileCabinet via `zen_sutra_filecabinet`. Run `mode=configure` against any stack to inspect its config state.

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
| `section` | No | Library department. Default: `stacks`. |
| `stack` | No | Stack provider to route to: `paperless`, `radar`, `wiki`. |
| `mode` | No | Generic verb or legacy mode. Default: `help`. |
| `query` | No | Text input or search query. |
| `input_json` | No | Structured JSON payload for modes that require it. |
| `caller_token` | No | Opaque pass-through token for correlation. Not interpreted. |

---

---

## `section=books` — Physical Book Catalog

v5.6.0–v5.9.0 adds a full physical book catalog backed by Grocy (ISBN lookup, stock locations, lending). Use `section=books` with a `mode=` to access.

```yaml
zen_dojotools_library:
  section: books
  mode: browse
```

### Book Modes

| Mode | Version | What It Does |
|------|---------|--------------|
| `browse` | v5.6.0 | List all books in the catalog. Optional `query` for title/author filter. |
| `find` | v5.6.0 | Search catalog by title, author, ISBN, or tag. Returns `{books[], count}`. |
| `search` | v5.6.0 | Full-text search across all book fields including notes. |
| `add` | v5.7.0 | Add a book to the catalog. Requires `isbn` or `title`. ISBN triggers Grocy product lookup/creation with dedup guard — if a product with that ISBN exists, returns the existing record rather than creating a duplicate. |
| `move` | v5.8.0 | Move a book to a different shelf location. Requires `item` (title or ISBN) and `location` (Grocy location ID or name). |
| `stock_transfer_location` | v5.8.0 | Bulk-move all books from one location to another. |
| `books_loan` | v5.9.0 | Check out a book to a person. Requires `item` (title or ISBN) and `person` (HA person entity or name). Writes loan record to the book's Grocy userfields. Uses `inventory_root` from the borrower's profile as the destination location. Schema v1.4.0 loan fields. |
| `books_return` | v5.9.0 | Return a loaned book. Requires `item`. Clears loan fields, restores book to home location. |
| `books_configure` | v5.9.0 | Configure the books catalog: set default home location (`bookshelf_default_location_id`), loan period (`loan_period_days`), and other catalog defaults. Written to household cabinet `books_config` drawer. |

### Book Schema (v1.4.0 loan fields)

Books in the Grocy catalog carry these userfields:

| Field | Description |
|-------|-------------|
| `isbn` | ISBN-13 (canonical) |
| `author` | Author string |
| `series` | Series name if applicable |
| `series_number` | Position in series |
| `tags` | Comma-separated tags |
| `on_loan_to` | Person entity ID or name of current borrower (`null` when in) |
| `loan_date` | ISO date loan was checked out |
| `loan_due` | ISO date loan is due back (loan_date + loan_period_days) |
| `loan_notes` | Free-form notes on the loan |

### Bookshelf Discovery

The Library tool discovers all bookshelf locations by looking for Grocy locations tagged with the HA label `bookshelf`. Tag a parent Grocy location with `bookshelf` and all child locations (shelves, bins) are automatically included in browse/find queries. Tag once, the whole shelf tree joins the library.

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
| v5.9.0 | `books_loan`, `books_return`, `books_configure` modes. KFC schema v1.4.0 loan fields: `on_loan_to`, `loan_date`, `loan_due`, `loan_notes`. Loan uses `inventory_root` from borrower's profile as destination location. |
| v5.8.0 | `move` mode (relocate a book by title/ISBN to a new Grocy location). `stock_transfer_location` (bulk move). |
| v5.7.0 | `add` mode with ISBN dedup guard (checks existing Grocy products before creating). |
| v5.6.0 | `section=books` introduced. `browse`, `find`, `search` modes. Bookshelf discovery via `bookshelf` HA label on Grocy locations. |
| v5.5.0 | Lens Bus architecture (`stack=` routing). `zen_stack_radar` registered as Radar provider. Generic verbs (`get/find/list/configure/by_anchor`). `zen_dojotools_wikijs` retired; wiki access via `stack=wiki`. |
