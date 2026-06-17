# Zen DojoTools Office — v5.1.0 (ZenOS-AI 2026.6.0 'Clue')

*M365 Teams and Mail tools for Home Assistant*

---

## Overview

`dojotools_office.yaml` contains two Microsoft 365 scripts: `zen_dojotools_teams` and `zen_dojotools_mail`. Both follow the standard DojoTools multitool pattern and return all responses as structured JSON.

Calendar (`zen_dojotools_calendar`) and Todo (`zen_dojotools_todo`) were split out to their own files in 2026.6.0 'Clue' and are no longer part of this module.

---

## zen_dojotools_teams

M365 Teams CRUD via the MS365 integration. Supports reading the latest incoming chat message, sending messages to a chat thread, setting your Teams presence status, and help.

### Modes

| `action_type` | Description |
|---|---|
| `read` | Returns latest chat message, your current Teams status, and the partner's chat ID |
| `send` | Sends a text message to an existing chat thread. Requires `chat_id` and `message` |
| `set` | Updates your Teams presence (availability + activity + expiration). Requires `availability` |
| `help` | Returns capability summary, field reference, and setup notes |

Update and delete are not supported by the MS365 Teams integration.

### Key Fields

| Field | Required | Description |
|---|---|---|
| `action_type` | Yes | `read`, `send`, `set`, `help` (default: `read`) |
| `chat_id` | send only | Target chat thread ID. Get it from a `read` response (`partner_chat_id`) |
| `message` | send only | Text to send |
| `availability` | set only | `Available`, `Busy`, `Away`, `DoNotDisturb` |
| `activity` | set optional | Extended status. Invalid pairings with `availability` are auto-filtered |
| `expiration` | set optional | ISO 8601 duration. Default: `PT5M` (5 min). Range: 5–240 min |

### Setup Helpers

| Helper | Purpose |
|---|---|
| `input_text.zen_teams_chat_id` | Your Teams chat thread ID. Run `read` — `partner_chat_id` in the response is the value to set here |
| `input_text.zen_teams_display_name` | Your display name as seen in Teams (e.g. `Home Assistant`) |

---

## zen_dojotools_mail

M365 Mail CRUD via the MS365 integration. Supports listing inbox messages, reading a specific message by UID, and sending new mail. Includes a whitelist gate on outbound sends.

### Modes

| `action_type` | Description |
|---|---|
| `list` | Lists messages from a folder (default: Inbox). Supports `from` and `query` filters |
| `read` | Fetches a single message by `uid`. Requires `folder` and `uid` |
| `create` | Sends a new email. Requires `to`, `subject`, `body` |
| `help` | Returns capability summary, field reference, examples, and setup notes |

Delete and restore are declared in the field selector but not implemented — `uid` required errors gate them. Attachments are listed on `read` but not retrievable.

### Key Fields

| Field | Required | Description |
|---|---|---|
| `action_type` | Yes | `list`, `read`, `create`, `help` (default: `help`) |
| `folder` | read/list | Folder name. Valid: `Inbox`, `Sent Items`, `Deleted Items`, `Junk Email`, `Outbox`. Default: `Inbox` |
| `uid` | read | Message UID from a `list` response |
| `to` | create | Recipient(s), comma-separated |
| `subject` | create | Subject line |
| `body` | create | Message body |
| `cc` / `bcc` | optional | Additional recipients, comma-separated |
| `importance` | optional | `Low`, `Normal`, `High`. Default: `Normal` |
| `content_type` | optional | `Text` or `HTML`. Default: `Text` |
| `from` | list filter | Filter by sender address |
| `query` | list filter | Search string for subject/body/sender |

### Whitelist Gate

Outbound sends are gated by `input_text.zen_mail_whitelist`. If the helper is missing or unavailable, sends are blocked and the AI surfaces setup instructions.

| Whitelist value | Behavior |
|---|---|
| `OFF` | All recipients allowed |
| `*@yourdomain.com` | Domain wildcard — any address in that domain |
| `user@domain.com` | Exact match only |

### Setup Helpers

| Helper | Purpose |
|---|---|
| `input_text.zen_mail_sender` | HA send-from address (e.g. `homeassistant@yourdomain.com`) |
| `input_text.zen_mail_domain` | Your email domain for wildcard matching |
| `input_text.zen_mail_whitelist` | Whitelist mode. Required for sends to proceed |

---

## Dependencies

| Dependency | Purpose |
|---|---|
| MS365 integration | Teams and Mail data source and action target |
| `sensor.homeassistant_chat` | Teams chat data |
| `sensor.homeassistant_status` | Teams presence state |
| `sensor.ms365_inbox` | Mail inbox data |
