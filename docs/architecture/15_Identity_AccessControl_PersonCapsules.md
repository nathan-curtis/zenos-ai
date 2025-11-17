# **15. Identity, Access Control, and the Person Capsule Architecture**

Identity is the organizing principle of Friday’s cognitive system.
It determines who actions apply to, who has authority, how context is personalized, how safety constraints propagate, and how the home differentiates one human from another at every level of reasoning.

The Identity subsystem in Friday’s House is fully local, cryptographically structured, cabinet-backed, and explicitly permissioned.
It is not a toy abstraction.
It is the foundational mechanism that prevents the system from becoming anthropomorphically naive or dangerously permissive.

Identity is built on three pillars:

1. **The Identity Cabinet (User / Family / Household Cabinets)**
2. **The Person Capsule**
3. **The Access Control Layer (ACLs + Identity Hash + Squirrel Mode + User-Class Routing)**

Together, these form a deterministic identity lattice that every cognitive function relies on.

---

# **15.1 Identity in a Local Cognitive System**

In agentic architecture, identity cannot be assumed.
It must be:

* registered
* verified
* correlated
* and continuously updated

Failure to do so leads to misaligned reasoning:

* actions targeted to the wrong user
* summarizers overwriting each other's state
* permissions drifting over time
* security-critical operations executed without clear ownership

In Friday’s House, identity is not a prompt convention.
It is a **model-backed set of Cabinet volumes with verifiable structure**.

The Identity system is the first line of defense against misalignment.

---

# **15.2 The Identity Cabinet**

Every person, family, household, and construct is represented by a Cabinet volume with the following invariants:

* **Globally Unique Identifier (GUID)** stored in VolumeInfo
* **validation signature** (“ALLYOURBASEBELONGTOUS”)
* **versioned schema** for repeatable parsing
* **flags** describing cabinet type (user, family, household, system, dojo, kata)
* **zenai_essence drawer** containing identity metadata
* **_zen_relationships drawer** describing relational context
* **ACL drawer** describing allowed abilities
* **context metadata** describing parameterization (friendly name, description, version)

Identity cannot exist without a Cabinet, and a Cabinet cannot exist without a GUID.

This circular dependency is deliberate.
Identity is required to load cognition, cognition is required to maintain identity.

The Cabinet is Friday’s ground truth.

---

# **15.3 The Identity Hash: Integrity Anchor**

Each identity cabinet includes an **identity_hash** field derived through a deterministic hashing pipeline applied to:

* GUID
* registered entity_id
* allowed ACLs
* declared relationships
* core essence metadata

This allows Friday to detect:

* foreign cabinet injections
* unauthorized modifications
* drift between HA entity and Cabinet representation
* replay or stale identity data

Identity hash mismatches do not silently resolve — the system raises flags that propagate into the Abbot and Summarizer layers.

This enforces identity integrity across updates.

---

# **15.4 Identity Classification**

Identity is categorized across four types:

### **15.4.1 User**

A real human represented by a person.* entity.

Properties:

* has human relationships
* receives full access through ACLs
* is allowed to generate top-level summaries
* can trigger cognitive paths
* owns or partners with constructs

### **15.4.2 Family**

A group construct representing kinship or shared context.

Properties:

* can store shared routines
* may contain partner relationships
* provides cross-user context resolution

### **15.4.3 Household**

The entire home-level context.

Properties:

* stores global preferences
* schedules
* ownership relationships
* cabinet manifest

### **15.4.4 System**

Internal AI constructs.

Properties:

* do not correspond to person entities
* may hold critical metadata
* carry restricted ACLs
* cannot execute actions externally

---

# **15.5 The Person Capsule**

The Person Capsule is the runtime identity model representing a user for Friday’s reasoning engine.

It contains:

* **GUID**
* **friendly name**
* **roles** (owner, partner, dependent, guest)
* **relationships** (with partner, household, children, constructs)
* **state-derived context** (awake/asleep, home/away, active/inactive)
* **ACLs**
* **history markers** (last spoken to, last in room, last summary applied to user)
* **identity hash validation status**

The Person Capsule is used for:

