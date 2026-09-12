# ZenOS-AI Infrastructure Console

**Version:** 1.3.0 (header) / 1.4.2 (tool_manifest)
**File:** `dojotools/dojotools_infra.yaml`

**Entities:**
- `script.zen_dojotools_infra` — MOM/Splunk-style IT ops console (**MCP-exposed**)

**Related, not in this file:**
- `script.zen_admintools_portainer_acl` — the only writer of the container-control codex's config (`dojotools/dojotools_portainer_admin.yaml`, **not MCP-exposed**)
- `script.zen_root_portainer` — internal REST broker used by the codex (`plugins/portainer/portainer.yaml`, **not MCP-exposed**)

---

## Overview

`zen_dojotools_infra` is a unified, label-driven IT ops console: node/VM inventory, container/Portainer host health, Uptime Kuma service monitors, pending updates, SSL certificate expiry, HA supervisor health, and a log-tail peek. All discovery is by HA label or integration lookup — no hardcoded entity IDs.

As of v1.3.0, it also carries a **container-control codex**: an optional, admin-gated add-on that lets an agent read live container state from Portainer and, if configured and certified, restart/start/stop/remove specific containers. The codex is entirely inert (`not_configured`) until an admin sets a Portainer URL — the read/status modes below are unaffected either way.

---

## Read/Status Modes (pre-existing, unchanged)

| Mode | Description |
|---|---|
| `status` (default) | MOM-style top-line summary across all systems, with a `note` field calling out anything that needs attention |
| `nodes` | Proxmox VM inventory — state, role labels (`inference`/`stt`/`tts`/`home_assistant`), uptime |
| `containers` | Portainer endpoint health + host info (mem/CPU) + image/container counts, per endpoint |
| `monitors` | Uptime Kuma service health, response times, cert expiry (with a `cert_warn` flag) |
| `updates` | Pending container image updates (WUD sensors) + pending HA platform updates |
| `certs` | Full SSL certificate expiry table, sorted soonest-first |
| `ha` | HA supervisor health — OS/core/supervisor update flags, host disk, CPU/mem %, addon status |
| `log` | Quick peek: last 30 log lines + a search for start/stop/reload/ERROR/WARNING. For deep search or paging, use `zen_dojotools_ha_log_viewer` directly |
| `discover` | Dump raw label→entity bindings — debug/setup only |
| `help` | Full mode + field reference |
| `tool_manifest` | Standard manifest response (`mcp_exposed: true`, `risk_class: read_soft_plus_gated_container_control`) |

### Label Bindings

| Binding | Scope |
|---|---|
| `label:proxmox` | Proxmox VM binary sensors (any endpoint) |
| `label:portainer` | Full docker scope — endpoint sensors + WUD + labeled VMs |
| `integration:portainer` | Portainer HA integration sensors only (endpoint health, counts, host) |
| `label:kuma` | Uptime Kuma status/response/cert sensors |
| `label:updates` + `label:container` + `label:docker` | WUD update sensors |
| `label:supervisor` / `integration:hassio` | HA core/supervisor/OS/addon entities |
| `integration:ipp` / `label:printer` | Printer status + ink sensors (see `printers` block in `status`) |

Each binding tries the label first and falls back to the integration lookup only if the label returns nothing — so labeling entities is the preferred setup path, but not required.

### `filter` Field

`filter` (text) narrows `nodes`, `monitors`, and `updates` by a case-insensitive name substring. Not used by other modes.

---

## Container-Control Codex (new in v1.3.0)

An optional add-on bolted onto the same console script — same "inert unless configured" pattern used by lens providers elsewhere in this codebase. It is entirely gated off (`status: not_configured`) until an admin sets `integrations_config.portainer.url` via `zen_admintools_portainer_acl`.

### Modes

| Mode | Class | Gating |
|---|---|---|
| `containers_list` | read | None beyond "codex configured" — lists every container across all endpoints, each annotated `controllable: true/false` per the admin allow-list |
| `container_get` | read | Same as above, resolved/inspects a single container by name/substring |
| `container_restart` | x (write) | Allow-list + `infra_container_control` cert level ≥ `min_cert_level.x` (default 2) |
| `container_start` | x (write) | Same gating as `container_restart` |
| `container_stop` | d (destructive) | Allow-list + cert level ≥ `min_cert_level.d` (default 3) **plus** a live household-admin acknowledgment |
| `container_remove` | d (destructive) | Same gating as `container_stop` |

### Fields (codex-only)

| Field | Required for | Description |
|---|---|---|
| `container` | `container_get`, all `container_*` write modes | Name/substring to resolve. Must resolve to exactly one container or the call is denied (`ambiguous_or_not_found_container`) — no best-guess execution |
| `endpoint_id` | optional | Restrict to one Portainer endpoint; omit to search all endpoints |
| `caller_id` | optional | **Audit label only — not an authenticated identity.** See Identity Model below |
| `portainer_url` | optional, one-time | If the codex is unconfigured and this is passed, it self-provisions the connection URL (via `zen_admintools_portainer_acl` internally) and continues with the requested action in the same call. Only sets the URL — the controllable-containers allow-list stays empty until an admin explicitly grants it |

