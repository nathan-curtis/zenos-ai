# Release Notes — 2026.11.0 'This Is Spinal Tap'

**Status:** In development (beta)
**Branch:** `feat/2026.11.0`
**Base:** 2026.10.1 (patch on 2026.10.0 'Tron')

---

## Summary

Security goes to eleven.

Tron made admission explicit and put a badge in every agent's hands. This release makes the badge matter, and puts locks on the cabinets behind it. FileCabinet holds identity state, component definitions, configuration, mounts and the material that becomes agent context, and its live drawers can invoke tools. Permission to change a drawer can change what the system later believes or does, so the drawer is the next boundary to enforce.

The rule is the one the guitar's owner laid down: don't touch it, don't look at it, don't even point at it. A cabinet your agent has no pass for doesn't just refuse the read. It never shows up in the listing.

**Backstage passes for agents, locks on the cabinets, and a working screen for the show.**

---

## Planned Scope

**The goal:** no unadmitted agent reaches a protected drawer, while System bootstrap and recovery keep working.

**The chain:** Postman approves, CertAdmin admits, the enclave stores credentials, `zen_root_authentik` authenticates, Identity authorizes, and FileCabinet enforces. Where a call came from is recorded for audit and is never authority.

1. **Access-path audit.** Every way into a protected drawer is mapped before any enforcement lands: tool calls, mounts, live drawers, boot and repair paths.
2. **FileCabinet security.** One enforcement point. Read, write, protected configuration and cabinet administration are separate permissions. They cover listing, search, copies, moves, mounts, expansion and live-drawer dispatch. Authorization follows the resolved destination, and a cross-cabinet operation checks both sides. Following a mount confers no authority, so mount resolution happens before the access check. Callers see only what they may read: a hidden drawer is indistinguishable from one that does not exist. You can't touch the guitar, and you can't point at it either. Effective inherited access can be shown per drawer.
3. **Trusted entry points.** System and Default Agent authority are separated. Flynn and the prompt loader are named trusted entry points, and there is no agent-selectable "system" bypass.
4. **Admission enforcement.** `zenos.agent.basic` stops being a badge that is requested and becomes one that is checked, through an admission-token contract with `zen_root_authentik`. A Postman approval is bound to the enrollment it was given for and cannot be replayed. Participation stays distinct from administrative authority.
5. **Bootstrap and recovery.** Flynn's cold boot, upgrade, revocation and recovery run on a trusted path, and each is proven.
6. **Audit-only rollout.** Enforcement ships first as would-deny logging, with a rollback, then turns on.
7. **Operator permission report.** Shows the resolved identity, granted capabilities, effective cabinet access and the reason an operation was denied. Secrets are excluded.

**FileCabinet interface cleanup.** The `action_type` field moves to `mode=` (with merge and replace behavior moving to `merge:`), and a compatibility shim carries existing callers. It follows the audit, because the audit maps the same callers.

**Also planned**
- Every tool guards its inputs (the contract sweep begun in 2026.10.1).
- One configure contract across plugin setup.
- Identity and Labels event-ordering hardening.
- Display, finished: the shared dashboard ships, interruption awareness covers the remaining Media Manager watch paths, and expiry and dismissal are tested against the real scheduler cadence.

**Later phase:** binding to the conversation or session. It needs its own design and acceptance criteria before it joins scope.

Order of work: audit, designs, audit-only mode, enforcement.

---

## Scope of the Claim

FileCabinet enforcement protects the ZenOS access boundary. Home Assistant administrators, and any path with direct access to the underlying state, remain part of the trusted platform.

---

## Added

- **Semantic roots.** The tool manifest publishes a root for each question word: who (Identity), what (Index), when (Taskmaster almanac), where (Room Manager), why (history), how (help) and knowledge (FileCabinet). Each root answers what matters now and names where to go deeper, and the boot prompt renders them as one line.
- **Taskmaster almanac, the "when" root.** One call returns a headline and the current period of the day, vacation, events from calendars carrying the `almanac` label (timed events first, at most two per calendar), tasks due by today, meals and weather. `horizon_hours`, `lookahead_days` and `max_items` tune it.
- **Per-tool roots on the core tools.** Room Manager, Index, FileCabinet, Identity and Taskmaster declare their own roots in their `tool_manifest` modes. A bare call returns them in about 350 characters.

## Changed

- **Help is the default.** A `zen_*` script answers help for a missing or unknown mode, and a bare call (no mode and no input) never acts, crashes or errors. Tools that used to act on a bare call (announce, the summarizers, the prompt loader, the reset and repair tools, Camera, Provisioner and FileCabinet GC) now return help. Calls that pass input are unchanged, and every internal caller now names its mode.
- **Compact bare replies.** Inventory, Kitchen and Finance return a short mode list on a bare call, down from roughly 100k, 65k and a few thousand characters. The full catalogue is for `mode=help`.
- **Manifest publish cadence.** Publishing runs on the events a reload itself fires, writes only when content changed, and cannot loop on its own events.

## Fixed

- **Event emitter refuses internal event kinds.** The agent-facing `zen_dojotools_event_emitter` passed any `kind` through to the event bus, and several listeners act on kinds: scheduled reloads, tool dispatch, approval acknowledgements, prompt injection, garbage collection, and cabinet, identity, alert and label signals. Those kinds are now reserved and refused with a `guarded:` error. Internal callers fire their own events directly. Free-form kinds and the summary and ninja forces still work.
- **The help proxy only reaches ZenOS tools.** `zen_dojotools_help` took any script name and called it with `mode=help`, which runs the default path of a script that has no help handler. It now proxies only existing `zen_dojotools_`, `zen_stack_`, `zen_sutra_` and `zen_codex_` tools and refuses anything else with a `guarded:` error.
- **Cabinet schema upgrade now runs.** SystemTools sent the cabinet admin tool an operation field it does not read, so the upgrade never started.
- **ZenZork `examine` returns lens results** instead of help.
- **Event emitter no longer fails when `severity` is omitted.**
- **Trapper Keeper has real input.** It was seeded with nothing and reported a false telemetry blackout. The Ninja summarizer also no longer calls the model when there is nothing to read, and it unwraps the index result envelope so the inspect supplement receives data.
- **A provider whose `kfc_manifest` answers `result: ok` with the definition elsewhere** handed FileCabinet's live-mount read the string "ok". `result` must carry the definition; AutoVac now does.
- **Home Mode anchors must be strictly ascending.** Six equal anchors, which is what a fresh install reports, used to pass the check and hold the house in Night-Late all day. They now fall back to the default schedule and raise `home_mode_anchor_misconfigured`.
