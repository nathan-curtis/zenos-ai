# UAT APPROVED — feat/ready-player-2

**Date:** 2026-03-23
**Signed off by:** household (via Claude flash test pass)
**Branch:** feat/ready-player-2 @ d3e0783

---

## Test Summary

### Provisioner (expansion_cabinet_1)
- `online_unmounted` → `provision` → `online_mounted` ✓
- GUID stamped, profile loaded, `zen_user_cabinet` label applied ✓
- `deprovision` → `online_unmounted`, verified ✓
- Occupied-cab guard: blocks slam on `online_mounted` ✓
- Auto-stamp: `init` → auto-stamp → `online_unmounted` → provision ✓

### Reload persistence
- Provision → `ha_reload_all` → state survives as `online_mounted` ✓
- No `restore: true` required — HA preserves in-memory state across reloads ✓

### Cabinet state machine
- One-write-behind fix: state evaluates post-write `_eff`, not pre-write `current` ✓
- `homeassistant.start` trigger: `event: start` (not `started`) ✓
- All 20 cabinet entity IDs align with friendly names ✓

### Index + history
- Label resolution, expand_entities, `+history` (24h buckets) all green ✓
- Stats-eligibility gate correct ✓

### Agent health
- Monastery degraded state surfaces as `ready_monastery_<state>` not `ready` ✓

---

## Bugs filed this session (all resolved in d3e0783 or local)

| Bug | Status |
|-----|--------|
| Cabinet state one-write-behind | Fixed, shipped |
| Expansion slot entity_id naming mismatch | Fixed, shipped |
| `event: started` invalid (reverted on merge) | Fixed in d3e0783 |
| `restore: true` invalid for trigger-based sensors | Fixed in d3e0783 |
| Agent health monastery note logic | Fixed, shipped |
| Scheduler kata tojson double-encode (FG-38 V3) | Fixed in d3e0783 |

---

## Open (not blocking merge)

- Reload-state-loss for brand-new entity IDs (no prior state history) — by-design limitation, document only
- Migration script for installs with data in old expansion entity IDs (secondary/tertiary/quaternary/secondary_ai)

---

**Merge when ready.**