### Gate Sequence (write/destructive modes)

1. **Container resolution** — `container` must match exactly one container across the searched endpoint(s); ambiguous or zero matches are denied.
2. **Allow-list check** — the resolved container's name must match an admin-configured pattern in `controllable_containers`.
3. **Identity + certification** — `zen_dojotools_identity mode=resolve_caller_identity` is called with `required_cert: infra_container_control` and the mode's minimum level. Denied if `policy_status != allowed` or the resolved cert level is below the required minimum.
4. **(`container_stop`/`container_remove` only) Live household-admin acknowledgment** — delegates to the shared `zen_dojotools_identity mode=request_live_ack` chokepoint (2026-08-15; this exact fire+poll block used to be hand-rolled here, in `zen_dojotools_locks`, and twice in `zen_dojotools_persona_editor`, before consolidation). A decline or timeout denies the action (fail-closed). Note: the pre-consolidation default (`notify_target: mobile`) never actually dispatched — only `notify_target: postman` wires real yes/no capture — so every ack on this path had been failing closed by accident prior to the fix, not by design. An admin can exempt a specific container from this per-call ack via `cert_grant cert_component=infra_container_control cert_scope=["<container name>"]` — resolution goes through the shared `cert_scope_check` macro (`target_entity=<container name>` on `resolve_caller_identity`), same centralized mechanism locks/covers/security_manager/room_manager/ZenLux all use as of 2026-08-16.
5. **Execute** — POSTs the Docker action (`restart`/`start`/`stop`/`remove`) through `zen_root_portainer`, logs the outcome via `zen_dojotools_event_emitter` (`component: portainer`), and returns `status: success` or `status: error`.

Every denial at any gate also fires a `container_action_denied` event with the specific `reason` (`ambiguous_or_not_found_container`, `not_in_allow_list`, `identity_policy_blocked`, `cert_insufficient`, `hoh_ack_timeout`, `hoh_ack_declined`).

### Identity Model

There is currently no mechanism binding an MCP session to a specific persona cabinet. `caller_id` is a free-text field the caller can set to whatever they like — it is recorded in audit events for traceability, but every gated action is checked against **the default agent's own `infra_container_control` certification**, regardless of what `caller_id` says. Practically: as of this build, all x/d actions run "as the default agent, or not at all." Per-`caller_id` ACL was considered and explicitly rejected as a false security boundary.

### Fail Policy

Fail-closed throughout: any ambiguity, missing config, insufficient certification, dispatch failure, or a failed/timed-out household-admin acknowledgment denies the action. There is no best-effort or partial-success path in the write gates.

---

## Z-Wave Diagnostics Codex

A second, independent add-on on the same console script. Read-only and always active — `zwave_js` is either installed or it isn't, so unlike the container-control codex there's no `not_configured` gate on the read modes.

`zwave_js` diagnostic sensors (RTT, dropped/successful commands, node status, last-seen) ship **disabled by default** per device, so cross-node correlation is worthless until they're turned on. "Enabled" is inferred from state-object existence (a disabled entity has none at all; an enabled-but-unavailable entity still has a state object) rather than an entity_registry read, since there's no clean write path to the registry anyway.

### Modes

| Mode | Class | Gating |
|---|---|---|
| `zwave_diag_audit` | read | None — every `zwave_js` device, which diagnostic sensors it has, and whether each is enabled |
| `zwave_ping` | read | None — fires a device's `button.*_ping` (resolved via `container=` name substring). Non-destructive despite the name |
| `zwave_capabilities_discover` | read | None — generic per-device select/number config parameter enumeration plus other entities. `container=` name substring |
| `zwave_mapping_audit` | read | None — flags devices missing an area assignment or with zero registered entities. Not a full registry-consistency sweep |
| `zwave_diag_enable` | x (write) | `infra_zwave_control` cert required. Enables a device's disabled diagnostic entities via `homeassistant.enable_entity`. `container=` name substring, required |
| `zwave_health_correlate` | read (+ optional ticket sync) | None for the scan itself. Live anomaly scan across dropped/timed-out commands, high RTT (>1000ms), and abnormal `node_status`, grouped by area to surface a probable shared cause (2+ anomalous devices in one area) rather than coincidence |

### `zwave_health_correlate` Scoping

`container=` (a name or area substring) scopes the **whole report** returned to the caller — not just ticket-sync filing. A scoped call only sees anomalies matching that device-name or area-name substring.

Auto-resolve (closing tickets for devices no longer anomalous) always evaluates against the full, unscoped anomaly list regardless of the `container=` scope — a scoped call must never wrongly auto-close an out-of-scope device's ticket just because that particular call didn't look at it.

