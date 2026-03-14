# Zen DojoTools Profile Editor — 4.2.0

*Read and write identity profiles for AI personas, households, users, and families*

---

## Overview

`zen_dojotools_profile_editor` is the universal read/write interface for ZenOS-AI identity cabinets. It is **MCP-exposed** and called by Friday during OOBE and on user request.

Four cabinet types are supported:

| Type | What It Stores | Default Cabinet |
|---|---|---|
| `ai_user` | AI persona identity — name, voice, appearance, familiar | `zen_default_ai_user_cabinet` |
| `household` | Address, timezone, contact info | `zen_default_household_cabinet` |
| `user` | Household member profiles | `zen_default_user_cabinet` |
| `family` | Extended family / non-resident profiles | `zen_default_family_cabinet` |

Writes are **non-destructive by default** — existing field values are never overwritten unless `force: true` is passed. Call `mode: read` before writing to see what's already set.

This tool is intentionally AI-accessible. Expose it to your conversation agent and Friday can help you configure your household, update persona details, and fill in profile fields conversationally — no YAML required. If you don't expose it, all profile edits are manual.

---

## Input Fields

### Core Control

| Field | Type | Default | Options | Description |
|---|---|---|---|---|
| `mode` | select | `help` | `help`, `read`, `write` | Operation to perform |
| `target_type` | select | `ai_user` | `ai_user`, `household`, `user`, `family` | Which cabinet type to target |
| `target` | entity (sensor) | — | Any cabinet sensor | Specific cabinet; omit to use the default for `target_type` |
| `force` | boolean | `false` | — | Overwrite existing field values when `true` |

---

### AI User Fields (`target_type: ai_user`)

These fields compose the persona's **Essence** — the structured identity object Friday reads at inference time.

| Field | Maps To | Description |
|---|---|---|
| `persona_name` | `identity.name` | The AI's name (e.g., "Friday") |
| `primary_user` | `identity.primary_user` | Primary user this persona serves |
| `pronouns` | `identity.pronouns` | Pronoun set (e.g., "she/her") |
| `motif` | `identity.motif` | Core personality note (e.g., "quiet brilliance") |
| `vibe` | `archetype.vibe` | Personality vibe (e.g., "calm-confident") |
| `voice_tone` | `voice.tone` | Tone of voice (e.g., "warm") |
| `voice_style` | `voice.style` | Style of voice (e.g., "conversational") |
| `humor` | `culture.humor` | Humor type (e.g., "dry wit") |
| `selfie` | `visuals.selfie` | Visual self-description |
| `familiar_name` | `familiar.name` | Companion name (e.g., "Pip") |
| `familiar_type` | `familiar.type` | Companion type (e.g., "fox", "cat", "spirit") |
| `familiar_fx` | `familiar.fx` | Companion mannerism (e.g., "flicks tail when curious") |
| `room` | `environment.room` | The AI's conceptual space (e.g., "the observatory") |
| `pov` | `pov` | Narrative perspective (e.g., "third-person cinematic") |
| `essence_patch` | deep-merge into essence | JSON object for complex nested fields (furniture, decor, music, etc.) |

Patches are **leaf-level** — changing `voice.tone` never touches `voice.style`. `essence_patch` supports deep-merging complex nested structures without overwriting named fields.

---

### Household Fields (`target_type: household`)

| Field | Maps To | Description |
|---|---|---|
| `household_name` | `_household_profile.household_name` | Name of the household |
| `address` | `_household_profile.address` | Street address |
| `city` | `_household_profile.city` | City |
| `state` | `_household_profile.state` | State or region |
| `zip` | `_household_profile.zip` | ZIP or postal code |
| `country` | `_household_profile.country` | Country |
| `phone` | `_household_profile.phone` | Primary contact number |

`timezone` is auto-detected from HA's local timezone on first write and never overwritten.

---

### User / Family Fields (`target_type: user` or `family`)

