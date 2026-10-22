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

## Read this before upgrading

- **The ACL v2 migration is mandatory.** Identity setup refuses an old-schema household or family with `acl_schema_upgrade_required`. After reloading (templates, automations, template entities, then scripts), run `zen_dojotools_identity mode=acl_migrate` to see the plan (a dry run is the default), then apply it with `dry_run: false` and `confirm_action: true`. A second run should report 0 changes. Flynn's bootstrap runs the same migration on its own: on a fresh install, and on an existing one the next time the bootstrap runs. Drawers need no migration of their own; existing ones read as before.
- **FileCabinet reads return content in `value`.** The `{path: value}` map that `read` put in `result` is retired; `result` is only the tool envelope, and `read` is a plain alias of `get`. Anything of your own that reads `result.<path>` needs to read `value`.
- **FileCabinet writes are honest.** A create, update, upsert or delete the cabinet never confirmed answers `status: error`, `code: write_unverified` instead of success. `update` on a missing key answers `not_found` (`force_action` creates it), and `create` on an existing key is an error unless `force_action`.
- **Every write is a merge or a replace.** `upsert` deep-merges by default and `update` replaces. A read-modify-write caller passes `mode: replace`; under a merge, a key you removed comes back.
- **Structural cabinet keys belong to the admin door.** VolumeInfo, `meta`, `_label_index`, `_zen_relationships`, the system cabinet and the vault are written only through `zen_admintools_cabinet_writer`, and FileCabinet refuses them. Raw `set_variable_*` events of your own aimed at those keys need to move to the door.
- **The lens and codex registries have one writer each:** `prompt_loader mode=lens_registry` and `mode=codex_registry`. Code writing those drawers directly needs to go through them.
- **`run_repair` has one action, `acl_migrate`.** The 4.5.6 repair names are refused, and `maint_4_5_6.yaml` and `repair_fam_cab_volumeinfo.yaml` are gone. The cabinet factory no longer takes `family_guid`.
- **`reset_all` wipes.** Cabinet admin's `reset_all` did nothing while answering success; it now wipes every cabinet. A wipe clears the console override and every cert along with everything else, so throw the override from the console and re-request certs afterwards.

---

## Added

- **Semantic roots.** The tool manifest publishes a root for each question word: who (Identity), what (Index), when (Taskmaster almanac), where (Room Manager), why (history), how (help) and knowledge (FileCabinet). Each root answers what matters now and names where to go deeper, and the boot prompt renders them as one line.
- **Taskmaster almanac, the "when" root.** One call returns a headline and the current period of the day, vacation, events from calendars carrying the `almanac` label (timed events first, at most two per calendar), tasks due by today, meals and weather. `horizon_hours`, `lookahead_days` and `max_items` tune it.
- **Per-tool roots on the core tools.** Room Manager, Index, FileCabinet, Identity and Taskmaster declare their own roots in their `tool_manifest` modes. A bare call returns them in about 350 characters.
- **Cortex 45.** Cortex 44 'Open Door' with one fix: its `core_tools` list named a log viewer that does not exist, and now names `zen_dojotools_ha_log_viewer`. 45 is the new latest and replaces the v42 slot, so the selectable versions are 45, 44 and 43.
- **The admin door.** `zen_admintools_cabinet_writer` is the one writer for structural, system and vault cabinet writes that FileCabinet refuses by design. It is not MCP-exposed, every write names its caller and is logged, and it waits until the cabinet shows each write before it answers. Identity's VolumeInfo writes now go through it.
- **Who knows where a member belongs.** `identity resolve` returns `name`, `person_entity`, `member_type` and `memberships`, and finds a person anywhere in the household tree, not only among the household's own users.
- **One ACL record.** Identity and FileCabinet share VolumeInfo `acls` v2: `{v: 2, owner, prime, partners, ops}`, with cabinet GUIDs as principals and `ops` holding a `cert`, `ack` and `who` rule for each of read, write, delete and admin. A drawer's own envelope `acl` may set `ops` only, and each op inherits on its own from drawer, then cabinet, then the default, which is open. Identity writes the record; FileCabinet and Identity read it only through the CABS `acl_*` macros and `ID.principals`.
- **Audit-only access gates.** FileCabinet resolves the real target behind any mount chain, then checks that target's ACL. It logs would-deny and would-ack decisions (`fc_acl_audit`) and returns `acl_audit`, but refuses nothing yet. The admin door does the same for agent and console writes against `zenos.system.cabinet_admin` (`gate_audit`); boot, Flynn and automation chains are exempt.
- **Identity ACL modes.** `acl_migrate` brings every cabinet to v2 and repairs bad identity data (missing ids, v1 lists, stray VolumeInfo copies, mismatched member GUIDs), and never drops a dangling principal. `acl_set` lets a cabinet's owner change its ops. `take_ownership` needs `zenos.identity.acl_take_ownership` and a live acknowledgement on every call, rewrites only the ACL, and is audited.
- **Admin door `merge` mode.** Lays the given keys over the stored value at write time, queued, so concurrent merges don't lose each other's keys. Every Identity ACL write uses it.
- **FileCabinet GC `repair` mode.** Normalizes one drawer to the `{value, meta}` envelope, with `dry_run`.

