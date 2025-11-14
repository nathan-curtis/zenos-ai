# 🚀 **ZenOS-AI Cabinet System**

### *The Identity, Semantic, and Preference Architecture of the ZenOS Home AI*

```markdown
# ZenOS-AI Cabinet System
### Unified Identity, Settings, and Semantic Storage for the ZenOS Home AI

The ZenOS Cabinet System is the **core identity and context architecture** behind every Home AI instance.  
Cabinets serve as structured JSON storage units that define:

- Who exists in the system  
- How people, families, and AIs relate  
- What the home environment is  
- What rules and preferences apply  
- How the AI should interpret and act on context  

Every cabinet contains **drawers**, and every drawer contains **JSON**.  
This preserves structure, consistency, and performance.

---

# 🏛 Cabinet Hierarchy

ZenOS uses a 3-tier identity scaffold:

```

Household (Home)
└── Families
└── Users (Human or AI)

```

Each type has specific responsibilities, scopes, and override rules.

---

# 🏠 1. Household Cabinet  
### *The Home’s Settings, Identity, and Digital Twin*

The **Household Cabinet** (sometimes called the Home Cabinet) is the **canonical source of truth** for:

- Home-level settings  
- Zones, rooms, structure  
- The home’s **digital twin**  
- Semantic map (labels, tags, domains)  
- High-level behavior rules  
- System-wide preferences  
- The Prime AI (via `owner.partner_ai`)  

### **🔁 Mirrored with DOJO**
Each Kung Fu component in the Dojo Cabinet should have a **1:1 matching drawer** in the Household Cabinet.

Example:

- Dojo drawer: `water_manager`
- Household drawer: `water_manager`

This allows:
- one source of operational instructions (Dojo)  
- one source of runtime state and settings (Household)  

ZenOS merges both at runtime.

---

# 👪 2. Family Cabinet  
### *Lore, Identity, History, Trust, and Shared Culture*

A **Family Cabinet** represents a relational group inside the home.  
This is where ZenOS stores the *human* parts of the system:

- family structure  
- lore & shared history  
- norms & boundaries  
- trust and emotional dynamics  
- shared preferences  
- rituals, holidays, vibes  
- membership and relationship graph  

Families map directly into Households.

A home may contain:
- nuclear families  
- chosen families  
- polycules  
- guest circles  
- housemate clusters  
- “trusted tribe” groups  

These shape how the AI interacts with its residents.

---

# 🧍 3. User Cabinets  
### *Per-Person Identity, Preferences, Profiles, and AI Partners*

Every human and every AI construct gets a User Cabinet.

Contains:
- identity  
- personal profile  
- preferences  
- private boundaries  
- consent model  
- AI persona (if applicable)  
- partner AI (if they own one)  
- device associations  
- tags / labels  
- personal settings  

### **Override Rules**
As context narrows, **overrides apply**:

```

Household-level setting
→ overridden by Family setting
→ overridden by User setting

````

ZenOS determines final behavior through these cascading scopes.

Users may have:
- their own AI  
- their own preferences  
- their own permitted automations  

All private drawers stay private unless explicitly shared.

---

# 🧩 SYSTEM & DOJO = OS + Operational Instructions

### **SYSTEM Cabinet**
- OS kernel  
- Cortex  
- Directives  
- Persona rules  
- Runtime policies  
- Boot metadata  
- Hidden  
- Read-only  
- Never visible to frontline AIs  

### **DOJO Cabinet**
- Kung Fu components  
- Workflow definitions  
- Runbooks  
- Operating instructions  
- Safety / reflection rules  

DOJO = *“how to operate the home”*  
SYSTEM = *“what AI you are and what rules you obey”*

---

# 🔐 Security Model  
### *Cabinet-Level Context & Boundaries*

Cabinets define visibility boundaries:

- **public**: visible to all household AIs  
- **family-scoped**: visible only to AIs serving that family  
- **user-private**: visible only to the user & SYSTEM  
- **system**: visible only to backend processes  

Frontline AIs only access drawers allowed by their ACLs.

---

# 🔗 Mounts: Relationship Graph

Each cabinet can define **mounts**, linking to related cabinets:

```yaml
mounts:
  household: sensor.home_cabinet
  families:
    - sensor.family_primary
  users:
    - sensor.nathan_user
  partner_ai: sensor.friday_cabinet
````

This creates a **semantic graph** that ZenOS uses to understand:

* who belongs to whom
* who owns what
* where context flows
* how preferences propagate
* which AI is primary for the system

---

# 🏷 Labeling System

ZenOS labeling is **consistent across all cabinets**, allowing:

* single-pass scanning
* fast domain detection
* cross-cabinet rule application
* unified semantic understanding

Labels propagate through:

* drawers
* cabinets
* relationships
* zones
* Kung Fu components

One label read = full semantic insight.

---

# 🗂 JSON-First Storage

Everything in ZenOS is stored as **JSON inside drawers**, ensuring:

* structure
* simplicity
* compatibility
* schema validation
* easy parsing
* version tracking
* safe merges

All tools (FileCabinet, CabinetAdmin, DojoLoader) operate on this JSON.

---

# 🧰 Tooling

### **FileCabinet**

* single-source writer
* ensures schema correctness
* prevents unsafe writes
* enforces read-only flags
* timestamps updates

### **CabinetAdmin**

* repairs cabinets
* creates new ones
* aligns schema
* validates mounts
* formats JSON
* enforces cabinet-level security

Together, they ensure the cabinet ecosystem stays consistent and healthy.

---

## 📚 Additional Documentation

- **Zen Summarizer**  
  https://github.com/nathan-curtis/zenos-ai/blob/main/docs/zen_summarizer/readme.md

- **Kung Fu Components (Dojo Subsystems)**  
  https://github.com/nathan-curtis/zenos-ai/blob/main/docs/kung_fu/readme.md

- **FES Trigger-Based Template Sensors**

This is the basis of how cabinets work. The volume redirector ties together a group of sensors. 
  https://community.home-assistant.io/t/trigger-based-template-sensor-to-store-global-variables/735474

```
