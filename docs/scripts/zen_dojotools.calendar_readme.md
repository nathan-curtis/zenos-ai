# **Zen DojoTools Calendar — Version 1.10.3**

*Modular Calendar Management Tool for Home Assistant*

---

## **Overview**

`zen_dojotools_calendar` is a unified Calendar orchestration layer for Home Assistant.
It provides **deterministic, safe, and label-aware calendar operations** — including:

* Event creation
* Event read + multi-calendar aggregation
* Event inspection (via DojoTools Inspect utility)
* Event update (delete+create strategy)
* Event delete (provider-verified)

The tool is designed as a **fully JSON-driven conversational endpoint**, intended for use by Friday, Veronica, or any LLM-driven subsystem within ZenOS-AI.

Version **1.10.3** introduces **major safety hardening**, expanded targeting logic, deterministic entity resolution, and completely unified response envelopes for *every* exit path.

---

# **Key Features in 1.10.3**

### ✅ **Label-Based Calendar Targeting**

* A new `label_targets` input allows calendars to be selected using labels rather than entity_ids.
* Supports multiple labels and resolves them into a consistent list of calendar entities.

### ✅ **Deterministic Entity Resolution**

* Friendly name fallback resolution.
* Lowercase + whitespace normalization.
* Guaranteed resolution into a single `calendar.<entity>` string when possible.

### ✅ **Multi-Calendar Read Aggregation**

* Reads all events across:

  * Label-targeted calendars
  * A single specific calendar
  * Or all calendars when `calendar_name="*"`

Includes:

* Unified event metadata
* Creation/update timestamps
* Optional event_id when provider supports it

### ✅ **Inspect Integration**

* Direct integration with `script.dojotools_zen_inspect`
* Used to extract event metadata including `event_id` where supported
* Enables visibility into provider features and raw source details

### ✅ **Strict Safety for Mutations (create/update/delete)**

Mutating actions require:

* Exactly **one** target calendar
* **event_id** for update/delete
* Provider support for delete or update
* Deterministic date ordering (`start < end`)

### Update & Delete Operations

Because many calendar providers **do not expose event_id**, the following rules apply:

| Provider          | Event ID Exposed? | Update/Delete Allowed? |
| ----------------- | ----------------- | ---------------------- |
| MS365             | ✔ Yes             | ✔ Full support         |
| Google            | ❌ No              | ❌ Blocked safely       |
| ICS / CalDAV      | ❌ No              | ❌ Blocked safely       |
| Local HA Calendar | ❌ No              | ❌ Blocked safely       |

LLMs receive structured error responses with instructions for remediation.

---

# **Response Format (Unified JSON)**

Every code path returns the following minimal envelope:

```json
{
  "status": "<success|error|info|delete triggered>",
  "message": "...",
  "results": [ ... ],      // for reads
  "event": { ... },        // for create/update
  "inspect": { ... },      // for provider safety output
  "note": "..."            // optional metadata
}
```

This guarantees that Friday, Kronk, and downstream systems always receive an unambiguous result.

---

# **Input Fields**

| Field           | Required      | Description                                             |
| --------------- | ------------- | ------------------------------------------------------- |
| `action_type`   | ✔             | `read`, `create`, `update`, `delete`, `inspect`, `help` |
| `calendar_name` | depends       | Friendly name, entity_id, or "*"                        |
| `summary`       | create/update | Event summary/title                                     |
| `start`         | optional      | ISO string or date; defaults to today @ 00:00           |
| `end`           | optional      | ISO string or date; defaults to tomorrow @ 00:00        |
| `description`   | optional      | Text body                                               |
| `location`      | optional      | Event location                                          |
| `attendees`     | optional      | Comma-separated list                                    |
| `event_id`      | update/delete | Required for provider-safe mutations                    |
| `label_targets` | optional      | Label-based targeting                                   |

---

# **Action Behavior**

---

## **📘 Help**

Returns module capabilities, defaults, caveats, and provider notes.

```yaml
action_type: help
```

---

## **📖 Read Events**

Reads events from one or more calendars.

Supports:

* Label target aggregation
* Single calendar read
* Full calendar list when using `"*"`

---

## **🔍 Inspect**

Uses the DojoTools Inspect tool to retrieve:

* Raw provider metadata
* Event IDs (when supported)
* Feature flags

This enables downstream tools to determine whether safe deletes or updates are possible.

---

## **🟢 Create Event**

Creates an event on one explicit calendar.

Safety:

* Requires exactly one target calendar
* Blocks ambiguous or multi-calendar targets
* Validates date ordering

Supports:

* ISO timestamps
* All-day events (date-only inputs)
* Description, attendees, location

---

## **🟠 Update Event**

Implements delete→create pattern.

* Valid only when provider exposes `event_id`
* Prevents accidental overwrite due to summary/date inference
* Returns success + warning message when old event not found

---

## **🔴 Delete Event**

Deletes event by exact provider `event_id`.

Safety:

* Checks provider delete support via `supported_features`
* Blocks delete when:

  * provider lacks support
  * event_id is missing
  * multiple calendars match

Returns:

* `"delete triggered"` for consistent downstream messaging
* Includes inspect output when blocking action

---

# **Timestamp Normalization**

Because calendar providers return timestamps in **mixed UTC and local offset formats**, 1.10.3 ensures:

* All outgoing timestamps are produced in **local offset ISO format**
* Incoming mismatch handling is **explicitly documented**
* UAT should normalize timestamps externally if absolute equality is required

This enables deterministic outputs across all backends.

---

# **Error Handling**

Every failure path returns structured JSON with guidance:

Examples:

```json
{
  "status": "error",
  "message": "Multiple calendars match the query. Provide a single calendar for this action."
}
```

```json
{
  "status": "error",
  "message": "Delete/update requires 'event_id'. Provider did not expose event_id; operation safely blocked."
}
```

---

# **Provider Notes**

### ⚠ Google, ICS, Local Calendar

Do **not** expose event_id
→ Update/Delete blocked safely

### ✔ Microsoft 365

Full support
→ event_id exposed
→ safe mutation ops enabled

---

# **Version History**

### **1.10.3**

Major safety and determinism update.

Changes include:

* Added label_targets + label_target_entities
* Added multi-target block (`target_is_multi`)
* Rewrote entity resolution logic
* Unified JSON envelopes for all branches
* Multi-calendar event aggregation
* Integrated inspect path
* Added explicit safety checks for provider capabilities
* Rewrote update to delete→create flow
* Normalized timestamp formatting (ISO with offset)
* Improved help output with provider notes
* Cleaned branching logic, alignment, error structure

---

# **Dependencies**

| Module                                  | Purpose                                         |
| --------------------------------------- | ----------------------------------------------- |
| `dojotools_zen_inspect`                 | Inspect calendar entity for metadata + event_id |
| Home Assistant `calendar.*` integration | Calendar data source                            |
| MS365 calendar integration              | Delete/update provider                          |
