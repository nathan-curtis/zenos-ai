# User and AI User Management — ZenOS-AI 2026.6.0 'Clue'

*Provisioning, deprovisioning, moving, and repairing identity cabinets for people and AI constructs*

---

## Concepts

ZenOS-AI manages human and AI identities through **cabinets** — HA template sensor entities that hold identity and memory data as drawers.

Every identity is backed by a cabinet. The system knows which cabinet plays which role via **labels**. Changing which cabinet is "the user cabinet" is a label operation, not a data migration.

### Cabinet States

| State | Meaning |
|---|---|
| `online_unmounted` | Cabinet exists, not assigned to an identity slot — sits in stacks, ready to provision |
| `online_mounted` | Cabinet is active and assigned to an identity slot via a type label |

### Identity Slots

| Slot (`cab_type`) | Label Applied | Example Cabinet |
|---|---|---|
| `ai_user` | `zen_ai_user` | `sensor.zenos_default_ai_user_cabinet` |
| `user` | `zen_user` | `sensor.zenos_default_user_cabinet` |
| `family` | `zen_family` | `sensor.zenos_default_family_cabinet` |
| `household` | `zen_household` | `sensor.zenos_default_household_cabinet` |

### The Default Label

The `zen_default_*` label (e.g., `zen_default_user_cabinet`) marks which cabinet in a slot is the primary one. Resolver sensors read this label. **Deprovisioning blocks if the target has a default label** — transfer the default label first via the Labels tool.

---

## The Provisioner

All provision/deprovision/move operations go through:

```
script.zen_dojotools_provisioner
```

Three modes:

| Mode | What It Does |
|---|---|
| `provision` | Pulls an `online_unmounted` cabinet into service. Validates GUID uniqueness, applies type label, mounts, preloads profile data, fires identity manifest rebuild. Rolls back on health gate timeout. |
| `deprovision` | Dismounts cabinet, strips type label, returns it to stacks. Blocked if cabinet holds a `zen_default_*` label. |
| `replace` | Deprovisions `replace_cabinet`, then provisions `target_cabinet` with the same `cab_type`. Atomic swap — the default label transfers with it. |

---

## AI User Management

### Add an AI User

**Requirement:** An `online_unmounted` cabinet must exist in stacks. Check `sensor.zen_cabinet_health` → `slot_entities` to see what's available. If no stacks cabinet exists, add a new cabinet sensor to `zenos_cabinets.yaml` and restart HA.

```yaml
script.zen_dojotools_provisioner:
  mode: provision
  cab_type: ai_user
  target_cabinet: sensor.<your_stacks_cabinet>
```

After provisioning, write the AI persona profile:

```yaml
script.zen_dojotools_profile_editor:
  mode: write
  target_type: ai_user
  target: sensor.<your_newly_provisioned_cabinet>
  persona_name: Vera
  pronouns: she/her
  voice_tone: precise
  voice_style: formal
```

> **Note:** If this is a second AI construct, use `target_type: secondary_ai_user` and target the specific cabinet. The profile editor's `target` field overrides the default slot resolver.

---

### Remove an AI User