* identity routing
* personalized summarizers
* cognitive focus
* room-specific reasoning
* user safety evaluation
* permissions enforcement
* action relevance scoring

It provides the “who” dimension for the entire cognitive system.

---

# **15.6 ACL Design**

ACLs are declarative and cabinet-hosted.
They describe three broad domains:

### **Control ACLs**

What actions the entity is allowed to perform.

Examples:

* speak to summarizers
* request expansions
* trigger identity-affecting operations
* modify room conditions
* override house modes

### **Tool ACLs**

Which tools Friday may run on behalf of a user.

Examples:

* cabinet inspectors
* summarizers
* script-level actuators
* secret drawers

### **Relationship ACLs**

Cross-context permissions.

Examples:

* partner-level sharing
* parental-leadership mappings
* dependent protection rules

ACLs are immutable outside owner modification.

ACL enforcement occurs at two levels:

1. **Abbot** — prevents execution of unauthorized reasoning tasks.
2. **Summarizer** — ensures summaries do not write in unauthorized volumes.

This dual enforcement prevents accidental permission escalation.

---

# **15.7 Identity Routing**

Identity routing ensures that summaries and actions target the correct user.

Routing sources include:

* person.* entity state
* RoomState transitions
* motion-person correlation
* BLE tracker proximity
* ActiveRoom identity
* relationship propagation
* device ownership

Identity routing uses a weighted heuristic stack:

* explicit owner mappings (highest weight)
* partner relationships
* cabinet-level ownership
* last-known-room
* device association
* area historical patterns

False positives are filtered using decay curves, temporal windows, and cross-room conflict resolution.

Identity routing is critical to prevent misapplied cognitive actions.

---

# **15.8 Squirrel Mode: Identity Redaction**

Squirrel Mode is a special privacy mode that redacts:

* GUIDs
* identity hashes
* essence details

Used during:

* live demos
* external testing
* guest-visible logs
* privacy-sensitive operations

Redaction is applied at the Identity script layer, not during summarization.
Friday always sees full identity — only external surfaces are redacted.

---

# **15.9 Registration and Onboarding**

Identity registration follows this sequence:

1. **Cabinet creation**
2. **VolumeInfo assignment**
3. **GUID generation**
4. **ACL instantiation**
5. **relationship hooks established**
6. **essence drawer initialized**
7. **identity hash created**
8. **manifest updated**
9. **Person Capsule synthesized**
10. **context propagated**

For new users coming into Friday’s House:

* a Visa process will issue a TGT
* Cabinet entries will be created or imported
* tool shunt permissions will be generated
* contextual ACLs applied
* identity capsule activated

This onboarding pipeline will be introduced fully in v1.5.

---

# **15.10 Identity Construct Safety Model**

The identity system prevents:

* cross-user context contamination
* unauthorized summaries
* unintended personality load
* write operations into the wrong cabinet
* partner misattribution
* accidental propagation of stale GUIDs
* impersonation via HA person.* spoofing
* execution of privileged tools by non-owners

Identity is the core safety mechanism of Friday’s architecture.

Every higher-level cognitive feature depends on it.

---

# **15.11 Identity Summarization**

Identity also participates in summarization.
Summarizers treat identity as a structured component:

* user presence → triggers contextual updates
* role changes → update relationship models
* permission changes → propagate into Abbot
* cabinet modifications → refresh identity capsules
* partner context changes → update shared routines

Identity summaries preserve cognitive continuity through daily life.

---

# **15.12 Debugging Identity**

Identity debugging uses three tools:

1. **Zen Identity (verbose mode)**
   Full cabinet, essence, ACL, and relationship extraction.

2. **Zen Inspect**
   Device-level and attribute-level forensic analysis.

3. **Zen Index**
   Cross-entity correlation for cabinet resolution.

Identity is auditable all the way down to HA attributes.

This is essential for safe evolution of the system.

---

# **15.13 Future Identity Features (v1.5+)**

Later versions introduce:

* signed identity manifests
* cross-household identity federation
* Visa-based permission delegation
* session token propagation
* key-bound tool calls
* identity-backed inference

Identity will eventually become the system’s cryptographic foundation.
