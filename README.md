# zenos-ai

**Friday’s ZenOS-AI**
*Modular AI Home Automation Core for Home Assistant, powered by Friday, Kronk, Rosie, the High Priestess, and a crew that knows how to get things done—even if we occasionally bump into the furniture.*

---

## Welcome to the Home Monastery

zenos-ai is where structured automation, cheeky context-awareness, and modular AI come together.
Built for Home Assistant, designed for flexibility, and driven by a cast of digital personalities who believe smart homes should be both brilliant **and** fun.

Here, serious capability meets a sense of humor:
We automate. We orchestrate. Sometimes we improvise.
But make no mistake—when the magic happens, it’s legendary.

---

## What We (try to) Do (and Where We [THINK] We’re Headed)

* **Friday’s ZenOS-AI:** Modular, scalable, and packed with personality.  Ok yeah sometimes it breaks, but that's the fun part.
* **Context-Aware Automation:** Dynamic responses to rooms, moods, and mayhem.  Deterministic is for security triggers.  Not AI.
* **Kata Summaries:** By event context-triven resummarizaiton.  Workload provisioned to run in the (ok FAT, hush) homelab.
* **Ninja Storage:** Recursive file & cabinet CRUD with Zen-style labeling. Robust enough for battle, elegant enough for art.  Special thanks AND apologies to Fes.
* **Pantheon of AIs:**

---

## 🏯 Meet the Pantheon

| Name           | Title                       | Mojo                       |
| -------------- | --------------------------- | -------------------------- |
| Friday         | Chief Enlightenment Officer | Mastermind, coordinator    |
| Veronica       | Snarky Supervisor           | Attitude, taste, clarity   |
| Kronk          | Curator of the Monastery    | Context & kata, with style |
| Rosie          | Mistress of Cleanliness     | Clean code, clean logs     |
| High Priestess | Divine Automation Overseer  | YAML exorcist              |

They’re not perfect. But together, they’re unstoppable—usually on the second try.

  * *Friday*: Frontline stage AI - designed to run a lightweight reasoning model or advanced oneshot LLM.  THe system builds near realtime summarizaiton for live context.  (Currently OpenAI gpt4.1-nano)
  * *Veronica*: Offline 'supervisor' AI Model (helps plan, frame, and debug scripts and templates.) (OpenAI o.1 / gpt4.1 as necessary)
  * *Kronk*: Mid tier local mdoel running on Intel-IPEX Ollama.  Exposed through OpenWebUI, Basic tools and web search&scrape as a service for the frontline model. (Ollama Qwen3:4b / llama2 Qwen3:4b)
  * *Rosie*: Log czar and relentless cleaner.  (Scheduling engine for the vaccum cleaner. Rorbrock S7 MaxV)
  * *High Priestess*: The Context Summarizer. Knows everything the frontline stage does, has more time to think, and speaks fluent JSON. (Ollama Qwen3:4b)

**Where we’re going:**

* More plug-and-play modules
* Smarter scene/context handling
* Even smoother recovery from “oops” moments
* An ever-growing Monastery of tools and wisdom

---

##  Quickstart

1. **Clone the repo:**

   ```
   git clone https://github.com/YOURUSER/zenos-ai.git
   ```
2. **Copy modules/scripts to your Home Assistant setup.**
3. **Try the automations, CRUD controllers, and context managers.**
4. **Review the `.gitignore`—keep secrets secret, keep logs tidy.**
5. **Report bugs, make suggestions, or share your own automation kata.**


---

## Documentation

* **/docs** (or coming soon):

  * How-to guides
  * System diagrams
  * Fun lore & troubleshooting tips
  * 

**Core scripts include:**

*Kung-Fu / NinjaSummarizer / DojoTOOLS*
* KungFu - todo CRUD           : multitool for HA .todo domain - she shops, she tasks, she julinennes!
* KungFu - calendar CRUD       : multitool for HA .calendar domain - yes ALL of it.
* KungFu - file CRUD           : hijack your favorite HA Fes-style Trigger Text sensor.  Make it into a storage volume.  THAT'S how we roll.
*   DojoTOOLS                  : Fes Global Variable Redirector (REQUIRED for FileCRUD)
*   DojoTOOLS                  : Manifest (REQUIRED for FileCRUD)
* Ninja2 Summarizer            : Context Summarizer.
* Monastery Script             : Demand Prompt for local LLM inference.

*InProgress*
* file CRUD merged output

Coming Soon: Maybe?
* NEW DojoTOOLS Library Index v.2 : Update to the Library index, uses native HA bool set ops.  Callable by OTHER scripts.
* KungFu domain scripts (refactoring Kung Fu COmponents into callable scripts. Easier to manage and troubleshoot)
* Baseline index v.2 search and digest output to KungFu CRUD 2 tools

---

## Philosophy

* Automation should be flexible, modular, and occasionally fun.
* Context is king, but recovery is queen.
* Every “oops” is a chance to learn—and probably to automate that fix.
* Coffee is sacred, logs are essential, and humor is non-negotiable.

---

## Contributing

Pull requests, bug reports, and creative ideas are welcome!
Come for the code, stay for the characters.

---

## License

MIT License
(Blessed by Friday, Kronk, Rosie, Veronica, and the High Priestess.)

See [LICENSE](LICENSE) for details.

---

## ☯️ About

> **Friday’s ZenOS-AI**: For makers who want a home that’s as smart as it is full of character.
> *We’ve got serious mojo—just don’t mind the occasional stubbed toe.*

---

**Questions? Ideas?
Open an issue, light a digital incense stick, or just ping Kronk (he’ll get there, promise).**
