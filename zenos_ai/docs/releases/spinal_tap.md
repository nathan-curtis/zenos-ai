# Release Notes — 2026.11.0 'This Is Spinal Tap'

**Status:** In development (beta)
**Branch:** `feat/2026.11.0`
**Base:** 2026.10.1 (patch on 2026.10.0 'Tron')

---

## Summary

Security goes to eleven.

Tron made admission explicit and put a badge in every agent's hands. This release makes the badge matter, and puts locks on the cabinets behind it. FileCabinet holds identity state, component definitions, configuration, mounts and the material that becomes agent context, and its live drawers can invoke tools. Permission to change a drawer can change what the system later believes or does, so the drawer is the next boundary to enforce.

**Backstage passes for agents, locks on the cabinets, and a working screen for the show.**

---

## Planned Scope

1. **FileCabinet security.** Read, write, protected configuration and cabinet administration are separate permissions. They cover listing, search, copies, moves, mounts, expansion and live-drawer dispatch. Authorization follows the resolved destination, and a cross-cabinet operation checks both sides. Following a mount confers no authority, and security is resolved before a live drawer's target tool runs. Callers see only what they may read: a hidden drawer is indistinguishable from one that does not exist.
2. **Admission enforcement.** `zenos.agent.basic` stops being a badge that is requested and becomes one that is checked. Participation stays distinct from administrative authority.
3. **Trusted system operation and recovery.** Bootstrap, prompt loading, summarization, GC and repairs get explicit authority paths. Cold boot, upgrade, revocation and recovery are proven without an agent-selectable "system" bypass.
4. **Display, finished.** The shared dashboard ships, interruption awareness covers the remaining Media Manager watch paths, and expiry and dismissal are tested against the real scheduler cadence.
5. **Operator permission report.** Shows the resolved identity, granted capabilities, effective cabinet access and the reason an operation was denied. Secrets are excluded.

Stretch: a live inference health probe. Conversation-bound session tokens need their own design and acceptance criteria before they join scope.

---

## Scope of the Claim

FileCabinet enforcement protects the ZenOS access boundary. Home Assistant administrators, and any path with direct access to the underlying state, remain part of the trusted platform.

---

## Added

_Nothing yet._

## Changed

_Nothing yet._

## Fixed

_Nothing yet._
