# ZenOS-AI Zammad (Radar) Plugin

**Version:** 0.4.0 (`zen_dojotools_servicedesk`) / 1.0.0 (`zen_stack_radar`)
**Package:** `packages/zenos_ai/plugins/zammad/zammad.yaml`
**MCP-facing script:** `zen_dojotools_servicedesk`
**Lens Bus stack provider:** `zen_stack_radar`
**Internal REST broker:** `zen_sutra_zammad` (not MCP-exposed)

---

## Overview

Zammad is the service desk and fulfillment surface for ZenOS-AI, codename **Radar**. It handles help tickets, work requests, triage, and customer/org management via the Zammad REST API.

ZenOS exposes two layers:

* `zen_dojotools_servicedesk` — the full-featured MCP tool for explicit ticket operations (create, update, triage, search, close, etc.).
* `zen_stack_radar` — a thin Lens Bus adapter that maps generic Library verbs to servicedesk modes, for use when another tool (e.g., Library) needs to look up ticket evidence by HA anchor without calling servicedesk directly.

---

## zen_dojotools_servicedesk Modes

Use `mode=help` for the full field catalog. Key modes:

| Mode | Purpose |
|------|---------|
| `configure` | Write Zammad URL to household cabinet and test-connect. Also registers `zen_stack_radar` in the Lens registry. |
| `radar_setup` | Idempotent Radar initialization wizard. Sub-actions: `init` (find-or-create house/zenos personas, write `radar_defaults` to household cabinet), `map_queue` (assign default group/queue for house or zenos reporter), `set_org_default` (set default org for a persona), `inspect_queue` (read current queue config). Input fields: `action` (default `init`), `house_email`, `dev_email` (persona email addresses). |
| `ticket_assign` | Assign a ticket to a Zammad agent by name, email, or HA person entity (`person.*`). Resolves the agent via the household roster. |
| `ticket_create` | Create a new ticket. Accepts `person_entity_id` for roster-based customer resolution or explicit `customer` email. Reporter shorthand: `reporter=house` (household persona) or `reporter=zenos` (dev persona) — bypasses group lookup; seeded by `radar_setup`. |
| `ticket_get` | Fetch a ticket including all articles, tags, and anchor context. |
| `ticket_find` | Search tickets by query string. |
| `ticket_list` | List tickets by state (open/closed/all). |
| `ticket_search` | Structured Radar worklist search — filters by queue, state, owner, tags, and priority. |
| `ticket_update` | Update ticket fields including group/queue move, org/customer reassignment, and pending time. |
| `batch_update` | Apply the same updates to multiple tickets in one call. |
| `ticket_close` | Set ticket state to closed. |
| `ticket_complete` | Add a completion receipt article, optionally tag/anchor, and close — all in one call. |
| `article_add` | Add an internal note or email reply to a ticket. |
| `articles_list` | List all articles on a ticket. |
| `ticket_tag` | Add a tag to a ticket. |
| `triage_set` | Record structured Radar triage metadata (rank, lane, sequence, blocked_by, milestone, release_train) as deterministic `radar_*` tags plus a receipt note. |
| `triage_get` | Read triage metadata back from ticket tags and receipt notes. |
| `tickets_by_anchor` | Find tickets tagged with an HA slug — label, area, person, zone, company, family, or household. |
| `ticket_link` / `ticket_unlink` | Create or remove parent/child/related links between tickets. |
| `customer_find` / `customer_create` | Look up or register Zammad customers. |
| `org_find` / `org_create` | Look up or create Zammad organizations. |
| `queue_get` | List Zammad groups (queues) or saved overviews. |
| `view_get` | Run a synthetic Radar worklist view (new_unassigned, open_unassigned, my_open, pending_reminder, escalating, recent_customer_reply) or return the view catalog. |
| `workflow_context_get` | Return read-only Radar workflow metadata and permission diagnostics. |
| `object_attribute_create` | Create a custom field on the Ticket object and run migrations to apply it. Requires admin token. |
| `tool_manifest` | Return the Lens Bus provider manifest for this tool. |

**Note on ticket IDs:** `ticket_id` accepts either the display number (e.g. `10073`) or the raw Zammad DB id (e.g. `73`) — both resolve correctly.

**Note on `priority` (2026.7.1):** `ticket_create` and `ticket_update` accept `priority` as either a bare number/word (`1`/`low`, `2`/`normal`, `3`/`high`, `4`/`"very high"`, `5`/`critical`, case-insensitive) or Zammad's canonical string form (`"1 low"`, `"2 normal"`, etc.) — both are normalized to the canonical form before the API call. **This normalization is required**: Zammad's ticket API silently ignores an unrecognized priority string rather than erroring, leaving the ticket at its default priority (`"2 normal"` on create, unchanged on update) with no error surfaced. Before this fix, passing `priority: high` or `priority: 3` produced a normal-priority ticket with no indication anything was wrong.

