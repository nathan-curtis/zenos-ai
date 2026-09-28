# Research

The research behind [The Book of Friday](../architecture/00_preface.md): every study the book cites, grouped by the claim it supports, the community posts the book draws on, and ZenOS's own research notes. The canonical citation list is [Appendix H](../architecture/27_appendices.md#h-research-cited); this page is generated from it.

## Studies cited in the book

### Cognitive architecture

*Cited in Chapter 4, and the convergence in Chapters 3 and 26.*

* Sumers, T. R., et al. (2023). [Cognitive Architectures for Language Agents. Transactions on Machine Learning Research, 2024](https://arxiv.org/abs/2309.02427). The framework ZenOS converged on: working memory, three kinds of long-term memory, internal and external actions, a decision loop.

### How models generate text

*Cited in Chapter 2.1, the sand dune.*

* Vaswani, A., et al. (2017). [Attention Is All You Need](https://arxiv.org/abs/1706.03762). The transformer architecture that computes the probabilities the dune is made of.
* Brown, T. B., et al. (2020). [Language Models are Few-Shot Learners](https://arxiv.org/abs/2005.14165). Autoregressive generation, and a prompt changing behavior with no change to the weights.
* Holtzman, A., et al. (2019). [The Curious Case of Neural Text Degeneration](https://arxiv.org/abs/1904.09751). Sampling from those probabilities changes what comes out.

### Hallucination

*Cited in Chapter 2.1 and 2.7.*

* Ji, Z., et al. (2022). [Survey of Hallucination in Natural Language Generation](https://arxiv.org/abs/2202.03629). The definition: fluent, confident output not supported by source or reality.
* Huang, L., et al. (2023). [A Survey on Hallucination in Large Language Models: Principles, Taxonomy, Challenges, and Open Questions](https://arxiv.org/abs/2311.05232). The same, for large language models specifically.
* Kalai, A. T., et al. (2025). [Why Language Models Hallucinate](https://arxiv.org/abs/2509.04664). Hallucination comes from training and grading that reward guessing over abstaining.
* Xu, Z., et al. (2024). [Hallucination is Inevitable: An Innate Limitation of Large Language Models](https://arxiv.org/abs/2401.11817). A formal argument that it can be reduced, never eliminated.

### Grounding and tools

*Cited in Chapter 2.2, the plinko board.*

* Lewis, P., et al. (2020). [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401). Grounding generation in retrieved knowledge.
* Shuster, K., et al. (2021). [Retrieval Augmentation Reduces Hallucination in Conversation](https://arxiv.org/abs/2104.07567). Grounding measurably reduced knowledge hallucination in dialogue.
* Yao, S., et al. (2022). [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629). Acting through tools while reasoning reduced hallucination again.

### Abstention and self-knowledge

*Cited in Chapter 2.3, the default out.*

* Kadavath, S., et al. (2022). [Language Models (Mostly) Know What They Know](https://arxiv.org/abs/2207.05221). Models can estimate whether their answers are correct and whether they know an answer.
* Wen, B., et al. (2024). [Know Your Limits: A Survey of Abstention in Large Language Models](https://arxiv.org/abs/2407.18418). Abstention as a recognized way to reduce hallucination.

### Instructions, personas, and framing

*Cited in Chapter 2.3, the hard pegs and persona lanes.*

* Ouyang, L., et al. (2022). [Training language models to follow instructions with human feedback](https://arxiv.org/abs/2203.02155). Instruction tuning made models follow instructions and improved truthfulness.
* Kong, A., et al. (2023). [Better Zero-Shot Reasoning with Role-Play Prompting](https://arxiv.org/abs/2308.07702). A designed role beat standard zero-shot prompting on most of twelve benchmarks.
* Xu, B., et al. (2023). [ExpertPrompting: Instructing Large Language Models to be Distinguished Experts](https://arxiv.org/abs/2305.14688). Detailed expert identities improved answer quality.
* Li, C., et al. (2023). [Large Language Models Understand and Can be Enhanced by Emotional Stimuli](https://arxiv.org/abs/2307.11760). Emotional and motivational framing improved performance across dozens of tasks.
* Zheng, M., et al. (2023). [When "A Helpful Assistant" Is Not Really Helpful: Personas in System Prompts Do Not Improve Performances of Large Language Models](https://arxiv.org/abs/2311.10054). One-line identities did not improve factual accuracy; the book reads this as support for persona-as-instruction.

### How models use context

*Cited in Chapter 2.3 to 2.5: units, order, every word, fewer pegs.*

* Shi, F., et al. (2023). [Large Language Models Can Be Easily Distracted by Irrelevant Context](https://arxiv.org/abs/2302.00093). Irrelevant context sharply reduces accuracy.
* Liu, N. F., et al. (2023). [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172). Models use the beginning and end of the context best, the middle worst.
* Sclar, M., et al. (2023). [Quantifying Language Models' Sensitivity to Spurious Features in Prompt Design](https://arxiv.org/abs/2310.11324). Formatting alone moved accuracy by up to 76 points.
* Hsieh, C.-P., et al. (2024). [RULER: What's the Real Context Size of Your Long-Context Language Models?](https://arxiv.org/abs/2404.06654). Effective context is far shorter than claimed context.
* Hong, K., Troynikov, A., and Huber, J. (2025). [Context Rot: How Increasing Input Tokens Impacts LLM Performance. Chroma](https://www.trychroma.com/research/context-rot). Performance varies with input length even on simple tasks, across eighteen models.

## Community sources

The design history and the book's framing come from posts on the Home Assistant community forum. The anchor posts are indexed in [Appendix G](../architecture/27_appendices.md#g-fridays-party-index).

* [Friday's Party: Creating a Private, Agentic AI using Voice Assistant tools](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862). The design record, February 2025 onward. Chapter 3.
* [So You Want to AI in Home Assistant](https://community.home-assistant.io/t/so-you-want-to-ai-in-home-assistant/1021777), chapters 1 through 3. The box, context versus memory, the harness as the walls. Chapter 1.
* [A general problem with AI, post 6](https://community.home-assistant.io/t/a-general-problem-with-ai/1026421/6). Grandma's box and the scrapbook. Chapter 1.
* [Issues getting a local LLM to tell me temperatures, humidity, etc., post 3](https://community.home-assistant.io/t/issues-getting-a-local-llm-to-tell-me-temperatures-humidity-etc/801976/3). The original grandma thought experiment, November 2024. Chapter 1.
* [Why Friday Doesn't Hallucinate (Much)](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/234). The sand dune and the plinko board. Chapter 2.

## ZenOS research notes

* [Cognitive Architectures for Agentic AI in the Smart Home](whitepaper_cognitive_architectures.md). The November 2025 whitepaper where the CoALA mapping first appeared. Chapter 4.