| Field | Maps To | Description |
|---|---|---|
| `name` | `_user_profile.name` | Display name |
| `first_name` | `_user_profile.first_name` | First name |
| `last_name` | `_user_profile.last_name` | Last name |
| `preferred_name` | `_user_profile.preferred_name` | Goes-by name |
| `pronouns` | `_user_profile.pronouns` | Pronoun set |
| `role` | `_user_profile.role` | Role in household (`head_of_household`, `partner`, `child`, `guest`) |
| `phone` | `_user_profile.phone` | Phone number |
| `email` | `_user_profile.email` | Email address |
| `birthday` | `_user_profile.birthday` | Birthday (YYYY-MM-DD) |
| `preferences` | `_user_profile.preferences` | JSON object of user preferences — merged, not overwritten |

`read` mode also returns the read-only relationship arrays: `partners`, `ai_partners`, `children`.

---

## Modes

### `help`

Returns the full field reference with example values for each target type. Safe to call any time.

```yaml
mode: help
```

---

### `read`

Returns the current profile for the specified cabinet type.

```yaml
mode: read
target_type: household
```

```json
{
  "status": "ok",
  "mode": "read",
  "target_type": "household",
  "target": "sensor.zenos_default_household_cabinet",
  "profile": {
    "household_name": "The Curtis House",
    "city": "San Antonio",
    "state": "TX",
    "timezone": "CST"
  }
}
```

Always read before writing — the non-destructive default means you need to know what's already set before using `force: true`.

---

### `write`

Patches the specified profile. Only non-empty fields in the call are written. Existing values are skipped unless `force: true`.

```yaml
mode: write
target_type: ai_user
persona_name: Friday
pronouns: she/her
voice_tone: warm
voice_style: conversational
humor: dry wit
```

```json
{
  "status": "ok",
  "mode": "write",
  "target_type": "ai_user",
  "target": "sensor.zenos_default_ai_user_cabinet",
  "message": "ai_user profile written."
}
```

---

## Safety Rules

**Non-destructive by default** — empty inputs are ignored; existing non-empty values are skipped unless `force: true` is passed.

**`force: true` behavior** — overwrites every field included in the call where the input is non-empty. Fields not included in the call are always left untouched.

**JSON fields** (`essence_patch`, `preferences`) — parsed with `from_json`. If parsing fails or the result is not an object, the field is silently skipped.

**`timezone`** (household) — auto-set on first write, never overwritten.

**Target validation** — if no cabinet resolves for the given `target_type`, the script stops with an error before any write occurs.

---

## Examples

### Set the AI persona's name and voice

```yaml
mode: write
target_type: ai_user
persona_name: Friday
voice_tone: warm
voice_style: conversational
```

### Give Friday a familiar

```yaml
mode: write
target_type: ai_user
familiar_name: Pip
familiar_type: fox
familiar_fx: flicks tail when curious
```

### Add complex visual detail via essence_patch

```yaml
mode: write
target_type: ai_user
essence_patch: >
  {
    "visuals": {
      "furniture": ["worn leather chair", "stacked books"],
      "decor": ["brass compass", "star charts"]
    }
  }
```

### Initialize household location

```yaml
mode: write
target_type: household
household_name: The Curtis House
city: San Antonio
state: TX
country: US
```

### Add a user profile

```yaml
mode: write
target_type: user
first_name: Nathan
preferred_name: Nathan
role: head_of_household
pronouns: he/him
```

### Overwrite an existing field

```yaml
mode: write
target_type: ai_user
voice_tone: dry
force: true
```

---

## Response Format

All modes return a consistent JSON envelope:

```json
{
  "status": "ok | error",
  "mode": "<mode>",
  "target_type": "<type>",
  "target": "<cabinet entity_id>",
  "profile": {},    // read mode only
  "message": "..."  // write and error modes
}
```

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `script.zen_dojotools_filecabinet` | All cabinet reads and writes |
| `zen_os_1rc.jinja` | `essence_defaults()` macro — baseline for ai_user essence assembly |
| `zen_default_ai_user_cabinet` label | Default cabinet resolution for `ai_user` |
| `zen_default_household_cabinet` label | Default cabinet resolution for `household` |
| `zen_default_user_cabinet` label | Default cabinet resolution for `user` |
| `zen_default_family_cabinet` label | Default cabinet resolution for `family` |