---

## zen_stack_radar — Lens Bus Stack Provider

`zen_stack_radar` is the Lens Bus stack adapter for Zammad. It maps Library verbs to servicedesk modes and is the correct path when another tool (Library, Room Manager, etc.) needs ticket evidence by HA anchor.

**Do not call `zen_stack_radar` directly for ticket operations.** Use `zen_dojotools_servicedesk` for that.

### Lens Contract

| Property | Value |
|----------|-------|
| Stack key | `radar` |
| Consumes | `label`, `person`, `area_id`, `ticket` |
| Returns | `ticket_evidence` |
| Security | `r-only` — redacted read evidence only; mutations stay on `zen_dojotools_servicedesk` |
| Failure policy | `soft` |
| Content policy | `redacted_by_default` |

### Using via Library `stack=radar`

```yaml
zen_dojotools_library:
  section: stacks
  stack: radar
  mode: by_anchor
  input_json: '{"anchor_ids": ["kitchen"], "anchor_type": "area_id"}'
```

**Mode map** (Library verb → servicedesk mode):

| Library mode | Servicedesk mode |
|-------------|-----------------|
| `get` | `ticket_get` |
| `find` | `ticket_find` |
| `list` | `ticket_list` |
| `by_anchor` | `tickets_by_anchor` |
| `configure` | `configure` |

### Ticket evidence shape

```json
{
  "id": "<Zammad ticket id>",
  "title": "<ticket title>",
  "state": "<ticket state>",
  "anchor_contexts": [{"type": "<anchor_type>", "id": "<slug>"}]
}
```

---

## Agent Queue Sync

An automation (`zen_agent_queue_sync`) polls Zammad on a configurable interval (default 10 minutes) for tickets tagged `agent_queue` + the active persona slug with state `new`. Results are written to the household cabinet under `_zen_agent_queue_<slug>`. The interval is tunable via `_agent_queue_interval_minutes` in the household cabinet (minimum 1 minute). A `zen_event` with `kind: agent_queue_refresh` forces an immediate sync.

---

## First-Time Setup

1. Add the Zammad API token to `secrets.yaml`. The secret stores the full Authorization header value, including the `Token ` prefix:

```yaml
zammad_token: "Token YOUR_ZAMMAD_API_TOKEN"
```

2. Run `configure` to write the Zammad URL to the household cabinet and register the Lens stack provider:

```yaml
zen_dojotools_servicedesk:
  mode: configure
  input_json: '{"url": "https://your-zammad-host"}'
```

A successful configure call connects to Zammad, writes the URL to `integrations_config.fulfillment.url` in the household cabinet, and registers `zen_stack_radar` in the Lens registry.

3. Expose `zen_dojotools_servicedesk` to the conversation agent (MCP). Do not expose `zen_sutra_zammad` or `zen_stack_radar`.

4. Tag Zammad tickets with HA entity slugs (labels, area IDs, person slugs) to make them discoverable via `tickets_by_anchor` and the Lens Bus.

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| `configure` returns connection error | Verify the URL and `zammad_token` secret. Use HTTPS. |
| `tickets_by_anchor` returns empty results | Confirm tickets are tagged with the correct HA slug. Use `ticket_tag` to add slugs. |
| `view_get` returns 401/403 | Check token expiry and group/ticket/search permissions in Zammad. |
| Agent queue is stale | Fire a `zen_event` with `kind: agent_queue_refresh` to force sync. |

---

## Version History

| Version | Change |
|---------|--------|
| v0.4.0 (2026.7.1 fix) | `ticket_create`/`ticket_update` priority normalization — bare numbers/words (`high`, `3`) now map to Zammad's canonical `"3 high"` form; previously silently ignored by the Zammad API, leaving tickets at default priority with no error. `zen_stack_radar` lens manifest: `required_labels` changed `["radar"]` → `["zammad"]`, `radar` moved to optional alongside `servicedesk`/`tickets`. |
| v0.4.0 (Radar) | `radar_setup` mode: idempotent init wizard (find-or-creates house/zenos personas, writes `radar_defaults` to household cabinet). `ticket_assign` mode. `batch_update` mode. `object_attribute_create` mode. `reporter` shorthand on `ticket_create`: `house`=household persona, `zenos`=dev persona. `ticket_id` accepts display number (e.g. `10073`) or raw DB id. |
| v0.3.1 | Baseline Radar: `configure`, `ticket_create/get/find/list/search/update/close/complete`, `article_add/list`, `triage_set/get`, `tickets_by_anchor`, `ticket_link/unlink`, `customer_find/create`, `org_find/create`, `queue_get`, `view_get`, `workflow_context_get`. `zen_stack_radar` Lens Bus registration. |
