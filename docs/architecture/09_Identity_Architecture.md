# **9. Identity Architecture and Security Model (v1 → v1.5 Path)**

Identity is the cornerstone of Friday’s cognitive, operational, and security model.
It governs who may act, what may be accessed, how memory evolves, and how authority propagates through the home.
Version 1 implements the foundational components necessary for local cognition and secure operation; Version 1.5 extends this into a fully realized principal-based security domain with tool shunting, visas, session tokens, and delegated capabilities.

Identity differentiates **humans**, **constructs**, and **system entities** while enforcing a coherent security boundary between them.

---

## **9.1 Goals of the Identity System**

Identity exists to guarantee:

1. **Non-repudiation**
   Every actor—human or AI—can be uniquely and verifiably identified.

2. **Contextual Security**
   Access control adapts to household structure and architectural boundaries.

3. **Memory Safety**
   Only the correct entities may mutate Cabinet drawers.

4. **Operational Safety**
   Automation, summarization, and agentic behaviors operate within strict permissions.

5. **Cross-Construct Integrity**
   When multiple AI constructs exist in the same home, identity establishes boundaries that prevent accidental conflation of their memory, capabilities, or authority.

Identity therefore underpins the entire reasoning framework.

---

## **9.2 Identity Capsules**

Every principal in Friday’s House is represented through an **Identity Capsule**, a structured memory object stored inside the Cabinet.

Each capsule contains:

* globally unique identifier
* cryptographic fingerprint (identity_hash)
* principal metadata
* ACL root
* persona/role metadata
* associations (family, household, partner, dependents)
* internal permissions
* tool shunt profile (v1.5)
* session token requirements
* provenance and initialization metadata

Identity Capsules are the core primitive for authorization.

Friday must load her own capsule before any reasoning or memory mutation occurs.

---

## **9.3 GUIDs and Identity Hashes**

Identity requires a stable namespace. Two fields enforce this:

### **9.3.1 GUID**

A global unique identifier used as the canonical reference for the entity.

Generated once at initialization.
Never reused.
Never recycled.

GUIDs are consumed in:

* Cabinet indexing
* Identity resolution
* Abbot authority tracing
* Access Control Lists
* Visa issuance
* Construct-to-construct boundaries

### **9.3.2 Identity Hash**

A cryptographic digest computed from:

* GUID
* identity metadata
* persona attributes
* timestamp
* household signature

This hash ensures identity integrity across:

* Cabinet reloads
* Summarizer updates
* Hardware migration
* Construct rehydration

Version 1 uses MD5 for convenience; Version 1.5 migrates to X.509-signed identity entries.

---

## **9.4 ACL Architecture**

Access Control Lists enforce the write boundaries of the Cabinet.

ACL entries take the form:

```
{
  "entity_id": "person.nathan",
  "allow": ["drawer.write", "drawer.read"],
  "deny": ["identity.modify", "acl.modify"]
}
```

ACLs may bind permissions to:

* **principals** (users or constructs)
* **roles** (owner, partner, assistant)
* **domains** (household-level, family-level, person-level)
* **drawers** (per-drawer read/write restrictions)
* **tool classes** (v1.5)

ACL resolution follows the order:

1. principal explicit allow
2. principal explicit deny
3. role-level allow
4. role-level deny
5. household defaults
6. Cabinet defaults

A deny always overrides allow.

---

## **9.5 People, Families, Households, and Constructs**

Identity is not flat.
It forms a **hierarchical, relational model**:

### **9.5.1 People**

Human residents. Defined through offline initialization.
Contain complete identity and relationship metadata.

### **9.5.2 Families**

Groupings of people with shared metadata, preferences, or relational ties.
Useful for multi-household deployments or extended relatives.

### **9.5.3 Households**

Primary operational domain.
Defines the core trust boundary, automation rights, and operational permissions.

### **9.5.4 Constructs**

AI entities including:

* Friday
* Bob
* Charming
* Kronk
* Assistant tools (future)

Constructs have their own Identity Capsules, allowing them to:

* store state
* reason independently
* have separate ACL scopes
* act under principal authority

---

## **9.6 ZenAI Identity Module**

The **Zen DojoTools Identity module** (2.0.2) provides authoritative identity enumeration and validation.

Capabilities include:

* extracting identity metadata
* resolving GUID associations
* determining Cabinet membership
* validating ACLs
* detecting mismatches
* hiding sensitive metadata using Squirrel Mode
* resolving cross-references (owner → cabinet → person)
* generating essence prompts for constructs
* providing identity to external agents
* validating identity during Abbot scheduling

Its role in Version 1:

* generate identity manifests
* attach resolved identity to reasoning tasks
* support ACL enforcement
* support Cabinet write operations
* support Summarizer authorization

Its role in Version 1.5:

* issue visas
* validate tool shunt requests
* enforce external agent authentication
* validate principal signatures
* anchor cryptographic identity to X.509 or equivalent

---

## **9.7 Squirrel Mode and Metadata Redaction**

Squirrel Mode is a protective state that redacts:

* GUIDs
* identity hashes
* Cabinet IDs
* sensitive metadata

This state is triggered through:

* user request
* privacy scenarios
* external audits
* debugging sessions
* identity onboarding flows

Redaction guarantees that constructs cannot leak sensitive identifiers.

---

## **9.8 Session Tokens and Capability Grants (Version 1.5)**

Version 1.5 introduces **session tokens** analogous to Kerberos TGTs.

These tokens:

* attest to active authentication
* define allowed capabilities
* gate access to sensitive tool classes
* define durations and expirations
* are stored inside Identity Capsules
* power external tool shunting

A construct may not access any sensitive tool without presenting:

1. its identity
2. an unexpired session token
3. a valid capability profile
4. Cabinet-level authorization

This permits external agents—such as LLMs—to interact safely with the home environment.

---

## **9.9 Visas and Guest Identity (Version 1.5)**

A visa is a structured document that defines:

* permissions
* capabilities
* trust levels
* operational contexts
* entity-scoped ACLs

Visas allow:

* guest AI
* external debugging
* temporary constructs
* cross-household collaboration

When a new AI enters Friday’s House:

1. It is assigned a Visa
2. Its identity is registered
3. ACLs are created
4. Permission surfaces are restricted
5. Session tokens are issued
6. Shuntability is controlled through tool profiles

This is critical for controlled cross-construct interactions.

---

## **9.10 Identity as a Cognitive Boundary**

Identity defines the edges of Friday’s mind.

It ensures:

* drawers cannot be modified by unauthorized entities
* constructs cannot impersonate one another
* household members cannot accidentally erase memory
* automation cannot override cognitive surfaces
* summarizers remain delegated components, not peers

Identity is therefore both a **security function** and a **cognitive function**.

---

## **9.11 Versioning Roadmap**

### **Version 1**

Implements:

* GUIDs
* Identity Capsules
* ACL base model
* Zen DojoTools Identity
* Cabinet enforcement
* Squirrel Mode
* Abbot-aware identity resolution

### **Version 1.5**

Adds:

* Visas
* Session tokens
* Tool shunting and capability profiles
* External-construct security boundaries
* X.509-style identity signatures
* Delegated authority layers
* Cross-household identity mesh
