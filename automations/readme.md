# ZenOS-AI Automations Directory  
This folder contains the **Home Assistant automations** that drive real-time orchestration for ZenOS-AI.  
Unlike the scripts directory—where DojoTools are called on demand—automations here run **autonomously**, reacting to system events and Cabinet changes without manual invocation.

These automations act as Friday’s “reflex arcs,” enabling background maintenance, routing, and event capture.

---

## 📡 Included Automations

### **1. dojotools_zen_index_event_handler.yaml**
This automation listens for index-related events published by the **Zen Index** toolset.

**Purpose:**
- Captures index update events  
- Routes changes to the appropriate drawers in the Cabinet  
- Ensures label and metadata sync  
- Maintains a fresh entity map for Friday, Kronk, and the Monastery

**Triggers:**  
- `zen_index_event`  
- DojoTools service calls  
- Entity metadata changes (depending on config)

**Dependencies:**  
- `dojotools_zen_index` script  
- Zen Labels  
- Spook (optional but recommended)

---

### **2. zen_dojotools_volume_redirector.yaml**
The Volume Redirector is the teleportation gatekeeper of the Cabinet system.  
Its job is to intercept writes to Cabinet volumes and redirect them to the correct drawers based on Index labels and manifest metadata.

**Purpose:**
- Dynamic volume routing  
- Drawer/volume resolution  
- Redirects writes from higher-level tool calls to the correct storage location

**Why it matters:**
Without this automation, **multiple tools could try to write to the same bucket**, and your Cabinets would lose structural integrity.  
This ensures smooth, predictable storage behavior—even as new tools and drawers are added.

**Dependencies:**  
- Zen Cabinet Manifest  
- FileCabinet Script  
- Zen Index

---

# 🔧 Installation Notes

1. Place both automations in your Home Assistant `/automations` folder.
2. Reload automations via:  
   **Settings → Developer Tools → YAML → Reload Automations**
3. Ensure you have also installed:  
   - Zen Index Kit  
   - File Cabinet Kit  
   - Spook (optional, for label operations)
4. **Do not rename** these automations unless you adjust internal references.

---

# 🧩 How Automations Fit Into ZenOS-AI

Think of the ZenOS-AI architecture as a mind:

- **Scripts** = deliberate actions  
- **Cabinets** = memory  
- **DojoTools** = skills  
- **Monastery** = reflection  
- **Automations** = reflexes  

This folder contains the reflexes.  
No deliberation.  
No prompting.  
Just pure, instantaneous operational flow.

---

# ☯️ Philosophy

Good automation stays out of the way,  
acts with clarity,  
and leaves perfect logs.

These two automations are small, but they’re essential to ensuring Friday’s world stays consistent, indexed, routable, and sane.

Welcome to the reactive layer of the Monastery.
