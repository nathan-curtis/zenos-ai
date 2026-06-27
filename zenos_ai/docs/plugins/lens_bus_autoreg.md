# Lens Bus — Auto-Registration

**Introduced:** ZenOS-AI 2026.7.0 'Neo'  
**Status:** GA  
**Scope:** `zen_dojotools_manifest` + any `zen_stack_*` / `zen_sutra_*` lens provider

---

## What This Does

Before auto-registration, adding a new Lens Bus provider required two manual steps: writing the provider declaration into the `lens_registry` cabinet key, and adding a dispatcher arm if you wanted MCP routing. Both steps were easy to forget and not idempotent.

Auto-registration eliminates both steps for standard stack providers. Any `zen_stack_*` or `zen_sutra_*` script that declares the right fields in `tool_manifest` is automatically discovered and registered on every HA boot and daily at midnight — no manual cabinet writes, no dispatcher changes required.

---

## How It Works

### The Bootstrap Run

`zen_dojotools_manifest mode=bootstrap_stacks`:

1. Scans the entity registry for all `zen_stack_*` and `zen_sutra_*` scripts
2. Calls `mode=tool_manifest` on each one
3. For each script that returns a `register_mode` field:
   - Checks if `provider_key` is already in `lens_registry` → skips (idempotent)
   - Checks if `provider_key` is in the `lens_bootstrap_skip` cabinet key → skips if listed
   - Otherwise calls `mode={{ register_mode }}` on the script to self-register
4. Returns: `registered`, `already_registered`, `skipped`, `no_register_mode`, `errors`

### When It Runs

The `zen_manifest_bootstrap_stacks` automation fires `bootstrap_stacks` on:
- `homeassistant_started` — every HA boot and reload
- Daily at `00:00:30`

No manual trigger needed under normal operation. Run `zen_dojotools_manifest mode=bootstrap_stacks` manually if you want to register a new provider immediately without restarting HA.

---

## Provider Contract

For a `zen_stack_*` or `zen_sutra_*` script to participate, its `tool_manifest` response must include:

| Field | Required value | Purpose |
|---|---|---|
| `tier` | `stacks` | Gates scan participation — only stacks tier scripts are processed |
| `provider_key` | e.g. `firefly_iii` | Key written into `lens_registry`; must be unique across all providers |
| `register_mode` | e.g. `register` | The mode bootstrap calls to complete registration |

The script must also implement:

| Mode | What it does |
|---|---|
| `register` | Writes provider declaration into `lens_registry` household cabinet key |
| `unregister` | Removes provider declaration from `lens_registry` |
| `stacks_by_anchor` | Consumer-facing lookup — receives `anchor_type`, `anchor_ids` |
| `health`, `inspect`, `audit`, `help` | Standard Lens Contract modes |

---

## Registration vs. Reachability

A provider registers as `status=available` even when its backend is not yet configured (URL not set, token not in secrets.yaml). Consumers get empty evidence (soft fail) on actual calls until the integration is live.

This separation is intentional: **registration confirms capability, not reachability.** A provider declares "I exist and I understand this domain" independently of whether it can currently reach its backend. This allows the lens registry to reflect the full intended topology without requiring every integration to be fully stood up before it can participate.

---

## Providers (as of 2026.7.0)

| Provider key | Script | First registered by | Consumes | Returns |
|---|---|---|---|---|
| `firefly_iii` | `zen_stack_firefly` | **Bootstrap (introduced this release)** | label, person | transaction_evidence |
| `wiki_js` | `zen_sutra_wikijs` | Previously manual; now bootstrap | label, concept, area_id, zone, person | article_evidence |
| `paperless_ngx` | `zen_stack_paperless` | Previously manual; now bootstrap | label, person, area_id, zone | document_evidence |

`zen_stack_firefly` introduced the pattern. WikiJS and Paperless-NGX adopted it in the same release.

---

## The Escape Hatch

`lens_bootstrap_skip` — household cabinet key, value is a JSON list of `provider_key` strings.

```json
["paperless_ngx"]
```

Any key in this list is skipped on every bootstrap run. Default: absent — register everything that participates. Normal installs never need this key.

---

## Adding a New Provider

1. Build a `zen_stack_<name>` or `zen_sutra_<name>` script with these modes:
   - `tool_manifest` — return `tier: stacks`, `provider_key: <name>`, `register_mode: register`
   - `register` — write provider entry to `lens_registry` via FileCabinet
   - `unregister` — remove provider entry from `lens_registry`
   - `stacks_by_anchor` — consumer-facing lookup
   - `health`, `inspect`, `audit`, `help` — standard contract

2. On next boot (or manual `zen_dojotools_manifest mode=bootstrap_stacks`), the provider is live.

3. Nothing else to change:
   - No dispatcher arm needed (the registry is the routing mechanism)
   - No consumer tool changes (Inspect, Room Manager, Service Desk all query the registry)
   - No manual cabinet write to `lens_registry`

---

## Dispatcher Note

The Lens dispatcher (`zen_dojotools_lens_dispatch`) does **not** need an arm for stack providers — the registry is the routing mechanism. Dispatcher arms are only required for tools that need direct MCP or event invocation outside of the lens dispatch flow.

> In 2026.7.0, the `zen_dojotools_manifest` dispatcher arm was also fixed: the `mode` field was missing, which meant only `mode=cabinets` could be routed through dispatch. All modes including `bootstrap_stacks` are now routable.

---

## Related Docs

- [DojoTools Manifest](../scripts/zen_dojotools_manifest_readme.md) — `bootstrap_stacks` mode reference
- [Firefly III Plugin](firefly_iii.md) — first provider to use auto-registration
- [Paperless-NGX Plugin](paperless_ngx.md)
- [WikiJS Plugin](wiki_js.md)
