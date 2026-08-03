# ZenOS-AI Rolodex (Twenty CRM) Plugin

**Version:** 1.10.0  
**Package:** `packages/zenos_ai/plugins/twenty/twenty.yaml`  
**Primary script:** `zen_dojotools_rolodex`  
**Internal REST broker:** `zen_sutra_twenty`  
**Internal stack provider:** `zen_stack_crm`

---

## Overview

Rolodex is the contact and company CRM surface for ZenOS-AI, backed by the Twenty CRM REST API. It covers:

- Contact and company lookup, create, and note-taking
- Guest stays as structured Twenty objects (`stay`) — area-validated, overlap-warned, with auto-dispatched turnover (and hot-tub recheck) on checkout
- Mid-stay housekeeping tasks (`stay_amenity_add`) distinct from checkout turnover
- Vendor appointments as structured Twenty objects (`appointment`) — full lifecycle (start/complete/no_show/cancel) with completion auto-writing to vendor history
- Guest service requests (`serviceRequest`) — shape-first routed to a Grocy chore, Radar ticket, or base To Do depending on request type
- Guest allergen/prefs storage (`guest_prefs_set`) and a shared "who's staying/servicing this room right now" lookup (`active_stay_prefs`, `active_appointment`) consumed by Kitchen's allergen flagging and Room Manager's vendor-activity signal
- Simple booking events on HA calendars (`appointment_create`/`appointments_list`, `stays_create`/`stays_list`) — the calendar-only path, now with real `area_id` room tagging
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

