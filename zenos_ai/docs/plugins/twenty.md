# ZenOS-AI Rolodex (Twenty CRM) Plugin

**Version:** 1.7.0  
**Package:** `packages/zenos_ai/plugins/twenty/twenty.yaml`  
**Primary script:** `zen_dojotools_rolodex`  
**Internal REST broker:** `zen_sutra_twenty`  
**Internal stack provider:** `zen_stack_crm`

---

## Overview

Rolodex is the contact and company CRM surface for ZenOS-AI, backed by the Twenty CRM REST API. It covers:

- Contact and company lookup, create, and note-taking
- Guest stays and service appointments on HA calendars
- Reminders and tasks in HA todo lists
- Vendor service history and aggregated timeline
- HA label tagging on contacts and companies for label-based discovery
- Area vendor lookup via HA label matching
- HA person entity linking for live presence enrichment
- Schema-level Grocy↔CRM binding (`vendor_link`) via `crm_company_id` userfield on Grocy shopping lists and shopping locations

The plugin follows the same two-layer pattern as Grocy and Mealie:

| Layer | Role |
|-------|------|
| `zen_dojotools_rolodex` | MCP/LLM-facing helper — always call this |
| `zen_sutra_twenty` | Internal REST broker — stays internal |
| `zen_stack_crm` | Internal lens bus provider — stays internal |

---

## Setup

1. Add to `secrets.yaml`:

```yaml
twenty_bearer: "Bearer YOUR_TWENTY_API_KEY"
```

Generate an API key in Twenty under Settings > API & Webhooks.

2. Run configure to store the Twenty URL and register the CRM lens:

```yaml
zen_dojotools_rolodex:
  mode: configure
  config_json: '{"url": "http://your-twenty-host:3000"}'
```

The URL is stored in `household.integrations_config.twenty.url`. The legacy `input_text.twenty_url` helper is a fallback that remains until configure is run. (2026.7.1: the `integrations_config` drawer read is now guarded against the value coming back as a JSON-encoded string instead of an already-parsed mapping — matters if the drawer was written by an older tool version or an external cabinet edit.)

3. Label HA entities for property operations:

- Label a calendar entity with `crm_stays` and `crm_appointments`
- Label a todo entity with `crm_reminders` and `crm_tasks`

Default label slugs can be overridden in configure via `stays_label`, `appointments_label`, `reminders_label`, `tasks_label`.

4. Deploy the ZenOS CRM schema to Twenty (creates `haLabels` and `haPersonId` fields):

```yaml
zen_dojotools_rolodex:
  mode: provision_schema
```

---

## Contact Modes

| Mode | Description | Required |
|------|-------------|----------|
| `contact_find` | Search people by name or email | `item` |
| `contact_get` | Full contact record + recent notes. Enriches with HA presence if `haPersonId` is set. | `item_id` |
| `contact_create` | Create a new contact | `first_name` (or `last_name`) |
| `contact_note` | Add a note to a contact or company. Default target: person; set `note_target_type: company` for company notes. | `item_id`, `note_body` |
| `contact_link` | Wire a contact to a company with optional `role` + note in one call. Replaces three separate API calls. | `item_id`, `company_id` |

---

## Company Modes

| Mode | Description | Required |
|------|-------------|----------|
| `company_find` | Search companies by name | `item` |
| `company_get` | Company record + people list | `item_id` |
| `company_create` | Create a new company | `item` |
| `vendor_history` | List notes for a vendor/company (service history) | `item_id` |

---

## Property Operations

These modes route to HA calendar and todo entities discovered at call time by label.

| Mode | Description | Required |
|------|-------------|----------|
| `stays_create` | Create a guest stay event on the `crm_stays` calendar | `summary`, `start_date` |
| `stays_list` | List stay events. Optional area filter via `area_id`. | — |
| `appointment_create` | Create a booking on the `crm_appointments` calendar | `summary`, `start_date` |
| `appointments_list` | List appointments. Optional date range. | — |
| `reminder_set` | Create a reminder todo linked to a contact/company | `summary` |
| `reminders_list` | List reminders. Optional filter by `item_id`. | — |
| `task_link` | Create a task todo linked to a contact/company | `summary` |
| `tasks_list` | List tasks. Optional filter by `item_id`. | — |

---

## Tagging and Discovery

| Mode | Description | Required |
|------|-------------|----------|
| `tag_contact` | Apply ZenOS HA label slugs to a contact (`haLabels` field). Run `provision_schema` first. | `item_id`, `tags` |
| `tag_company` | Apply ZenOS HA label slugs to a company (`haLabels` field). Run `provision_schema` first. | `item_id`, `tags` |
| `contacts_by_label` | Find contacts by HA label slug | `item` (label slug) |
| `companies_by_label` | Find companies by HA label slug | `item` (label slug) |
| `area_vendors` | Find vendor companies + contacts associated with an HA area via `haLabels` matching | `area_id` |

