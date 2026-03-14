# Zen DojoTools Identity — 4.2.0

*Resolve user and construct identities within the ZenOS-AI household*

---

## Overview

`zen_dojotools_identity` is the identity resolver for ZenOS-AI. When called, it looks up a registered user, AI construct, or cabinet in the household roster and returns the corresponding identity record.

Called with no target, it returns the full redacted roster of all registered identities. Called with a specific target, it returns that identity's record.

The resolver is **MCP-exposed** and stateless — it makes no writes, fires no events, and has no side effects.

> **Note on scope:** The identity system currently handles resolution only — looking up who something is and returning their record. Privilege enforcement, session tokens, and security masking are planned post-GA features. The resolver is the foundation; the gatekeeper is coming.

---

## Input Fields

| Field | Type | Status | Description |
|---|---|---|---|
| `user_label` | text | Active | Label referencing a registered ZenOS-AI user |
| `user_cabinet` | entity (sensor) | Planned | Cabinet sensor entity_id |
| `user_entity_id` | entity (person) | Planned | HA person entity_id |
| `user_guid` | text | Planned | ZenOS-AI user GUID (UUID format) |

All fields are optional. Omit all inputs to retrieve the full roster.

When multiple inputs are provided, only one is used — priority order:

```
user_label → user_cabinet → user_entity_id → user_guid
```

`user_label` is the primary and recommended input path. The other three inputs are defined and validated but not yet functional.

---

## Resolution Flow

1. **Input selection** — the first non-empty field in priority order becomes the target
2. **Normalization** — the target is cleaned and classified:
   - `person.*` → preserved as person entity
   - `sensor.*` → preserved as cabinet sensor
   - UUID format → preserved as GUID
   - Plain string → treated as a label
   - Empty / whitespace / invalid → `None` (returns full roster)
3. **Lookup** — delegates to `zen_cabinets(target)` in `zen_os_1rc.jinja`
4. **Response** — returns the identity record(s) with a timestamp

---

## Usage

### Full roster (no target)

```yaml
# No inputs — returns all registered identities
```

```json
{
  "result": [
    { "name": "Nathan", "role": "head_of_household", ... },
    { "name": "Friday", "type": "ai_construct", ... }
  ],
  "target_normalized": null,
  "timestamp": "2026-03-14T12:00:00.000000+00:00"
}
```

### Lookup by label

```yaml
user_label: nathan
```

```json
{
  "result": { "name": "Nathan", "role": "head_of_household", ... },
  "target_normalized": "nathan",
  "timestamp": "2026-03-14T12:00:00.000000+00:00"
}
```

### Not found

```json
{
  "result": [],
  "error": "not_found"
}
```

---

## Response Format

| Field | Description |
|---|---|
| `result` | Identity record (object) for single lookup, array for full roster, empty array on error |
| `target_normalized` | The resolved target after normalization (`null` for full roster) |
| `timestamp` | ISO 8601 response timestamp |
| `error` | `"not_found"` or `"empty_input"` on failure (replaces result fields) |

---

## What's Coming Post-GA

The identity module is the foundation for the ZenOS-AI security model. Once the resolver is stable, the planned additions are:

- **Session tokens** — issued per interaction, tied to caller identity
- **Security masks** — every tool call will pass through the identity resolver to receive a privilege mask before execution
- **ACL enforcement** — cabinet-level access control based on identity role and relationship
- **Privilege gating** — constructs can only see and act within their authorized scope

The current resolver is the first layer. The gatekeeper builds on top of it.

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `zen_os_1rc.jinja` → `zen_cabinets()` | Cabinet-backed identity lookup |
| Zen AI User Cabinet | Identity record storage |