> ⚠️ If this cabinet has the `zen_default_ai_user_cabinet` label, transfer that label to another cabinet first (see [Transfer the Default Label](#transfer-the-default-label)).

```yaml
script.zen_dojotools_provisioner:
  mode: deprovision
  target_cabinet: sensor.<cabinet_to_remove>
```

The cabinet returns to `online_unmounted`. Cabinet data is **not wiped** — the drawers remain. If you want to fully reset the cabinet afterward:

```yaml
script.zen_admintools_cabinetadmin:
  mode: reset
  target_cabinet: sensor.<cabinet_to_remove>
```

---

### Move an AI User to a Different Cabinet

Use `replace` mode to swap the active cabinet without data loss:

```yaml
script.zen_dojotools_provisioner:
  mode: replace
  cab_type: ai_user
  target_cabinet: sensor.<new_cabinet>
  replace_cabinet: sensor.<current_cabinet>
```

`replace` deprovisions the current cabinet and provisions the new one with the same `cab_type`. The identity manifest rebuilds automatically.

> **Profile data:** Profile drawers stay in the old (deprovisioned) cabinet. After the swap, write profile data to the new cabinet via the profile editor, or use FileCabinet to clone individual drawers from old to new.

---

## Human User Management

### Add a Person

Same provisioner call, different `cab_type`. Optionally pass a `person_entity` to preload the person's HA-registered name and user_id automatically.

```yaml
script.zen_dojotools_provisioner:
  mode: provision
  cab_type: user
  target_cabinet: sensor.<stacks_cabinet>
  person_entity: person.alex  # optional — preloads name and user_id
```

Write the profile after provisioning:

```yaml
script.zen_dojotools_profile_editor:
  mode: write
  target_type: user
  target: sensor.<newly_provisioned_cabinet>
  first_name: Alex
  preferred_name: Alex
  role: head_of_household
  pronouns: he/him
```

---

### Remove a Person

> ⚠️ If this cabinet has `zen_default_user_cabinet`, transfer that label first.

```yaml
script.zen_dojotools_provisioner:
  mode: deprovision
  target_cabinet: sensor.<user_cabinet>
```

Optional hard reset of the cabinet after deprovision:

```yaml
script.zen_admintools_cabinetadmin:
  mode: reset
  target_cabinet: sensor.<user_cabinet>
```

---

### Move a Person to a Different Cabinet

```yaml
script.zen_dojotools_provisioner:
  mode: replace
  cab_type: user
  target_cabinet: sensor.<new_cabinet>
  replace_cabinet: sensor.<current_cabinet>
```

---

## Transfer the Default Label

If you're deprovisioning or replacing a cabinet that holds a `zen_default_*` label, you must move that label first — otherwise the provisioner blocks.

```yaml
# Remove the default label from the outgoing cabinet
script.zen_dojotools_labels:
  action_type: untag
  label: zen_default_user_cabinet  # or zen_default_ai_user_cabinet, etc.
  entity: sensor.<outgoing_cabinet>
  confirm: true

# Apply it to the incoming cabinet
script.zen_dojotools_labels:
  action_type: tag
  label: zen_default_user_cabinet
  entity: sensor.<incoming_cabinet>
  confirm: true
```

Fire `zen_resolver_refresh` after:

```
Developer Tools → Events → Event type: zen_resolver_refresh → Fire Event
```

---

## Household and Family Management

ZenOS-AI has a three-layer membership model:

```
System
  └─ Household (occupancy — "lives here")
       └─ Family (belonging — "part of this unit")
            └─ Sub-family (extended family — not on-premises)
```

```mermaid
flowchart TD
  System["ZenOS system"]
  Household["Household\noccupancy: lives here"]
  Family["Family\nbelonging: part of this unit"]
  SubFamily["Sub-family\nextended family"]
  User["User cabinet"]
  AI["AI user cabinet"]

  System --> Household
  Household --> Family
  Family --> SubFamily
  Family --> User
  Household --> AI
```

**Household membership = residence.** Being in the household's `members` list means you occupy that space.

**Family membership = belonging.** Being in a family means you're part of that unit. Inviting your AI into a family is a special act — it fires a `family_member_joined` event with `prompt_profile_update: true` so the AI can surface naming and profile suggestions.

**Depth rule:** Families nest arbitrarily deep in the data model, but security resolution only chases 2 levels. `is_member(A, B)` checks: is A directly in B? Or is A in C which is directly in B? Anything deeper is not resolved for security purposes.

**Multiple households:** A system can host multiple households (duplex, shared building). `zen_default_household` is the primary. Each household manages its own membership independently.

---

### Onboarding Sequence (Standard)

When provisioning a fresh system, the recommended sequence:

1. **Mint household** — provision a cabinet with `cab_type: household`
2. **Add family to household** — wire your default family cabinet in
3. **Mint user** — provision a cabinet with `cab_type: user`
4. **Add user to family** — first family auto-sets `default_family_guid`; user also lands in household as HoH (first user fills the owner slot)
5. **Mint AI user** — provision a cabinet with `cab_type: ai_user`
6. **Add AI to household** — first AI fills the `prime` partner slot
7. **Link user and AI** — bidirectional partner link
8. **Optionally invite AI into family** — explicit invite required; fires join event

```mermaid
flowchart LR
  Household["Mint household"]
  Family["Add family to household"]
  User["Mint user"]
  UserFamily["Add user to family"]
  AI["Mint AI user"]
  AIHouse["Add AI to household"]
  Partner["Link user and AI"]
  Manifest["Rebuild manifest"]

  Household --> Family --> User --> UserFamily --> AI --> AIHouse --> Partner --> Manifest
```

---

### Wire a Family into a Household

```yaml
script.zen_dojotools_identity:
  mode: household_add_family
  family_entity: sensor.<family_cabinet>
```

Adds the family to the household's `members.families` list at depth 1. Family members become household residents by graph traversal (depth-1 resolution).

---

### Add a Member to a Household

```yaml
script.zen_dojotools_identity:
  mode: household_add_member
  member_entity: sensor.<user_or_ai_cabinet>
  member_type: user   # or ai_user
```

**First occupant behavior:**
- First `user` added → also fills `VolumeInfo.acls.owner` (Head of Household slot)
- First `ai_user` added → also fills `VolumeInfo.acls.partner` with `role: prime`

**Occupied slot behavior:** If the HoH or prime slot is already filled, this mode is blocked. Use an admin operation (SP1) to transfer ownership.

The response includes a `slot_filled` field (`hoh`, `prime`, or empty) so the caller knows whether a privileged slot was assigned.

---

### Remove a Member from a Household

```yaml
script.zen_dojotools_identity:
  mode: household_remove_member
  member_entity: sensor.<cabinet>
  member_type: user   # or ai_user
```

Removes from `members.users` or `members.ai_users`. Does **not** clear the `acls.owner` or `acls.partner` slots — those require a dedicated admin transfer (SP1).

---

### Add an External Family Member (No HA Account)

For people who matter to the household but don't have a Home Assistant account and no pre-provisioned cabinet — use `provision_member`. One call handles the full lifecycle.

```yaml
script.zen_dojotools_identity:
  mode: provision_member
  member_name: Marianne
  profile_payload: '{"first_name": "Marianne", "role": "extended_family", "pronouns": "she/her"}'
  # family_entity: sensor.<family_cabinet>   # optional — defaults to zen_default_family_cabinet_resolved
  # member_type: user                        # optional — default: user
```

`provision_member` will:
1. Find the first available expansion slot (`sensor.zenos_expansion_cabinet_1` through `_5`) in `init` state
2. Provision it as a user cabinet, seeding the name and any profile fields
3. Wire it into the target family via `family_add_member`
4. Rebuild the identity manifest

Hard-stops with a clear error if all 5 expansion slots are occupied — deprovision an existing member cabinet first.

> **OOBE:** The first-run setup (`flynn_oobe.yaml` step `3_people`) routes external family members through `provision_member` automatically. You do not need to call it manually during initial setup.

---

### Add a Member to a Family

```yaml
script.zen_dojotools_identity:
  mode: family_add_member
  family_entity: sensor.<family_cabinet>
  member_entity: sensor.<user_ai_or_sub_family_cabinet>
  member_type: user   # user | ai_user | family
```

**Default family:** If this is the member's first family (no `default_family_guid` set), the family is automatically marked as their default.

**Sub-families:** `member_type: family` wires in a sub-family at depth 2. Sub-family members are extended family — not household residents.

**Join event:** Fires `zen_event kind: family_member_joined` with `prompt_profile_update: true`.

---

### Remove a Member from a Family

```yaml
script.zen_dojotools_identity:
  mode: family_remove_member
  family_entity: sensor.<family_cabinet>
  member_entity: sensor.<cabinet>
  member_type: user   # user | ai_user | family
```

**Default family cleanup:** If the removed family was the member's `default_family_guid`, that field is cleared. The member must explicitly set a new default via `set_default_family`.

**Leave event:** Fires `zen_event kind: family_member_left` with `prompt_profile_update: true` and `was_default_family` flag.

---

### Deprovisioning a Family Cabinet

> ⚠️ **Order matters.** `deprovision` does **not** remove a family from the household. Skipping the identity teardown steps first leaves a stale entry in the household's `members.families` list — re-provisioning and re-adding the family creates a duplicate.

Correct teardown sequence:

```yaml
# 1. Remove all members from the family
script.zen_dojotools_identity:
  mode: family_remove_member
  family_entity: sensor.<family_cabinet>
  member_entity: sensor.<member_cabinet>
  member_type: user  # repeat for each member

# 2. Remove the family from the household
script.zen_dojotools_identity:
  mode: household_remove_family
  family_entity: sensor.<family_cabinet>

# 3. Deprovision member cabinets (if applicable)
script.zen_dojotools_provisioner:
  mode: deprovision
  cabinet_entity: sensor.<member_cabinet>

# 4. Deprovision the family cabinet
script.zen_dojotools_provisioner:
  mode: deprovision
  cabinet_entity: sensor.<family_cabinet>
```

---

### Link Partners (Delegation Authority)

**Partner in this model means: authorized to delegate on your behalf.** This is a governance relationship, not a social one. No token is issued without an explicit allow — the link records who has delegation authority, it does not grant it automatically.

Works for any entity pair:

```yaml
script.zen_dojotools_identity:
  mode: link_partners
  member_entity: sensor.<cabinet_a>   # any type: user, ai_user
  ai_entity: sensor.<cabinet_b>       # any type: user, ai_user
```

Writes to `VolumeInfo.acls.partner[]` on **both** entities. Each entry carries `{guid, entity_id, cab_type, role: partner, sid}`.

Idempotent — re-running does not create duplicate entries.

**Unlink:**

```yaml
script.zen_dojotools_identity:
  mode: unlink_partners
  member_entity: sensor.<cabinet_a>
  ai_entity: sensor.<cabinet_b>
```

Removes each entity from the other's `acls.partner[]`.

---

### Change a User's Default Family

```yaml
script.zen_dojotools_identity:
  mode: set_default_family
  member_entity: sensor.<user_or_ai_cabinet>
  family_entity: sensor.<family_cabinet>
```

Patches `default_family_guid` in the member's `VolumeInfo`. The first family joined is set automatically; use this to change it explicitly.

---

### Set or Replace a Principal (HoH or Prime AI)

```yaml
script.zen_dojotools_identity:
  mode: set_principal
  member_entity: sensor.<cabinet>
  member_type: user      # user → fills/replaces acls.owner (HoH)
                         # ai_user → fills/replaces acls.partner prime slot
  # family_entity: sensor.<family_cabinet>  # optional — targets family instead of default household
```

Sets or replaces the Head of Household (`member_type: user`) or prime AI partner (`member_type: ai_user`) for the target container cabinet.

**Targets:** Defaults to `zen_default_household_cabinet`. Pass `family_entity` to target a specific family cabinet instead — the same slot semantics apply.

**Replacement:** Previous occupant of the slot is replaced. For the prime AI slot, any non-prime partner entries are preserved.

---

## Targeted Repairs (Small / Mid Nuke)

Use these when one identity or cabinet is broken but the rest of the system is healthy.

### Repair: Reseed a Cabinet's VolumeInfo Header

If a cabinet is showing schema errors or a missing GUID in `sensor.zen_cabinet_health`:

```yaml
script.zen_admintools_cabinetadmin_factory:
  target_cabinet: sensor.<cabinet_entity>
  cabinet_type: "AI Data Storage Cabinet"  # or relevant type
  confirm_action: true
```

---

### Repair: Soft-Reset a Single Cabinet

Wipes all drawers from one cabinet without touching labels or other cabinets:

```yaml
script.zen_admintools_cabinetadmin:
  mode: reset
  target_cabinet: sensor.<cabinet_entity>
```

After reset, re-run the provisioner (provision mode) to re-initialize it, or write profile data directly via the profile editor.

---

### Repair: Fix a Broken Label Assignment

If a cabinet resolver shows `unavailable` and a `zen_resolver_refresh` didn't fix it:

```yaml
# Soft label reset — wipes entity assignments from all zen_ labels, labels survive
# Flynn re-assigns via Gate 1 on the next health sensor change
script.zen_dojotools_labels:
  action_type: reset
  confirm: true
```

Then fire:
```
zen_resolver_refresh
```

If the cabinet is still stuck after soft reset, try a targeted re-tag:

```yaml
script.zen_dojotools_labels:
  action_type: tag
  label: zen_ai_user      # or zen_user, zen_family, etc.
  entity: sensor.<cabinet_entity>
  confirm: true
```

---

### Repair: Rebuild the Identity Manifest

If `sensor.zen_agent_health` shows identity manifest missing or stale:

```yaml
script.zen_dojotools_identity:
  mode: build_identity_manifest
```

Or fire the force event:
```
Developer Tools → Events → Event type: zen_event
Event data: {"kind": "identity_manifest_rebuild", "component": "system"}
```

---

## Full Wipe and Restart

For a complete teardown of the identity layer — all users, AI users, labels, and Ring-0 cabinets — follow the nuclear sequence in [Troubleshooting — Step 7](troubleshooting.md#step-7--nuclear-cabinet-reset-last-resort).

**Order matters:**
1. `script.zen_admintools_reset_labels` (confirm: true) — nukes all `zen_` labels
2. `script.zen_admintools_cabinetadmin` (op: reset_all, confirm: true) — wipes all Ring-0 cabinets, reseeds schemas, fires Flynn bootstrap

> **User cabinets are not touched** by the nuclear sequence. Only Ring-0 canonical cabinets (Dojo, Kata, System, Household, etc.) are reset. If you need to wipe a specific user or AI user cabinet, use `cabinetadmin mode: reset` against that cabinet directly before or after the nuclear sequence.

After the nuclear sequence completes, re-provision each identity cabinet via the provisioner and re-write profiles via the profile editor. Flynn handles label rebuild and schema seeding automatically.

---

## Quick Reference

| Operation | Tool | Mode |
|---|---|---|
| Add AI user | `zen_dojotools_provisioner` | `provision`, `cab_type: ai_user` |
| Remove AI user | `zen_dojotools_provisioner` | `deprovision` |
| Move AI user to new cabinet | `zen_dojotools_provisioner` | `replace` |
| Write / update AI user profile | `zen_dojotools_profile_editor` | `write`, `target_type: ai_user` |
| Add person (has HA account) | `zen_dojotools_provisioner` | `provision`, `cab_type: user` |
| Add external family member (no HA account) | `zen_dojotools_identity` | `provision_member` |
| Remove person | `zen_dojotools_provisioner` | `deprovision` |
| Move person to new cabinet | `zen_dojotools_provisioner` | `replace` |
| Write / update user profile | `zen_dojotools_profile_editor` | `write`, `target_type: user` |
| Wire family into household | `zen_dojotools_identity` | `household_add_family` |
| Add member to household | `zen_dojotools_identity` | `household_add_member` |
| Remove member from household | `zen_dojotools_identity` | `household_remove_member` |
| Add member to family | `zen_dojotools_identity` | `family_add_member` |
| Remove member from family | `zen_dojotools_identity` | `family_remove_member` |
| Link partners (delegation authority) | `zen_dojotools_identity` | `link_partners` |
| Unlink partners | `zen_dojotools_identity` | `unlink_partners` |
| Change default family | `zen_dojotools_identity` | `set_default_family` |
| Set/replace HoH or prime AI | `zen_dojotools_identity` | `set_principal` |
| Transfer default label | `zen_dojotools_labels` | `untag` + `tag` |
| Reseed cabinet header | `zen_admintools_cabinetadmin_factory` | — |
| Reset one cabinet | `zen_admintools_cabinetadmin` | `reset` |
| Reset label assignments | `zen_dojotools_labels` | `reset` |
| Rebuild identity manifest | `zen_dojotools_identity` | `build_identity_manifest` |
| Full nuke | See [Troubleshooting](troubleshooting.md) | Steps 5–7 |

---

## Related Docs

- [DojoTools Identity](../scripts/zen_dojotools_identity_readme.md) — mode reference and membership graph rules
- [DojoTools Profile Editor](../scripts/zen_dojotools_profile_readme.md) — profile drawer writer
- [Cabinet Specification](../cabinets/cabinet_spec.md) — valid cabinet/person/family/household/AI-user shapes
- [Troubleshooting](troubleshooting.md) — graduated repair sequence and nuclear reset path
- [Getting Started Overview](readme.md) — return path to the first-run suite
