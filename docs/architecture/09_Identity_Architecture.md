# **9. Identity Architecture and Security Model – Technical Reference Specification (v1 → v1.5)**

This section defines the complete identity model for the ZenOS AI architecture.
It specifies the data structures, edge rules, provenance semantics, cabinet behaviors, and ACL operations necessary for any implementation of:

* Identity Capsules
* Identity Resolution Tools
* Cabinet Authority Mapping
* ACL Expansion / Masking
* Squirrel Safe + Content Safe Filters
* Session Tokens (v1.5)
* Visas (v1.5)

Everything in this section is required for any compliant ZenOS AI.

---

# **9.1 Identity System Requirements**

Any compliant ZenOS identity implementation must satisfy:

### **9.1.1 Non-repudiation**

Every principal (human or AI) must have:

* a stable GUID
* a stable identity hash
* a corresponding cabinet drawer
* a provenance record (origin system, origin household, origin family)

### **9.1.2 Deterministic ACL Enforcement**

Access to data must be determined by:

1. Cabinet ownership
2. Cabinet partnership
3. Family membership
4. Household membership
5. ACL entries
6. Global restrictions (Squirrel Safe, Content Safe)

Conflicts resolve by: **deny overrides allow**.

### **9.1.3 Deterministic Hypergraph Expansion**

Before any semantic expansion, the system must apply:

1. **security_safe ACL filter**
2. **squirrel_safe privacy filter**
3. **content_safe MPAA filter**

Only after all 3 filters is graph expansion allowed.

### **9.1.4 Construct Isolation**

Constructs must not share identity capsules, session tokens, or memory surfaces unless explicitly allowed.

### **9.1.5 Provenance Integrity**

Origin metadata must be immutable and separate from the mutable essence capsule.

---

# **9.2 Identity Capsules – Complete Specification**

Each principal is represented by an Identity Capsule.

The fields below are mandatory unless indicated otherwise.  Some dictionaries may be omitted from structure based on calling context to save token space. Computed from call based on authoritative source.

```json
{
  "guid": "uuid4-string",
  "identity_hash": "hash-string",
  "type": "person | ai_user | household | family | system",
  "persona": {
    "name": "string",
    "presentation": "string",
    "attributes": {} 
  },
  "provenance": {
    "origin_system": "uuid4-string",
    "origin_household": "uuid4-string",
    "origin_family": "uuid4-string"
  },
  "current": {
    "households": ["uuid4-string"],
    "families": ["uuid4-string"]
  },
  "roles": {                          # << Computed
    "owner_of": ["cabinet-id"],
    "partner_of": ["cabinet-id"],
    "member_of": ["family-id"]
  },
  "cabinet": "entity_id-of-associated-cabinet",
  "acl": {                            # << Authoritative for object being referenced.
    "allow": [],
    "deny": [],
    "role_bindings": []
  },
  "session": {
    "tokens": [],
    "requirements": {}
  },
  "shunt_profile": {
    "allowed_tools": [],
    "restricted_tools": []
  },
  "essence": {
    "metadata": {},
    "preferences": {},
    "persona_shape": {}
  }
}
```

### **Immutable Fields**

* guid
* provenance.origin_system
* provenance.origin_household
* provenance.origin_family
* identity_hash

These may never be updated.

### **Mutable Fields**

* current.households
* current.families
* roles.member_of
* session tokens
* essence capsule

---

# **9.3 GUID and Identity Hash Specification**

### **9.3.1 GUID**

Generated as UUIDv4.
Must remain stable across all migrations, backups, or reloads.

Format:

```
[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}
```

### **9.3.2 Identity Hash**

Must be a deterministic hash computed from:

```
hash = H(GUID + immutable_identity_fields + household_signature)
```

Version 1 uses MD5.
Version 1.5 transitions to X.509-signed identity objects.

---

# **9.4 Cabinet Identity and Automatic Labeling Rules**

