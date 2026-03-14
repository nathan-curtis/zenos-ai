# ZenOS-AI: Install Guide

> **Version:** 4.2.0 | **Last Updated:** March 2026

---

## Prerequisites

- **Home Assistant** 2024.x or newer
- **Spook Integration** Installable through HACS. [Spook Install instructions](https://spook.boo/installation/)
- **A conversation agent** configured in HA with a tool-calling capable model. Models smaller than ~8B parameters or with short context windows are not recommended — ZenOS-AI prompts are large and tool use is required.
- SSH or filesystem access to your HA config directory

---

## Step 1 — Copy the Files

Copy two directories into your HA config:

**Packages:**
```
packages/zenos_ai/  →  <ha_config>/packages/zenos_ai/
```

**Jinja templates:**
```
custom_templates/zenos_ai/  →  <ha_config>/custom_templates/zenos_ai/
```

If `packages/` or `custom_templates/` directories don't exist in your HA config yet, create them.

---

## Step 2 — Enable Packages in configuration.yaml

Add this to your `configuration.yaml` if it isn't already there:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

This tells HA to load all YAML files under `packages/` as configuration packages.

If you already have a packages block, merge this in — don't add a second `homeassistant:` key.

---

## Step 3 — Add Required Secrets

ZenOS-AI requires a HA long-lived access token for its system tools. Add this to your `secrets.yaml`:

```yaml
ha_bearer: "Bearer <your-long-lived-token>"
```

Generate a token at **Profile → Security → Long-Lived Access Tokens** in your HA UI.

> **Plugin secrets:** If you install the Mealie or Grocy plugins, add these too:
> ```yaml
> mealie_bearer: "Bearer <mealie-api-token>"
> grocy_api_key: "<grocy-api-key>"
> ```
> Leave them out entirely if you're not using those plugins.

---

## Step 4 — Configure the Conversation Agent Prompt

ZenOS-AI uses a custom prompt template to build its context. You need to paste this into your conversation agent's system prompt.

1. Open `custom_templates/zenos_ai/conversation_agent_prompt_template.yaml`
2. Copy everything between the marked lines (the file is clearly annotated)
3. Paste it into your conversation agent's system prompt in HA

You can test the template output in **Developer Tools → Templates** before pasting.

---

## Step 5 — Restart Home Assistant

A full restart is required for packages and templates to load. A configuration reload is not sufficient.

After restart, watch the **Notifications** panel. Within a minute you should see one of:

- **"ZenOS-AI: System Ready"** — all gates passed, system initialized
- **"ZenOS-AI: Welcome — Let's name your AI"** — system initialized, OOBE pending (this is normal on first install)
- Nothing visible yet — check `sensor.zen_agent_health` and `sensor.zen_cabinet_health` in Developer Tools → States

If you see errors in the HA logs related to missing labels or cabinets, wait for Flynn to finish — it runs automatically and is self-correcting.

---

## Step 6 — Point Flynn at Your Conversation Agent

One input_text needs to be set for Flynn to bootstrap correctly:

**Settings → Helpers → ZenOS: Conversation Agent** (`input_text.zenos_conversation_agent`)

Set this to the entity ID of your HA conversation agent, for example:
```
conversation.claude
```

After setting this, trigger a restart or wait — Flynn re-runs on any health sensor change and will complete bootstrap on its next pass.

> **Note:** Flynn confirms the entity exists and is reachable but does not perform a live inference test at boot. If your model is misconfigured or offline it will pass this gate — the failure surfaces at runtime when the summarizer first calls it.

**Optional pre-seeds** (can also be set conversationally via OOBE):

| Helper | Entity | Purpose |
|---|---|---|
| ZenOS: Household Name | `input_text.zenos_household_name` | Your home's name |
| ZenOS: Primary User | `input_text.zenos_primary_user` | Your name |
| ZenOS: Persona Name | `input_text.zenos_persona_name` | Your AI's name |

If you leave these blank, OOBE will collect them conversationally. Only `zenos_conversation_agent` is required before first boot.

---

## Step 7 — Verify

Check these sensors in **Developer Tools → States**:

| Sensor | Expected state |
|---|---|
| `sensor.zen_label_health` | `ok` |
| `sensor.zen_cabinet_health` | `ok` |
| `sensor.zen_agent_health` | `ok` |

If any sensor shows `warn` or `error`, check its attributes for detail. Flynn will attempt self-repair on the next HA restart or health sensor change.

---

## Plugins (Optional)

Plugins live under `packages/zenos_ai/plugins/`. Install only the integrations you need — each is independent.

| Plugin | File | Requires |
|---|---|---|
| Mealie | `plugins/mealie/mealie.yaml` | Mealie instance + `input_text.mealie_url` |
| Grocy | `plugins/grocy/grocy.yaml` | Grocy instance + `input_text.grocy_url` |
| Calderas Spas | `plugins/calderaspas/` | Balboa-compatible spa integration |
| Kitchen Sync | `plugins/kitchen_sync/` | (see plugin readme) |

---

## Next Step

Once sensors are green and the conversation agent is configured, continue to the **[First Run Guide](first_run.md)** to complete OOBE and name your AI.