Passing `caller_id=sync_tickets` additionally files/updates/closes Radar/Zammad tickets (tag `zwave_issue_<device slug>` plus a shared `zwave_issue` tag) for anomalous devices — one open ticket per device, a note added only if its flags changed since the last note, never re-filed. Inert (`status: not_available`) if Radar/Zammad isn't configured on the install; scan-only calls (no `caller_id=sync_tickets`) work regardless.

Devices with no diagnostic sensors enabled are invisible to `zwave_health_correlate` — run `zwave_diag_audit` first, then `zwave_diag_enable` (requires `infra_zwave_control`) to close any gaps.

### Cert: `infra_zwave_control`

Gates `zwave_diag_enable` only (x-class, cert only — no live-ack tier). Grant via `persona_editor mode=cert_grant cert_component=infra_zwave_control` (or `zen_dojotools_identity mode=cert_list` to see current status).

---

## Admin Configuration — `zen_admintools_portainer_acl`

**Not MCP-exposed.** Run via HA Developer Tools → Actions only. This is the sole writer of `integrations_config.portainer` (connection URL, `controllable_containers` allow-list, `min_cert_level.x`/`.d`); the container-control codex only ever reads it.

| Mode | Description |
|---|---|
| `help` | Field/mode reference |
| `config_get` | Read current url, allow-list, and min cert levels |
| `config_set` | Set/replace url, `controllable_containers` (a real list, e.g. `["wud"]`), and/or `min_cert_x`/`min_cert_d`. Requires `confirm: true` |
| `tool_manifest` | Standard manifest response |

`config_set` refuses without `confirm: true` — same defense-in-depth pattern used elsewhere in this repo for admin-plane writes (e.g. `stacks_bulk_edit` delete/split, `reload_all`).

---

## Internal Broker — `zen_root_portainer`

**Not MCP-exposed**, internal to the codex only (`plugins/portainer/portainer.yaml`). Normalizes GET/POST against the Portainer REST API (`X-API-Key` auth, from `secrets.yaml: portainer_token`), resolves the configured base URL from the household cabinet, and — importantly — never returns raw container inspect data (env vars can carry secrets); only shaped fields (`id`, `name`, `state`, `image`, `endpoint_id`) reach the codex.

---

## Setup

1. No setup required for the read/status modes (`status`, `nodes`, `containers`, `monitors`, `updates`, `certs`, `ha`, `log`, `discover`) — self-bootstrapping via label/integration lookup, same as before v1.3.0.
2. For the container-control codex:
   - Set `secrets.yaml: portainer_token` to a Portainer API key.
   - Run `zen_admintools_portainer_acl mode=config_set url="https://portainer.<your-domain>:9443/" controllable_containers='["wud"]' confirm=true` (or pass `portainer_url` on a codex call to self-provision the URL only — the allow-list still needs an explicit admin grant).
   - Read-only codex modes (`containers_list`, `container_get`) work once the URL is set; write modes additionally need the target container on the allow-list and the default agent's `infra_container_control` certification at the required level.

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `sensor.zen_default_household_cabinet_resolved` | `integrations_config.portainer` storage |
| `script.zen_dojotools_filecabinet` | Drawer reads (config) |
| `script.zen_root_portainer` | Portainer REST calls (codex only) |
| `script.zen_dojotools_identity` (`mode=resolve_caller_identity`, `mode=request_live_ack`) | Cert check for x/d actions; household-admin acknowledgment round-trip for d-class actions (delegated, not hand-rolled here) |
| `script.zen_dojotools_event_emitter` | Audit trail for codex actions/denials |
| `script.zen_dojotools_ha_log_viewer` | Backs `mode=log` |
| Proxmox / Portainer / Uptime Kuma / WUD / IPP integrations (as labeled) | Data sources for the read modes |

---

## Version History

| Version | Change |
|---|---|
| 1.4.2 (2026-08-15) | d-class household-admin acknowledgment delegated to the shared `zen_dojotools_identity mode=request_live_ack` chokepoint, replacing a hand-rolled fire+poll block. Also fixed the underlying `notify_target` default — the acknowledgment path had been silently failing closed since it was first built (`mobile` was never a real dispatch target). |
| 1.3.0 | Added `mode=log` (tail + restart/reload/error search peek) |
| 1.2.0 | Added `mode=ha` (HA supervisor/addon/disk health); HA platform updates folded into `mode=updates` |
| 1.1.0 | Label-driven rewrite; `integration_entities()` for Portainer isolation; fixed memory unit (MiB→GiB); `mode=discover` added |
| 1.0.0 | Initial release |

> The container-control codex (`containers_list`/`container_get`/`container_restart`/`container_start`/`container_stop`/`container_remove`) is present in the current file but postdates the changelog entries above — treat the codex sections in this doc as the source of truth for that surface until the changelog catches up.
