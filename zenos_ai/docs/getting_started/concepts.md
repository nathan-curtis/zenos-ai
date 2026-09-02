# Concepts — The Words You'll Keep Seeing

> **Version:** 2026.9.0 'Steel Magnolia'

*A plain-language reference for the recurring cast of names and terms in these docs. Read it straight through once, or jump back to it whenever a doc says "see Concepts."*

---

## Flynn

Flynn is the system's own startup checker — not an assistant you talk to, and not your AI's name. Every time Home Assistant restarts, Flynn runs first, checks that everything ZenOS-AI needs is in place, and either gives the all-clear or tells you plainly what's wrong. You'll see Flynn's name in restart notifications and in troubleshooting steps. If your system is healthy, you'll rarely think about Flynn at all.

## Your AI persona ("Friday")

**"Friday" is an example name, not a fixed one.** ZenOS-AI ships with a default persona named Friday, but during first-time setup you name your own AI whatever you like — these docs keep saying "Friday" purely as a placeholder for "whatever you named yours."

## Cabinets and Drawers

Think of a **Cabinet** as a filing cabinet the system keeps for a specific purpose (one for your household, one for your AI's own identity, one for family members, and so on), and a **Drawer** as one labeled folder inside it holding a specific piece of information. When a doc says "stored in a drawer," it means: saved in one of these labeled slots, not scattered across random files.

## Kata

A **Kata** is a short, structured summary the system writes about something that happened — a compressed note-to-self, not a raw log. Your AI reads Katas instead of re-reading everything that ever happened, the same way you'd read a meeting summary instead of a full transcript.

## DojoTools vs. AdminTools

**DojoTools** are the everyday actions your AI is allowed to use — turning on lights, checking a room's status, firing an alert. These are the tools you expose to your conversation agent.
**AdminTools** are repair and recovery actions (fixing a broken cabinet, rebuilding an index) meant for a human operator, not for your AI to reach for during normal conversation. Keep these un-exposed unless you're deliberately doing repair work.

## AlertManager and Postman

**AlertManager** decides whether something happening in your house is worth a person's attention, and prevents the system from repeating itself about the same ongoing thing.
**Postman** is what actually delivers that attention to a person — a notification, routed by who should be asked and when (respecting quiet hours, for instance) — and waits for an answer: acknowledge, escalate, or dismiss.

## Room Manager

**Room Manager** is the system's map of your house — which rooms exist and how they connect. OOBE builds this map for you conversationally.

## Room Manager v3 and REFLEX

The map above only tells the system *where* things are — it doesn't say what's happening in them. **Room Manager v3** is a separate, always-running layer that tracks each room's actual live status (Vacant, Occupied, Engaged, Asleep, or Hold) from real signals like motion, doors, and media playing, updated continuously rather than on a fixed schedule. **REFLEX** watches those status changes and reacts to them automatically — firing the right lighting scene, for example — without your AI needing to actively decide anything each time. See the [Room Manager v3 & REFLEX manual](room_manager_operators_manual.md) for the full picture; OOBE sets this up but doesn't walk you through it conversationally.

## CabCeption

An internal term for cabinets that can contain other cabinets, nested inside each other (a "drawer" that is itself a whole filing cabinet). You'll only encounter this word in a specific startup message about history cabinets — see [First Run's troubleshooting section](first_run.md#if-something-goes-wrong) for what to do if you see it.

---

→ Back to the **[Getting Started hub](readme.md)**