These modes route to HA calendar and todo entities discovered at call time by label. This is the simple calendar-event path — see [Structured Guest Stays](#structured-guest-stays) and [Structured Vendor Appointments](#structured-vendor-appointments) below for the richer, area-validated Twenty-object path.

| Mode | Description | Required |
|------|-------------|----------|
| `stays_create` | Create a guest stay event on the `crm_stays` calendar. Optional `area_id` tags it to a room (`ha_area:` token in the description) — enables `active_stay_prefs`/room-scoped lookups for that room. | `summary`, `start_date` |
| `stays_list` | List stay events. Filters by `area_id` if given. | — |
| `appointment_create` | Create a booking on the `crm_appointments` calendar. Optional `area_id` tags it to a room (same `ha_area:` convention as `stays_create`) — enables `active_appointment`/vendor-activity lookups for that room. | `summary`, `start_date` |
| `appointments_list` | List appointments. Optional date range. Filters by `area_id` if given. | — |
| `reminder_set` | Create a reminder todo linked to a contact/company | `summary` |
| `reminders_list` | List reminders. Optional filter by `item_id`. | — |
| `task_link` | Create a task todo linked to a contact/company | `summary` |
| `tasks_list` | List tasks. Optional filter by `item_id`. | — |

---

## Structured Guest Stays

`stay_*` modes operate on Twenty's native `stay` object — a real record with lifecycle state, not just a calendar entry. This is the path to use for anything that should drive downstream automation (turnover chores, allergen-aware housekeeping, room occupancy checks).

| Mode | Description | Required |
|------|-------------|----------|
| `stay_create` | Create a stay. `area_id` is hard-validated against real HA areas. An overlapping active stay in the same room warns by default; set `block_on_overlap: true` to hard-block instead. | `item_id` (guest contact UUID — resolve via `contact_find`/`contact_create` first), `area_id`, `start_date` |
| `stay_get` | Fetch a single stay record. | `item_id` |
| `stays_active_list` | Front-desk view — every stay not yet `RELEASED` (`RESERVED` + `ACTIVE`). Optional `area_id` scopes to one room. | — |
| `stay_release` | Checkout. Idempotent — a stay already `RELEASED` with `turnoverSpawned: true` is a no-op, not a re-fire. Always dispatches a baseline turnover chore via Taskmaster (upgraded to a deep-clean variant on allergen-flagged guests or stays of 3+ nights); no Radar tickets are auto-created — humans escalate anything found. Also checks whether the hot tub was active during the stay window and, if so, dispatches a separate "hot tub recheck" chore. Does not touch the guest contact or any chores already spawned by a prior release. | `item_id` |
| `stay_delete` | Delete a stay record. Twenty soft-deletes to trash (recoverable there for a retention window), not an immediate hard purge. | `item_id` |
| `stay_amenity_add` | Mid-stay housekeeping task (e.g. "Fresh towels", "Welcome basket") — a dated, location-tied task distinct from the checkout turnover. Use `amenity` for the task title and `day_offset` (days after check-in, `0` = day of check-in) for timing. | `item_id`, `amenity` |

---

## Structured Vendor Appointments

`appt_*` modes operate on Twenty's native `appointment` object with a full `SCHEDULED → ACTIVE → COMPLETED/NO_SHOW/CANCELLED` lifecycle.

| Mode | Description | Required |
|------|-------------|----------|
| `appt_create` | Schedule a vendor appointment. `area_id` is hard-validated against real HA areas. An overlapping active appointment in the same room warns by default; set `block_on_overlap: true` to hard-block instead. | `item_id` (vendor contact/company UUID), `area_id`, `start_date` |
| `appt_get` | Fetch a single appointment record. | `item_id` |
| `appts_active_list` | Everything not yet terminal (not `COMPLETED`/`NO_SHOW`/`CANCELLED`). Optional `area_id` scopes to one room. | — |
| `appt_start` | Mark an appointment in progress. | `item_id` |
| `appt_complete` | Mark an appointment completed and auto-write a completion note to the vendor's `vendor_history` (create-then-link via `noteTargets`, same pattern `contact_note` uses). Set `requires_followup: true` if the vendor did not fully resolve the issue — flags both the appointment and the vendor_history note. | `item_id` |
| `appt_no_show` | Mark the vendor a no-show. | `item_id` |
| `appt_cancel` | Cancel a scheduled appointment. | `item_id` |
| `appt_delete` | Delete an appointment record. | `item_id` |

---

## Guest Service Requests

`service_request_*` modes log guest requests on Twenty's `serviceRequest` object, routed by shape through Taskmaster: `request_type: HOUSEKEEPING` fulfills via a Grocy chore, `MAINTENANCE` via a Radar ticket (when configured), and `CONCIERGE`/`OTHER` via general tracking (Radar ticket when configured, otherwise a base To Do). `area_id` is optional — not every request is room-scoped (Wi-Fi, front door code) — but is hard-validated against real HA areas when given.

| Mode | Description | Required |
|------|-------------|----------|
| `service_request_create` | Log a request. Response includes `linked_backend`/`linked_task_id` showing which system actually owns fulfillment. | `note_body` |
| `service_request_get` | Fetch a single request. | `item_id` |
| `service_requests_active_list` | Everything not yet `RESOLVED`. Optional `area_id` scopes to one room; house-wide (no-area) requests are always included. | — |
| `service_request_resolve` | Mark a request resolved. Does not auto-complete the underlying chore/ticket. | `item_id` |
| `service_request_delete` | Delete a request record. | `item_id` |

`request_type` options: `HOUSEKEEPING`, `MAINTENANCE`, `CONCIERGE`, `OTHER` (default `OTHER`).

---

## Guest Prefs and Occupancy Lookups

| Mode | Description | Required |
|------|-------------|----------|
| `guest_prefs_set` | Write a contact's `guestPrefs` field — a JSON string for allergens/music/spa/etc, extensible to any keys. Run `provision_schema` first. | `item_id`, `prefs` (JSON string) |
| `active_stay_prefs` | One-call "is a guest actively staying right now (optionally scoped to one room via `area_id`), and what are their prefs" — finds the stay whose start/end window contains the current time, resolves the linked contact, returns `guestPrefs`. Read-only. This is the shared lookup Kitchen's allergen-flagging builds on. | — |
| `active_appointment` | The appointment counterpart to `active_stay_prefs` — is a vendor/contact mid-appointment in this room right now (optional `area_id`)? Resolves the linked contact as a company first (typical vendor case), person as fallback. A caution signal for automations (Room Manager's `vendor_activity` field), not a prefs source — a vendor's presence doesn't change whose meal/music prefs apply. | — |

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
| `provision_schema` | Idempotent deploy of ZenOS CRM schema to Twenty — creates `haLabels` (TEXT) on People and Companies, `haPersonId` (TEXT) on People, and `guestPrefs` (TEXT, JSON string) on People if missing. Also idempotently creates the `stay`, `appointment`, and `serviceRequest` custom Twenty objects used by the structured guest-stay/vendor-appointment/service-request modes above. Stamps `crm_schema` to cabinet. Safe to re-run. |
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
| `block_on_overlap` | For `stay_create`/`appt_create`. Default `false` (warn but allow). Set `true` to hard-block creation when the room already has an overlapping active record. |
| `requires_followup` | For `appt_complete` only. Default `false`. Set `true` if the vendor did not fully resolve the issue. |
| `request_type` | For `service_request_create`. `HOUSEKEEPING` / `MAINTENANCE` / `CONCIERGE` / `OTHER` — determines fulfillment backend. Default `OTHER`. |
| `amenity` | For `stay_amenity_add` — the mid-stay task title, e.g. `"Fresh towels"`. |
| `day_offset` | For `stay_amenity_add` — days after check-in the task is due. `0` = day of check-in. |
| `prefs` | JSON string for `guest_prefs_set` — a contact's `guestPrefs` field. Extensible; any keys are stored as-is. |

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