## Changed

- **Help is the default.** A `zen_*` script answers help for a missing or unknown mode, and a bare call (no mode and no input) never acts, crashes or errors. Tools that used to act on a bare call (announce, the summarizers, the prompt loader, the reset and repair tools, Camera, Provisioner and FileCabinet GC) now return help. Calls that pass input are unchanged, and every internal caller now names its mode.
- **Compact bare replies.** Inventory, Kitchen and Finance return a short mode list on a bare call, down from roughly 100k, 65k and a few thousand characters. The full catalogue is for `mode=help`.
- **Manifest publish cadence.** Publishing runs on the events a reload itself fires, writes only when content changed, and cannot loop on its own events.
- **The KungFu Loader is retired.** Every component self-registers through KF5 (each tool's own `mode=kfc_manifest`, registered by `bootstrap_kfc` on reload and daily), so a prompt load now only loads the prompt and the `ship_zen_system` option is gone. The manifest whitelists `<tool>:kfc_manifest`.
- **FileCabinet has one exit per action.** Get, create, update, upsert, delete and list each set a result and fall through to the engine's single exit. Unconfirmed writes answer `write_unverified` and log a warning (logger `zen_sutra_filecabinet`), so a caller that never reads the answer still leaves a trace. Delete answers `children_deleted`. In-repo callers call `zen_sutra_filecabinet` directly, one script run instead of two; the wrapper still works.
- **The drawer envelope is `{value, meta, timestamp[, acl]}`.** A cabinet set keeps `meta` and `acl` instead of rebuilding the drawer, and `meta` is always present. Expiry (`expires_after`, `no_autoexpire`, `no_autorecycle`) is stored in `meta` and reported by `get`. Expiry still stored inside an older value counts, and GC normalizes every drawer it moves.
- **Admin writes go through the door.** AdminTools, Flynn, FileCabinet GC and maint no longer fire raw cabinet events.
- **Mount chains resolve whole.** Get, create, update and upsert follow a mount chain up to 8 hops to the real target, where writes used to stop at the first hop. A loop, a chain too deep or a missing target refuses with `mount_traversal`.
- **The console override stands in for every live acknowledgement.** While `cert_admin_console_bypass` is thrown, Identity's `request_live_ack` approves at once with `reason: console_bypass`, and the approval is audited. Hammer, reset, reset_all and take_ownership honor it.
- **`prompt_loader` runs queued,** up to 25 at once, so a burst of registry calls waits its turn instead of being dropped. When `sim_mode_allowed` is unset, it stamps it from Flynn's `input_boolean.zenos_sim_mode_override` (on by default) instead of `false`, so a load right after `reset_all` still resolves its caller.
- **Identity `area_labels` holds only real HA areas,** so data-routing labels no longer show up in a "who" answer.

## Fixed

- **Event emitter refuses internal event kinds.** The agent-facing `zen_dojotools_event_emitter` passed any `kind` through to the event bus, and several listeners act on kinds: scheduled reloads, tool dispatch, approval acknowledgements, prompt injection, garbage collection, and cabinet, identity, alert and label signals. Those kinds are now reserved and refused with a `guarded:` error. Internal callers fire their own events directly. Free-form kinds and the summary and ninja forces still work.
- **The help proxy only reaches ZenOS tools.** `zen_dojotools_help` took any script name and called it with `mode=help`, which runs the default path of a script that has no help handler. It now proxies only existing `zen_dojotools_`, `zen_stack_`, `zen_sutra_` and `zen_codex_` tools and refuses anything else with a `guarded:` error.
- **Cabinet schema upgrade now runs.** SystemTools sent the cabinet admin tool an operation field it does not read, so the upgrade never started.
- **ZenZork `examine` returns lens results** instead of help.
- **Event emitter no longer fails when `severity` is omitted.**
- **Trapper Keeper has real input.** It was seeded with nothing and reported a false telemetry blackout. The Ninja summarizer also no longer calls the model when there is nothing to read, and it unwraps the index result envelope so the inspect supplement receives data.
- **A provider whose `kfc_manifest` answers `result: ok` with the definition elsewhere** handed FileCabinet's live-mount read the string "ok". `result` must carry the definition; AutoVac and Taskmaster now do.
- **Home Mode anchors must be strictly ascending.** Six equal anchors, which is what a fresh install reports, used to pass the check and hold the house in Night-Late all day. They now fall back to the default schedule and raise `home_mode_anchor_misconfigured`.
- **Security Manager refreshes after a restart.** Its reading now also runs on Home Assistant start and on home mode updates, with a 60 second settle, so a reading taken while the panel was still catching up no longer sits at the top of the summary until midnight.
- **Cabinet admin refusals hold.** A cert denial, a missing `hammer_ok`, or a refused live acknowledgement set an error and then fell through to the operation, so the operation ran anyway: a hammer without `hammer_ok` wiped the cabinet. Everything after the guards now runs only when no refusal was set, and `reset` asks for the same per-call live household-admin acknowledgement as hammer.
- **Identity remove modes find the member's real list.** A family removed without `member_type` stayed put while the call reported success.
- **Import recursion in `zen_identity.jinja`.** The header's usage example ran a real import of the file, so any import recursed. The header is now a Jinja comment.
- **Flynn log noise.** `flynn_unified_engine` defaults `value` and `action_type` in the template instead of logging `'value' is undefined` on every stepgate pass.
- **The camera heartbeat runs again.** Its old drawer matched no label, so the Ninja empty-input guard skipped it. It is now a KF5 component of the camera tool, enabled only while cameras carry the `exterior` or `interior` label.
- **FileCabinet's `write` alias meets the write guards.** A direct engine call with `action: write` skipped the system cabinet, read-only, protected drawer, reserved key and health guards. Aliases now resolve before any guard.
- **Cabinet guards block.** A write past the 128 KB size cap, or to a cabinet whose contents are unknown at boot, is refused and the cabinet stays as it was. FileCabinet answers `write_unverified`.
- **`move` onto its own key refuses** with `code: same_key` instead of writing the drawer onto itself and deleting it.
- **Identity VolumeInfo writes touch only the changed keys,** so an empty read can no longer replace the whole drawer.
- **Flynn's history mount** reads the mount pointer from the drawer's value and stops re-sending `set_mount` on every bootstrap; a failed mount logs a warning. Flynn's primary-user picker now hides the head of household.
- **Smaller fixes.** A re-fired alert's answer replaces the cached one. Provisioner's deprovision reports an error when the cabinet doesn't end up unmounted. `flip_schema_version` flips back. Stack and lens unregisters remove their entry. Grocy, Lights, Media Manager, Room Manager and cabinet inspect read drawer content from `value`.

## Removed

- **`maint_4_5_6.yaml` and `repair_fam_cab_volumeinfo.yaml`.** `acl_migrate` covers both, for every cabinet, and Flynn's bootstrap runs it.
- **The 4.5.6 `run_repair` actions** and the cabinet factory's `family_guid` field.
- **The CABS `cabinet_acls` and `drawer_acl` macros.** Use `acl_record` and `acl_for`.