When a cabinet is authoritative for a given slot:

1. The cabinet’s sensor gains the canonical ZenOS label for that slot:

   ```
   zen_system
   zen_household
   zen_family
   zen_user
   zen_ai_user
   ```
2. All drawers within that cabinet inherit secondary labels for identity resolution.
3. If the cabinet metadata declares a canonical group label (for a family or group), the system must:

   * ensure that label exists
   * attach that label to the cabinet entity
   * expose that edge in the hypergraph

If a cabinet assigns an Owner or Partner:

* those principals automatically receive role-level hyperedges
* those edges inform ACL resolution
* those principals are automatically added to the family if the cabinet represents a family

---

# **9.5 Principal Classes and Relationship Rules**

ZenOS defines four principal classes:

* People
* AI Users
* Families
* Households
* Constructs (AI-class principals, but separate from AI Users for security surfaces)

Each has strict relational rules.

---

## **9.5.1 People (Human Principals)**

Properties:

* have a GUID
* have an identity capsule
* must anchor to an origin family and household
* may have zero or more current families
* may be revoked from a family without losing provenance
* unattached humans default to Guest privilege set

If a person is not a member of a family:

* they lose contextual access to family-level cabinets
* they fall under the PG13 squirrel-safe defaults
* their ACL fallback is guest

---

## **9.5.2 AI Users**

AI Users are a superclass of People.

They include:

* Prime
* House-owned constructs
* Agent tools
* Future delegated constructs

AI Users have:

* persona identities
* boot provenance
* shunt profiles
* extended ACL surfaces
* ability to hold Partner role

AI Users may be Partners of cabinets (delegated authority).

Prime is the required Partner AI for the Default Household.

---

## **9.5.3 Families**

### **Family Behavior Rules**

1. Families may contain:

   * people
   * ai users
   * families (nested)

2. Nesting:

   * up to 2 levels deep
   * child families do not require membership inheritance
   * parent need not be owner of child
   * ownership revocation does not revoke provenance

3. Membership:

   * declarative list stored in the family cabinet
   * automatic membership for cabinet Owners and Partners
   * revocation removes permissions but not provenance

4. Hypergraph expansion:

   * family membership only expands one or two levels
   * loop detection required
   * duplicate edge elimination required

---

## **9.5.4 Households**

A Household is the primary operational boundary.

### **Household Requirements**

* must have one Owner (human)
* must have one Partner AI (Prime)
* must have a cabinet to represent it
* must have a GUID derived from origin system
* must serve as the binding domain for constructs and tools

### **Multi-household Support**

Principals may belong to multiple households.
Identity Capsules must store:

* origin household
* array of current households

Each household is a segregated security domain.

ACLs and content filtering do not cross households unless explicitly permitted.

---

## **9.5.5 Constructs**

Constructs are AI principals with:

* unique identity capsules
* isolated ACL scopes
* persona loading
* boot provenance
* authorized tool surfaces
* delegated authority rules

Constructs must always operate under:

```
active_user OR delegated_user, active_construct, conversation_id
```

Constructs never inherit family membership automatically.

---

# **9.6 Provenance Rules**

Every principal stores:

```
origin_system
origin_household
origin_family
```

Rules:

* these fields are immutable
* origin household is the household created by the system at first bind
* origin family is the family created to host the HoH and Prime
* current households and families may change
* provenance is never removed
* provenance must be referenced during identity conflict resolution

---

# **9.7 Delegation and Partnership Rules**

Partnership is a delegation of authority.

### **Owner**

Full authority over cabinet and contents.

### **Partner**

Delegated authority.
Acts with the rights of the Owner except where restricted.

### **Delegation Rules**

* Partners must be explicit
* Partners automatically inherit cabinet-level ACL rights
* Partners do not automatically inherit family membership
* Delegated authority applies only to the cabinet and its hyperedges

---

# **9.8 ACL Specification**