Tags are comma-separated label slugs, e.g. `plumber,vendor,active`.

---

## Presence and Timeline

| Mode | Description | Required |
|------|-------------|----------|
| `person_link` | Link a Twenty contact to an HA person entity (`haPersonId`). Enables live presence enrichment in `contact_get`. Run `provision_schema` first. | `item_id`, `person_entity_id` |
| `people_home` | List HA persons in a zone (default `home`), enriched with linked Twenty contact where `haPersonId` matches. | — |
| `recent_activity` | Recent notes across all contacts | — |
| `timeline` | Aggregated view — stays, appointments, notes, reminders, tasks sorted by date. Optional: `item_id`, `start_date`, `end_date`, `limit`. | — |

---

## Schema and Integration

| Mode | Description |
|------|-------------|
| `configure` | Set Twenty URL + CRM routing labels in cabinet. With no `config_json`, inspects current config. |
| `provision_schema` | Idempotent deploy of ZenOS CRM schema to Twenty — creates `haLabels` (TEXT) on People and Companies, `haPersonId` (TEXT) on People if missing. Stamps `crm_schema` to cabinet. Safe to re-run. |
| `vendor_link` | Link a CRM company UUID to a Grocy `shopping_list` or `shopping_location` via `crm_company_id` userfield. Makes the vendor chain schema-level and permanent. Requires `item_id`, `grocy_entity` (`shopping_lists` or `shopping_locations`), `grocy_id`. |
| `stacks_by_anchor` | Lens bus provider entry point — returns bounded CRM evidence for a company or person anchor. Consumed by the ZenOS lens dispatch layer. `content_redacted: true` always. |

---

## Key Fields

| Field | Description |
|-------|-------------|
| `mode` | Operation name (see tables above) |
| `item` | Name for search, or company name for create |
| `item_id` | Twenty UUID — bypasses name resolution |
| `first_name` / `last_name` | Contact name for create |
| `email` / `phone` | Contact contact info |
| `company_id` | Twenty company UUID for contact linking |
| `note_title` | Note title (auto-generated from date if omitted) |
| `note_body` | Note body text |
| `note_target_type` | `person` (default) or `company` |
| `tags` | Comma-separated HA label slugs |
| `config_json` | JSON dict for configure — keys: `url`, `stays_label`, `appointments_label`, `reminders_label`, `tasks_label` |
| `area_id` | HA area ID for `area_vendors` or to tag a stay to a room |
| `person_entity_id` | HA person entity ID for `person_link` (e.g. `person.member_name`) |
| `role` | Job title / relationship role for `contact_link` |
| `grocy_entity` | `shopping_lists` or `shopping_locations` for `vendor_link` |
| `grocy_id` | Grocy record integer ID for `vendor_link` |
| `limit` | Max results (default 20, max 100) |

---

## Twenty API Quirks

These are documented in the plugin and matter if you ever call `zen_sutra_twenty` directly or debug raw responses:

- **Note target fields:** Twenty uses `targetPersonId` / `targetCompanyId` in `/noteTargets`, not `personId` / `companyId`.
- **Note body field:** Note body is `bodyV2` (RICH_TEXT) — pass as `{blocknote: null, markdown: "text"}`. The legacy `body` field does not exist.
- **Filter syntax:** `name[ilike]:%value%` — do not URL-encode the brackets.
- **REST path:** Use `/rest/{entity}` for both list and single record. The `/rest/core/` prefix is wrong.
- **Response shape:** Lists return `{ "data": { "companies": [...] }, "pageInfo": {...}, "totalCount": N }`. Single records return `{ "data": { ... } }`.

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| `not_configured` status on every call | Run `configure` with the Twenty URL |
| `auth_failed` (HTTP 401) | Check `twenty_bearer` in `secrets.yaml` — confirm token is valid and prefixed with `"Bearer "` |
| `non_json_response` | The base URL is returning a login or proxy page — check reverse proxy auth bypass and the `/rest` path |
| `not_found` (HTTP 404) | Wrong object path or UUID — confirm endpoint and ID |
| `tag_contact` / `tag_company` fails | Run `provision_schema` first to create the `haLabels` field in Twenty |
| `people_home` shows no CRM enrichment | Run `person_link` to bind HA person entities to Twenty contacts |
| Stays or appointments not routing to calendar | Confirm a calendar entity has the `crm_stays` or `crm_appointments` label applied |
