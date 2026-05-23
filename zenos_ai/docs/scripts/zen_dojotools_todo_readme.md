# Zen DojoTools ToDo — v2.2.0

**File:** `packages/zenos_ai/dojotools/dojotools_todo.yaml`
**Script:** `zen_dojotools_todo`
**MCP-exposed — HA to-do list CRUD**

Wraps HA todo services with `continue_on_error` isolation so auth failures (401s from MS365, Google, or other integrations) return clean JSON errors instead of crashing the pipeline. Resolves list names and label targets automatically.

---

## Actions

| Action | Description |
|--------|-------------|
| `read` | Read items from one or all lists. Omit `list_name` or pass `*` for wildcard discovery. |
| `create` | Create one or more items. Accepts strings or `{item, due_date, description, reminder}` objects. |
| `update` | Single-item full edit (rename, due_date, description, status) or bulk status update for multiple items. |
| `delete` | Delete item(s) by exact name. Uses `continue_on_error` — verify the list if auth is stale. |
| `help` | Return full field docs and examples. |

**Default:** `action_type: read`

---

## Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `action_type` | select | `read`, `create`, `update`, `delete`, `help`. Alias: `action`. |
| `list_name` | text | Friendly name or `todo.*` entity ID. Omit or `*` = wildcard (all lists). Alias: `list_id`. |
| `items` | list | Strings or `{item, status, due_date, description, rename}` objects. |
| `status` | text | `needs_action` (default) or `completed`. Bulk update: applies to all `items`. |
| `due_date` | text | ISO `YYYY-MM-DD`. Create or single-item update only. |
| `description` | text | Notes text. Create or single-item update only. |
| `rename` | text | New title. Single-item update only. |
| `label_targets` | list | HA label names — reads across all `todo.*` entities carrying those labels. Comma-separated or newline-separated string also accepted. |
| `reminder` | text | MS365 only: ISO datetime. Routes create through `ms365_todo.new_todo`. |
| `caller_token` | text | Opaque pass-through token. |

---

## Name Resolution

`list_name` accepts friendly name or `todo.*` entity ID. The script normalizes to `todo.<name_slugified>` and validates against live `states.todo`. If the entity doesn't exist, returns `{status: error, message: "list_not_found"}`.

**Label targeting:** `label_targets` fans out to all `todo.*` entities carrying those HA labels. Useful for reading across a logical group of lists (e.g. all household chores lists tagged `zen_chores`).

---

## `continue_on_error` Behavior

All write actions (`create`, `update`, `delete`) use `continue_on_error: true` on the underlying HA service calls. This means:

- Auth failures (401s from expired MS365/Google tokens) return a clean `{status: error, message: "auth_failure"}` response rather than crashing the calling script.
- If an item fails mid-batch, remaining items still attempt to execute.
- After an auth failure, verify the list manually — items may not have persisted.

**To fix auth failures:** re-authenticate via **Settings → Integrations** for the affected integration.

---

## MS365 Support

When `list_name` resolves to an entity in the `ms365` integration, `create` routes through `ms365_todo.new_todo` if a `reminder` field is provided. Standard HA `todo.add_item` is used otherwise.

---

## Examples

```yaml
# Read all lists
action_type: read

# Read one list
action_type: read
list_name: Household Chores

# Read across label-tagged lists
action_type: read
label_targets: [zen_chores]

# Create an item with due date
action_type: create
list_name: Household Chores
items:
  - Take out trash
due_date: "2026-05-25"

# Bulk complete multiple items
action_type: update
list_name: Household Chores
items:
  - Task A
  - Task B
status: completed

# Single-item rename + update
action_type: update
list_name: Household Chores
items:
  - item: "Old task name"
    rename: "New task name"
    due_date: "2026-06-01"

# Delete
action_type: delete
list_name: Household Chores
items:
  - Old task
```

---

## Error Responses

| Error | Cause | Fix |
|-------|-------|-----|
| `list_not_found` | `list_name` doesn't match any `todo.*` entity | Check spelling; wildcard read returns all valid names |
| `auth_failure` | Integration token expired (401) | Re-authenticate via Settings → Integrations; verify list after write |
| `bulk_complex` | Bulk update with multi-field edit attempted | Send one item at a time for multi-field edits |