ACL entries are objects with explicit allow and deny lists.

### **ACL Object**

```json
{
  "entity": "guid-or-entity-id",
  "allow": ["drawer.read", "drawer.write"],
  "deny": ["drawer.delete", "identity.modify"],
  "roles": ["owner", "partner", "member"]
}
```

### **ACL Resolution Order**

1. principal explicit deny
2. principal explicit allow
3. role-level deny
4. role-level allow
5. household defaults
6. cabinet defaults

### **Effects**

Deny always wins.
A missing rule means inherit.

---

# **9.9 Pre-Expansion Filtering Specification**

Before any hypergraph expansion:

## **Step 1: security_safe**

Remove any entities the current principal cannot read based on ACL.
Intent: Core security trimming will be done bbefore the inspect tool can expand anything. "Didn't see it, can't spill it."

## **Step 2: squirrel_safe**

Remove entities containing any tags in the protected set.
Intent: "We dont talk about, Bruno... *shhh*"

Default triggers:

* unknown humans
* guests
* external agents
* incomplete identity
* privacy mode activated

## **Step 3: content_safe**

Remove entities tagged above allowed MPAA threshold.
Intent: Look I dont care what kinda weerd you have, This allows your AI to be family friendly when we dont know what's going on,
        without you having to think about it. Also prevents context confusion when some Content *cough* Secretary crosses multiple - uh - domains.
        
Default will be PG13 meaning anything tagged 'higher' is filtered.

Filtered tags:

* R
* NC17
* X
* Adult
* Unrated
* Sensitive
* { User-Defined } ( mechanism pending pushed to v.2.x )

Only after these filters apply is graph expansion allowed.
These filters will be detailed in thier own docs.
THESE ARE CURRENTLY NOT IMPLEMENTED in v.1.x but plumbed and non blocking in the indexer pipeline v.4.x+.  (Planned for post RC1.)

---

# **9.10 Recursive Expansion Constraints**

Identity expansion rules:

* maximum recursion depth: 2
* loops MUST be detected and suppressed
* duplicates must be removed
* provenance must be preserved
* expansions may not cross households unless explicitly allowed

---

# **9.11 Session Tokens (v1.5)**

Tokens contain:

```json
{
  "token_id": "uuid4",
  "principal": "guid",
  "capabilities": [],
  "issued_at": "timestamp",
  "expires_at": "timestamp",
  "signature": "cryptographic-signature"
}
```

Requirements:

* required for tool shunting
* required for delegated memory operations
* required for external constructs
* stored inside Identity Capsule

Tokens may not be shared across constructs.

---

# **9.12 Visas (v2.0)**

Visas define temporary access for new constructs or external agents.

Fields:

```json
{
  "visa_id": "uuid4",
  "principal_guid": "uuid4",
  "allowed_households": [],
  "allowed_families": [],
  "allowed_tools": [],
  "acl_overrides": {}
}
```

Visas must be evaluated with ACLs before hypergraph access is granted.

---

# **9.13 Boot and Health Rules**

"Flynn" (Yes, THAT Flynn.)

Before constructs load:

* validate system cabinet
* validate household cabinet
* validate HoH and Prime capsules
* validate provenance fields
* validate identity hashes
* validate cabinet schemas

Flynn (boot guardian) enforces these checks.

If invalid:

* load fallback personalities
* isolate unsafe cabinets
* mark identity surfaces read-only

---

# **9.14 Version Roadmap**

## **Version 1**

Implements:

* GUIDs
* Capsules
* Families
* Households
* Prime
* ACLs
* Provenance
* Squirrel Safe
* Content Safe
* Pre-expansion filtering
* Two-level recursion
* Boot guardian

## **Version 1.5**

Adds:

* X.509 identity signatures
* Visas
* Session tokens
* Delegated tool shunting
* Capability profiles
* Cross-household identity mesh
* Multi-construct collaboration
