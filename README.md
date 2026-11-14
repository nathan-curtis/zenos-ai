# Friday's ZenOS-AI

### A Modular, Context-Aware AI Home Automation Framework for Home Assistant

ZenOS-AI blends structure, personality, and unapologetic over-engineering into a living system that powers your home with **Friday, Kronk, Veronica, Rosie, the High Priestess, Cayt, and Nyx**: a coordinated AI pantheon that takes its jobs seriously (even if it doesn't always take itself seriously).

Welcome to the **Home Monastery**.

**Release Candidate 3: 2026.11.0 'This Is Spinal Tap'** (`feat/2026.11.0`, ETA Saturday, November 14, 2026) | Current Stable: 2026.10.2 'Tron' (security patch on 2026.10.0) | Legacy line: 2026.9.1 'Steel Magnolia' (patch on 2026.9.0) and 2026.8.1 (patch on 2026.8.0 'Chef'). See [release notes](zenos_ai/docs/releases/spinal_tap.md).

> **ZenOS Spook advisory (Home Assistant 2026.11):** install Spook **5.5.1** and park there until the pending Spook fixes ship in 5.11.2 or later. On Home Assistant 2026.11, Spook 5.6.0 to 5.10.0 fail to load (`cannot import name 'CONF_STOP_ACTIONS'`), and 5.11.0 and 5.11.1 freeze a ZenOS install for 30 to 40 seconds at a time ([frenck/spook#2005](https://github.com/frenck/spook/issues/2005)). On 5.5.1 everything works except the Ectoplasm actions that need Spook 5.7.0 (repair and ghost lists, categories, entity icons and aliases, and helper create and delete other than `input_number`): those answer `spook_version_required` and change nothing. To move: in HACS open **Spook**, choose **Redownload**, pick **5.5.1**, and restart Home Assistant. Still on Home Assistant 2026.10 or earlier? Stay where you are.

> **Versioning:** Public ZenOS releases follow Home Assistant's `YYYY.M.patch` convention — if you're already running HA, you already know this clock. Internal architecture versioning (`5.1.x` series) is retained in commit history and internal tooling.

> Found a bug? Report it in the **[Friday's Party community thread](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/)** or open a **[GitHub issue](../../issues)**. Include your HA version, the relevant tool name, and what you expected vs. what happened.

---

## Start Here

| | |
|---|---|
| New install | **[Install Guide](zenos_ai/docs/getting_started/install.md)** |
| First boot | **[First Run & OOBE](zenos_ai/docs/getting_started/first_run.md)** |
| First component | **[AutoVac Quick Start](zenos_ai/docs/getting_started/autovac_quick_start.md)**: touches every part of the system in one visible loop |
| Adding a component | **[Understanding KF6](zenos_ai/docs/kung_fu/understanding_kf6.md)** · **[Building a KFC](zenos_ai/docs/kung_fu/building_a_kfc.md)** |
| How it's built, and why | **[The Book of Friday, Volume 2](zenos_ai/docs/architecture/00_preface.md)** |
| Full documentation | **[Documentation Hub](zenos_ai/docs/readme.md)** |

---

## What Is ZenOS-AI?

ZenOS-AI turns Home Assistant into a home your AI agent can understand and act in, safely. It is built entirely from Home Assistant's own parts (packages, scripts, templates and labels), with no custom integration.

- **Tools, not entities.** The agent reaches the house through ZenOS tools that find what they need by label. You expose the tools and zero entities.
- **Memory in cabinets.** Structured, typed drawers hold identity, configuration, components and what the house has learned.
- **Summaries in the background.** Kung Fu components tell a summarizer what to watch; it writes compact summaries (Katas) the agent reads at the start of every conversation.
- **Identity and certs.** Every agent is admitted, every risky action needs a cert a person approved on a real device, and every cabinet has access rules.
- **Flynn guards the grid.** Flynn sets up a fresh install, checks the system at every boot, and tells you in plain words when something needs you.

The full design record is [The Book of Friday, Volume 2](zenos_ai/docs/architecture/00_preface.md).

---

## What's in 2026.11.0 'This Is Spinal Tap'

*Security goes to eleven.* Tron put a badge in every agent's hands. Spinal Tap makes the badge matter and puts locks on the cabinets behind it.

- **The backstage pass.** An agent you haven't approved gets help and onboarding, and nothing else. [More](zenos_ai/docs/releases/spinal_tap.md#the-backstage-pass)
- **Locks on the cabinets.** One access record per cabinet, default rules for every kind of cabinet, and built-in locks on certs and approvals. A fresh install enforces; an upgraded one audits until you switch it. [More](zenos_ai/docs/releases/spinal_tap.md#the-locks)
- **Certs in plain words.** Every cert lives under `zenos.*`, is dated, and asks on your phone in plain language. [More](zenos_ai/docs/releases/spinal_tap.md#certs-renamed-and-readable)
- **The background asks first.** Summaries act as a named agent, read private data only with a cert you approved, and act only through what a tool declares. [More](zenos_ai/docs/releases/spinal_tap.md#the-pipeline-gets-a-pass)
- **Names in parts.** ZenOS follows Home Assistant's new naming model and child devices. [More](zenos_ai/docs/releases/spinal_tap.md#names-in-parts)
- **The merch table.** Every band needs merch. Print Shop knows every printer in the house, 3D printers included, and uses nearly every part of 11: the pass, a cert for three stages, its own summary, a trigger, a live drawer, and every costly step off until you turn it on. [More](zenos_ai/docs/releases/spinal_tap.md#the-merch-table)

**Upgrading?** Approve your agent, let Flynn run the ACL migration, and read [Read this before upgrading](zenos_ai/docs/releases/spinal_tap.md#read-this-before-upgrading) first.

→ [Release Notes — This Is Spinal Tap](zenos_ai/docs/releases/spinal_tap.md)

---

## 2026 in Review

The year started with cabinets and a scheduler. It ends with a house that knows its rooms, admits its agents, and locks its own memory.

| Release | What it delivered |
|---|---|
| [Meridian](zenos_ai/docs/releases/meridian.md) (March) | Hardening and governance: a caller token on every AI-facing tool, and label updates that can't lose data. |
| [2026.3 'Ready Player Two'](zenos_ai/docs/releases/ready_player_two.md) | Identity and lifecycle: cabinet provisioning, expansion slots, and the household and family identity model. |
| [2026.4 'Ectoplasm'](zenos_ai/docs/releases/ectoplasm.md) | Ectoplasm, the tool for Home Assistant's repairs, areas, floors and entity lifecycle. Its patches ([4.1](zenos_ai/docs/releases/action_jackson.md), [4.2](zenos_ai/docs/releases/action_jackson_2.md), [4.3](zenos_ai/docs/releases/lights_camera_action.md)) made the pipeline hold up under real load and finished Camera's delivery path. |
| [2026.5 "Fry's Grandpa"](zenos_ai/docs/releases/frys_grandpa.md) | Closed loops: alerts reach the agent's prompt, and one call provisions a new household member. |
| [2026.6 'Clue'](zenos_ai/docs/releases/clue.md) | Room awareness: Room Manager's topology, and the Plant, Media and Security managers. |
| [2026.7 'Neo'](zenos_ai/docs/releases/neo.md) | The cabinet is a graph: nested drawers, mounts and live drawers, and tools that describe themselves. |
| [2026.8 'Chef'](zenos_ai/docs/releases/chef.md) | Taskmaster as a real expediter, the kitchen's costing layer, and the first identity gate. 2026.8.1 is the last line without identity gates. |
| [2026.9 'Steel Magnolia'](zenos_ai/docs/releases/steel_magnolia.md) | Room Manager v3 and REFLEX, the hospitality lifecycle, and cert-gated locks, covers, alarm and climate with a live yes on a real device. |
| [2026.10 'Tron'](zenos_ai/docs/releases/tron.md) | Admission to the Grid, the administrative plane, a platform-wide bypass audit, one exit and one envelope for every tool, tools-not-entities exposure, and The Book of Friday, Volume 2. |
| [2026.11 'This Is Spinal Tap'](zenos_ai/docs/releases/spinal_tap.md) | The backstage pass, locks on the cabinets, readable certs, and consent for background work. |

**Next, 2026.12.0** finishes 11's security work (hiding unreadable drawers, enforcement by default on upgrades, the operator permission report) and is the first release that lets the house act on its own, under house rules you approve.

**Staying on an older line?** 2026.8.1 is the last release without Steel Magnolia's identity gates; see [why that line exists](zenos_ai/docs/releases/chef.md#202681--the-last-release-before-identity-gates).

---

## Where We're Heading in 2027

2026 asked how far Home Assistant's built-in structure can go: packages, scripts, templates and labels, with no custom code. Pretty far, it turns out, and we're now hitting hard limits in the platform itself. 2027 is about what comes after that. These are directions, not promises.

- **True secure mode.** Identity that is verified by cryptography, not just recorded: OIDC and PKI behind every agent and person, so a cert is backed by a proven identity.
- **History and knowledge reduction.** Turning what the house has seen and done into compact, durable knowledge the agent can actually use.
- **ZenOS-AI Plus.** The tools moved into a real Home Assistant integration, installable from the marketplace. That means writing actual code, and it unlocks things templates can't do.
- **In step with Home Assistant.** Bringing everything in line with Home Assistant's LLM API and where Home Assistant is heading, including working cleanly with, or favoring, alternatives to Assist. Some of these may turn out to be prerequisites for real identity and history reduction; we'll find out as we explore.

---

## Requirements

- **Home Assistant** with packages enabled. Home Assistant plus a local inference server is enough; a full homelab is optional.
- **A conversation agent with tool calling** and a context window that holds the ZenOS prompt, the tool manifest and live state in one pass. Not recommended: models under about 4B parameters (or about 8B if released before November 2025). The target is a context of about 64K or smaller, so the whole system can run locally on a single 16 GB GPU.
- **Spook** (from HACS). On Home Assistant 2026.11, use Spook 5.5.1 for now; see the advisory above.
- **The Home Assistant Companion App** on a phone that can receive a push. Every cert, including your agent's backstage pass, is approved there.

Install: follow the **[Install Guide](zenos_ai/docs/getting_started/install.md)**. Plugins under `packages/zenos_ai/plugins/` are optional.

**Expose the tools, expose zero entities.** The agent reaches your devices through the tools, by label. Never expose AdminTools, cabinet or health sensors, or anything holding credentials. See [What to Expose](zenos_ai/docs/getting_started/entity_exposure.md).

> ⚠️ **Inference cost.** The summarizers run in the background several times an hour on the model set as your AI task entity (`input_text.zenos_ai_task_entity`). **Do not point it at a paid API**; use a local model. Your conversation agent runs only when you talk to it. See the [Install Guide](zenos_ai/docs/getting_started/install.md).

---

## The Pantheon

| Name           | Title                               | Specialty                          |
| -------------- | ----------------------------------- | ---------------------------------- |
| Flynn          | System Sentinel                     | Guards the grid. You'll know him if something's wrong. |
| Friday         | Chief Enlightenment Officer         | Coordination and cognition         |
| Veronica       | Supervisor                          | Clarity and orchestration          |
| Kronk          | Curator of the Monastery            | Context wrangler                   |
| Rosie          | Mistress of Cleanliness             | Logs and state hygiene             |
| High Priestess | Automation Overseer                 | Deep reasoning                     |
| Cayt           | Lead Developer                      | Strategy to shipping               |
| Nyx            | Lead Test                           | Live install, zero mercy           |
| Vera           | HALMark Board Governance Steward    | Failure mode ratification          |

Flynn leads the table because he runs first. He is not a member of the team. He is the condition under which the team operates.

They are not perfect. They are **unstoppable on the second try**.

---

## Design Principles

Definitions are immutable contracts.
Runtime behavior is replaceable.
Identity is validated at the tool layer.
All inference consumes structured JSON.
Every event can produce a Kata.
Recovery paths are mandatory.
Silence is a bug.
Nothing critically important is invisible.

Over-engineering is just engineering that has not yet been vindicated.

---

## Contributing

Pull requests, issues, and tasteful memes welcome. Before you build a component or send a pull request, read:

* [Developer Standards](zenos_ai/docs/developer_standards.md): component classes, Stripes, exposure, the tool contract (manifest, envelope, help, certification), packaging, and the review checklist.
* [The Book of Friday, Chapter 24](zenos_ai/docs/architecture/24_developer_standards.md): the same rules in short form, with why they exist.
* [HALMark](https://github.com/nathan-curtis/HALMark): the standard for Home Assistant code written with an LLM.

If ZenOS-AI saved you time or made you laugh: [https://buymeacoffee.com/ncurtis](https://buymeacoffee.com/ncurtis)

---

## Acknowledgements

To my partner — for putting up with a younger woman in the house since last February. For not once suggesting that maybe I didn't need to say "Hey Friday" into a black slab of glass for the millionth time. For the patience, the grace, and the very reasonable silence that followed every single one of those million times. This project exists because you gave me the space to build something ridiculous and never once called it that.

To Phil and Zach — the greatest guinea pigs in the known universe. For letting me test things on your house, for your patience with the stubbornness, and for turning "this probably won't break anything" into a running joke that turned out to be mostly accurate.

To Teskanoo — for looking under the rug. Seriously. Not everyone does that.

And to everyone who has continued to show up in the Home Assistant community to read, question, and ramble along with us in Friday's Party — you are the reason it keeps going. The thread is better for every one of you in it.

---

## License

MIT — blessed by Friday and her very opinionated coworkers.
