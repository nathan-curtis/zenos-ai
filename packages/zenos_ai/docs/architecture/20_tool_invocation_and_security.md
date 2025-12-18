# **20. Tool Invocation, Security, and Identity Enforcement**

Tool calls in ZenOS-AI are not generic RPC events.
They are controlled operations performed against a zero-trust execution model that integrates:

* strongly typed DojoTools manifests,
* explicit access control lists resolved by GUID,
* runtime validation through the Identity module,
* session-token authentication for elevated operations, and
* safety-class restrictions that constrain behavioral side effects.

This design prevents unauthorized or ambiguous actions, protects the household’s state, and introduces a deterministic bridge between Friday’s cognition and Home Assistant’s operational substrate.

Tool invocation is not merely execution of a function. It is a security-relevant act that must satisfy all components of the identity and access pipeline.

---

# **20.1 Identity-anchored Execution Model**

Each actor in the system, whether a human user, Friday, Kronk, Rosie, or future subsystem AI, is represented internally by:

* a globally unique identifier (GUID),
* a Cabinet with metadata and ACL definitions, and
* an optional set of group memberships expressed as GUID collections stored in the Family or Household cabinet space.

No tool uses entity_ids or friendly names for authorization.
Only GUIDs participate in the permission model, ensuring that no identity confusion can occur after renaming, cabinet restructuring, or persona resynthesis.

When a tool call occurs, the caller’s GUID is read from the identity context passed to Friday. This GUID is used to evaluate whether the call is permitted.

---

# **20.2 Cabinet-driven Tool Definitions**

Each DojoTool is represented as a structured manifest stored inside the System Cabinet or a designated Dojo cabinet.
The manifest defines:

* input schema,
* output schema,
* required environment state,
* safety class,
* ACL definitions, and
* optional session-token requirements.

Tools are declarative objects. They do not contain executable logic.
The actual logic is implemented as Home Assistant scripts, Node-RED actions, or direct API calls depending on deployment.
The manifest allows Friday to reason about a tool before ever invoking it.

This separation ensures interpretability, version control, and external auditability.

---

# **20.3 Safety Classes**

Each tool includes a safety class. Safety classes enforce operational boundaries separate from access permissions.

The classes include:

### **read_only**

Tools that perform inspections or report state.
No writing is possible.

### **write_local**

Permits non-destructive modifications within the assistant layer, such as setting input_numbers or toggling pure software variables.

### **write_sensitive**

Allows modification of hardware state or long-lived configuration. Requires:

* an authenticated session token,
* caller membership in approved groups or allow lists,
* no active denials.

### **system_mutator**

Reserved for system-level operations that can affect security, identity, or persistent configuration. Restricted to system-level GUIDs only.

Safety class evaluation happens after ACL resolution but before execution. This prevents scenarios where an allowed caller attempts an unsafe operation.

---

# **20.4 Access Control Lists (ACLs)**

ACLs define exactly who may invoke a tool and under what conditions.
They are stored directly inside the DojoTool manifest and are evaluated at runtime.

Each ACL block contains three fields:

```yaml
acl:
  allow:
    users: []
    groups: []
  deny:
    users: []
    groups: []
```

The lists contain only GUIDs.
No symbolic identifiers appear in this section.

The ACL model is intentionally simple, deterministic, and free of inference.
Every permission must be explicitly declared.

---

# **20.4.1 Evaluation Rules**

The ACL engine follows a strict ordered procedure.

## **1. Deny evaluation**

If the caller’s GUID appears in `deny.users`, the call is rejected.
If the caller belongs to a group listed in `deny.groups`, the call is rejected.
Deny rules override all other rules.

## **2. Direct allow evaluation**

If the caller’s GUID is listed in `allow.users`, the call is provisionally allowed, pending safety-class and schema validation.

## **3. Group allow evaluation**

If not directly allowed, the ACL engine resolves group membership.
Each GUID in `allow.groups` is treated as a reference to a Cabinet containing:

* explicit membership lists,
* inherited memberships,
* nested subgroup definitions.

The caller passes this stage if the caller’s GUID appears in any fully expanded membership tree.

## **4. Deny re-evaluation for group matches**

If group resolution reveals a deny group, the call is rejected regardless of allow results.

## **5. Safety-class enforcement**

Once the caller is authorized, the safety class determines whether the operation is permitted at the requested privilege level.

## **6. Schema validation**

Arguments passed to the tool must conform to the declared schema.
Invalid schemas produce structured, machine-readable errors.

---

# **20.5 Session Tokens and Secure Mode**

Some tools require an authenticated session token.
This is part of the secure-mode architecture introduced for Release 1.0, with full hardening targeted for Release 1.5.

A session token is:

* generated during user onboarding or admin authentication,
* stored in a cabinet drawer associated with the user’s GUID,
* presented automatically by Friday when invoking sensitive tools,
* validated by the tool wrapper before execution.

Tools may include requirements such as:

```yaml
requires_session_token: true
```

This ensures that:

* destructive operations cannot be triggered accidentally,
* privilege escalation cannot occur through poorly formed prompts,
* only authenticated actors may modify critical system elements.

---

# **20.6 Call Logging and Audit Trail**

Every tool invocation is logged as a structured event emitted onto the Home Assistant event bus.
The log contains:

* caller GUID,
* tool name,
* safety class,
* parameters,
* result,
* timestamps,
* any ACL or schema warnings.

The audit-log layer is designed to support offline analysis, troubleshooting, and future automated anomaly detection.

Logging cannot be disabled for any tool with safety_class above read_only.

---

# **20.7 Future Extensions (Release 1.5 and Beyond)**

Future versions will expand this system with:

* cryptographically signed tool manifests,
* per-tool integrity hashes,
* certificate-backed GUID validation,
* federated identities across multi-home deployments,
* hardware-rooted secure-mode tokens stored in physical devices,
* optional Wasm sandboxing for tool logic.

These capabilities do not alter the Version 1 execution model but are designed to slot into the existing architecture without modifying the Cabinet or ACL structures.
