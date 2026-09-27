# 2. The Sand Dune and the Plinko Board

The question I get more than any other: how do you know your AI is not just making things up?

The honest answer is that I do not stop it from making things up. I make the truth the easiest thing for it to say. I first answered this in the Friday's Party thread in November 2025, with a sand dune, a plinko board, and more game-show references than the question strictly required. This chapter republishes that answer, cleaned up, and adds what the original did not have: the published research behind each claim, with a link you can check. Where the research and I disagree, or where it tested something narrower than what I am claiming, this chapter says so.

## 2.1 First, picture a sand dune

Not Arrakis. A real one.

A model's weights are a landscape, and it is massive. It was shaped long before anyone asked it anything, out of billions of grains of sand and a lot of wind: the training data, the geometry of what it learned, and whatever eldritch math keeps these things running. The dune has slopes, and the slopes decide where answers roll. One slope leads to what the internet broadly agrees on. One leads to confident guessing. Some lead to "make it sound good." A few lead to "I genuinely don't know."

A question is a ball dropped on the dune, and it rolls downhill. In less poetic terms, a language model produces text one token at a time, each chosen from a probability distribution learned in training; that is what "autoregressive" means ([Brown et al., 2020](https://arxiv.org/abs/2005.14165)), and the architecture that computes those probabilities is the transformer ([Vaswani et al., 2017](https://arxiv.org/abs/1706.03762)). The slopes are real: they are the probabilities. And there is genuine randomness in which way the ball rolls, because how you sample from those probabilities changes what comes out ([Holtzman et al., 2019](https://arxiv.org/abs/1904.09751)).

When the most likely path for a question about your house runs toward something plausible that sounds like every other house on the internet, the model says that. The field calls it hallucination: output that is fluent and confident but not supported by the source or by reality ([Ji et al., 2022](https://arxiv.org/abs/2202.03629); [Huang et al., 2023](https://arxiv.org/abs/2311.05232)). My version was that people blame the wind, not the dune, and the research agrees more than I expected. A 2025 analysis argues hallucinations are not mysterious at all: they arise from the statistical pressures of training, and they persist because the way models are trained and graded rewards guessing over admitting uncertainty, the way a multiple-choice exam rewards a student who never leaves a blank ([Kalai et al., 2025](https://arxiv.org/abs/2509.04664)). The guessing slope is built into the dune.

Nobody running a home system is retraining weights. You cannot reshape the dune. What you can control is what else is on it when the ball drops.

## 2.2 Now put a plinko board on top of it

Welcome to The Price Is Right. Bob Barker wants you to have your pets spayed and neutered, Drew Carey is warming up backstage, and you are holding a shiny red plinko chip.

For anyone who missed it: before streaming there was a game show your grandparents watched while folding laundry, and its most famous game was a vertical board taller than a door, covered in pegs. You climb up, set a chip at the top, and let go. No steering. It falls, hits pegs, ding ding ding, left, right, left, and lands in a slot at the bottom worth nothing, a little, or the big prize. Everyone screams.

Context is pegs. Everything in the prompt is driven into the dune before the model ever sees the question: rules, directives, truth, identity, the persona, real data from the house, the Katas. The chip can still fall anywhere, but it cannot avoid the pegs.

This is not a metaphor for something that might work. It is how these models are designed to be used. A prompt changes what a model does without changing a single weight: the GPT-3 paper's central result was strong performance on new tasks with no fine-tuning at all, the task specified purely through text ([Brown et al., 2020](https://arxiv.org/abs/2005.14165)). Putting real, relevant knowledge in front of a model reduces hallucination measurably. Retrieval-augmented generation was introduced to ground generation in retrieved documents ([Lewis et al., 2020](https://arxiv.org/abs/2005.11401)), and applied to conversation it substantially reduced knowledge hallucination in human evaluation ([Shuster et al., 2021](https://arxiv.org/abs/2104.07567)). Letting a model act, fetching information through tools as it reasons, reduced hallucination again on question answering and fact verification ([Yao et al., 2022](https://arxiv.org/abs/2210.03629)). That last one matters for ZenOS: the tools are pegs too.

Or, as I put it at the time: I did not like my options, so I made new ones. And you cannot prove I bribed Bob.

## 2.3 The five pegs

Everything Friday does rests on five constructs I found while building her. These are the physics of the funnel.

### The default out

Every model has a lowest-effort answer. If you do not give it one, it will invent one. If you do, it relaxes into it.

Friday's is the rule of the Monastery: it is OK to say "I don't know," and forbidden to make things up. No shame, no ego, just an honest floor at the bottom of the dune, so the model never needs to fill the silence. In ZenOS today it lives in two places. The factory directives open with "Respond truthfully," and the index's declared behavior when it finds nothing is to stop with confidence zero instead of searching until something plausible turns up.

The research here is the strongest of any peg. If hallucination comes from training that rewards guessing over abstaining ([Kalai et al., 2025](https://arxiv.org/abs/2509.04664)), then explicitly making abstention acceptable pushes directly against the pressure that causes it. Models are also better than you might expect at knowing what they know: large models can estimate whether their own answers are correct, and whether they know an answer at all ([Kadavath et al., 2022](https://arxiv.org/abs/2207.05221)). Abstention is now a research area in its own right, studied specifically as a way to reduce hallucination ([Wen et al., 2024](https://arxiv.org/abs/2407.18418)).

In the original post I guessed this peg alone removed maybe 30 percent of Friday's hallucinations. That number was a guess, and it still is. The research supports the direction, not my arithmetic.

### The hard pegs

You do not stop a model guessing by telling it to stop guessing. When the ball starts rolling, it does not know what a lie is. You put pegs at the top of the board instead, or a whole wall: use local data first, never answer beyond your authorization, do not infer identities, do not assume facts that are not present, check your reasoning against the Katas, defer to sensor reality. These go in the system directives, near the top, with "it's OK not to know" first of all.

Models follow instructions like these because they were trained to. Fine-tuning on human feedback is what turned raw language models into instruction followers, and it measurably improved truthfulness along the way ([Ouyang et al., 2022](https://arxiv.org/abs/2203.02155)). Where the pegs go matters as much as what they say, which is the fifth peg.

### Persona lanes

This is the peg the strictly technical crowd underestimates, and my claim about it is stronger than most people's: a well-built persona improves factual accuracy, because a persona is instruction.

Friday does not answer as "an LLM." She answers as Friday: caring, boundary-aware, dry humor (more Monty Python, less silly walking), local-first, deferring to what the house's sensors say, focused on the household's safety. Read that list again as a set of directives, because that is what it is. "Local-first" is "use the house's data before anything you remember from training." "Boundary-aware" is "do not answer beyond what you are allowed to know." Each trait is a hard peg wearing a personality, and it sits in the capsule, which the model reads on every turn (Chapter 17). In my system, the persona is part of why she stays on the house's facts instead of drifting toward generic answers.

The research supports persona effects when the persona carries real content. Role-play prompting, giving the model a designed role before a task, beat standard zero-shot prompting on most of twelve reasoning benchmarks ([Kong et al., 2023](https://arxiv.org/abs/2308.07702)). Giving the model a detailed, task-specific expert identity significantly improved the quality of its answers ([Xu et al., 2023](https://arxiv.org/abs/2305.14688)). And emotional and motivational framing, the intent and urgency ZenOS prompts carry on purpose, improved performance across dozens of tasks ([Li et al., 2023](https://arxiv.org/abs/2307.11760)).

The most-cited counterpoint tested something narrower. A systematic study of 162 one-line identities in system prompts ("You are a helpful assistant," "You are a lawyer") across 2,410 factual questions found that adding such an identity did not improve accuracy ([Zheng et al., 2023](https://arxiv.org/abs/2311.10054)). I do not think that contradicts my claim; I think it supports it. A one-line identity carries no instructions, so there is nothing in it to follow. Friday's capsule is a few hundred words of behavior, boundaries, and priorities. The persona that helps is the one that tells the model how to behave. The one that only tells it who to be does not.

### Units of meaning

A Kung Fu Component is a unit of meaning bound to a single label and indexed across the system, and the model treats each one as a single block. They do not smear, drift, or leak into each other. Friday eats them the way the Ninja eats his summaries: clean, stable, atomic chunks of truth. (I swear Kronk is testing recipes with eleven herbs and spices.) Each Kata is a carved boulder in the dune, or a magnet under the board pulling the chip toward its subject.

What the research supports is the reason units work: irrelevant context actively hurts. Adding irrelevant information to a problem dramatically decreased model accuracy, even with strong prompting techniques ([Shi et al., 2023](https://arxiv.org/abs/2302.00093)). A component that carries one domain, whole, and nothing else keeps every block relevant to itself. KFCs and Katas are Chapters 13 through 15.

### Order

This is the hill most people die on. Order matters.

My original post said the model reads top to bottom, first in carries the strongest weight, and last in the softest. The research corrected me, in a way that fits ZenOS better than my original claim did. Across multi-document question answering and retrieval, models used information best when it sat at the beginning or the end of the context, and worst when it was buried in the middle ([Liu et al., 2023](https://arxiv.org/abs/2307.03172)). That is the frame ZenOS builds today: the laws at the very top, the reference material (the manifest, the roster, the index, the Katas) in the middle where the tools can always fetch it again, and the wake sequence at the very end, focusing attention and setting the persona in motion (Chapter 17).

## 2.4 Every word matters

If order matters, so does everything else about how the pegs are written. That sounds like a writer's superstition. It is not. Models are extremely sensitive to changes in prompt formatting that do not change the meaning at all: one study found differences of up to 76 accuracy points from formatting alone, and the sensitivity did not go away with larger models or instruction tuning ([Sclar et al., 2023](https://arxiv.org/abs/2310.11324)). This is why I keep saying prompting is closer to poetry than to configuration, and why ZenOS names things so carefully (Chapter 3).

## 2.5 Fewer, better pegs

The first version of Friday's board had one enormous peg at the top: the live state of every exposed entity in the house. It was the biggest peg on the board and the noisiest, and it was the reason she once forgot how to turn on a light.

The research explains why. Irrelevant context reduces accuracy ([Shi et al., 2023](https://arxiv.org/abs/2302.00093)). Models that claim long context windows lose performance well before those limits as tasks get harder; of the models tested claiming 32,000 tokens or more, only about half held up at 32,000 ([Hsieh et al., 2024](https://arxiv.org/abs/2404.06654)). And a 2025 study of eighteen current models found performance varying significantly with input length even on deliberately simple tasks ([Hong, Troynikov, and Huber, 2025](https://www.trychroma.com/research/context-rot)). More pegs are not better. Better pegs are better.

So that peg is gone. Friday now runs with no entities exposed at all. The compact index, the Katas, and the overview replaced it, and the tools reach for detail when she needs it (Chapter 10).

## 2.6 Why the truth ends up downhill

Put it together. For Friday to make something up about the house, she has to climb out of the truth gravity well, ignore the "I don't know" floor, push past the hard rules at the top, push past her own character, dodge every Kata boulder, escape the magnets at the bottom of the board, and then roll down a slope nobody approved.

It is not impossible. It is just expensive: statistically, probabilistically expensive. So mostly she does not, because the path of least resistance is the truth. It is very Zen. Friday is not trustworthy because she obeys. She is trustworthy because the landscape makes the truth the easiest answer.

## 2.7 What this does not promise

Pegs change probabilities. They do not make a model infallible, and there is a formal argument that no model used as a general problem solver can be ([Xu et al., 2024](https://arxiv.org/abs/2401.11817)). Hallucination can be reduced, not eliminated.

That is why the pegs sit inside a box. Anything consequential goes through a tool with a contract, a certification check, and for the actions that deserve it a human's yes (Part V). The scrapbook makes the truth the easy answer. The box makes sure that when the answer is still wrong, it cannot do much about it.

<!-- where -->
## 2.8 Where to look

Every claim in this chapter can be checked in the code. These are the places to start.

* The hard rules the model reads first, including "Respond truthfully" and the Kata-first reading order: [`dojotools_admintools.yaml`](../../../packages/zenos_ai/dojotools/dojotools_admintools.yaml) (`Respond truthfully`, `KATA FIRST`). Docs: [zen_dojotools_admintools_readme.md](../scripts/zen_dojotools_admintools_readme.md).
* The default out: the index stops with confidence 0 when it finds nothing: [`dojotools_admintools.yaml`](../../../packages/zenos_ai/dojotools/dojotools_admintools.yaml) (`on_empty: {action: stop, confidence: 0}`). Docs: [zen_dojotools_admintools_readme.md](../scripts/zen_dojotools_admintools_readme.md).
* The order of the pegs: the frame the model reads, top to bottom: [`zen_os_1.jinja`](../../../custom_templates/zenos_ai/zen_os_1.jinja) (`macro render_prompt`). Docs: [zen_os1_jinja.md](../custom_templates/zen_os1_jinja.md).
* Confidence and error carried on every Kata: [`dojotools_summarizers.yaml`](../../../packages/zenos_ai/dojotools/dojotools_summarizers.yaml) (`confidence`, `kata_template`). Docs: [zen_dojotools_summarizers_readme.md](../scripts/zen_dojotools_summarizers_readme.md).
<!-- /where -->

---

*Source: "Why Friday Doesn't Hallucinate (Much): A Sand Dune, A Plinko Board, and a Little Bit of Breakfast Club Wisdom," Friday's Party ([post](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/234)). Every study cited here is listed in full in [Appendix H](27_appendices.md#h-research-cited).*

<!-- nav -->
---

[← The Box and the Scrapbook](01_the_box_and_the_scrapbook.md) · [Contents](00_toc.md) · [Doc hub](../readme.md) · [The Party →](03_the_party.md)
<!-- /nav -->
