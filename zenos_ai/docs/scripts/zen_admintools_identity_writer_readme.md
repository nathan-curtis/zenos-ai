# Zen AdminTools Identity Writer — v1.2.0

**File:** `packages/zenos_ai/dojotools/dojotools_identity_writer.yaml`
**Script:** `zen_admintools_identity_writer`
**Tier:** admintools. Not MCP-exposed.

## Overview

Flynn's own door for the identity wiring the Sentinel does at boot, before any user or agent exists. Agents use `zen_dojotools_identity`, which is gated like every agent-facing tool. The Sentinel's boot calls go here instead, so gating Identity can't stop a boot.

It has no cert gate because its only caller is the Flynn Sentinel (`flynn_bootstrap_content`). It's on the admin plane: Flynn's admin plane guard removes every `script.zen_admintools_*` from Assist, Alexa and Google, and no agent-facing tool may call it or carry a bypass for it.

## Modes

| Mode | What it does |
|---|---|
| `build_identity_manifest` | Builds the identity manifest |
| `household_add_family`, `household_add_member`, `family_add_member` | Household and family membership |
| `link_partners` | Links two cabinets as partners |
| `acl_migrate` | The ACL schema migration |
| `cert_cache_refresh` | Rebuilds the cert catalog cached in the vault (`force_refresh` to rebuild live) |
| `persona_write` | Writes the agent's name and primary user into its essence (`persona_name`, `primary_user`, optional `target`, `force`, `autosign`, `signed_by`) |
| `help`, `tool_manifest` | Self-description; never a write |

Every mode sets one response and leaves through a single exit.

## Two copies, kept in step

These are Flynn's own copies of operations `zen_dojotools_identity` also has. Identity keeps its implementation, unchanged and gated for agents, and never redirects here; only the Flynn Sentinel is pointed at this door. When either copy changes, the other has to be changed to match by hand.

## Version History

| Version | Change |
|---------|--------|
| v1.2.0 | New in 2026.11.1. The Sentinel's household, family, partner, identity manifest, ACL migration, cert cache refresh and persona name write calls. |
