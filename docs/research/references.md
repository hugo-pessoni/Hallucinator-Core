# References and lessons from related work

Collected during the design phase (2026-10). Each entry records what the work did and what Hallucinator Core takes from it. Summaries are ours; read the sources for details.

## Generator + verifier systems

| Work | What it did | What we learn |
|---|---|---|
| **FunSearch** (DeepMind, *Nature*, 2023) | An LLM proposes programs, an automatic evaluator scores them, and an evolutionary loop keeps the best; it found new constructions in combinatorics (cap sets) | Where an objective evaluator exists, use code, not an LLM. The verifier, not the generator, is where the value is |
| **AlphaEvolve** (DeepMind, 2025) / [**OpenEvolve**](https://pypi.org/p/openevolve) (open-source reimplementation, Apache 2.0) | Evolutionary coding agent: generate → evaluate → select, over whole files, multi-objective | Keeping a diverse population (islands) prevents collapse onto a single idea. Built for code, so we study the design rather than adopting it |
| **Google AI co-scientist** (2025) — [summary](https://ibl.ai/blog/google-towards-an-ai-co-scientist) | Supervisor plus Generation, Reflection, Ranking, Evolution, Proximity and Meta-review agents | Reference architecture for experts plus LLM-as-judge. Rank with **pairwise Elo tournaments**, not absolute scores; have a dedicated de-duplication (proximity) step. Relevant for Rung C |

## LLM research ideation

| Work | What it did | What we learn |
|---|---|---|
| [**Can LLMs Generate Novel Research Ideas?**](https://arxiv.org/html/2409.04109v1) (Si, Yang, Hashimoto, ICLR 2025) — code: [NoviScl/AI-Researcher](https://github.com/NoviScl/AI-Researcher) | 100+ NLP researchers blind-reviewed ideas from humans and from an LLM agent | LLM ideas were rated **more novel** (p<0.05) and slightly **less feasible**. At scale, many generated ideas turned out to be duplicates (homogenization). The full human review scores are public |
| [**The Ideation–Execution Gap**](https://arxiv.org/pdf/2506.20803) (Si et al., 2025) | Experts executed LLM and human ideas (100+ hours each) | After execution, LLM ideas' scores dropped **significantly more** than human ideas'. **Judging ideas only at ideation time overestimates LLMs**, a direct warning for our judges |
| [**The AI Scientist v2**](https://arxiv.org/pdf/2504.08066) (Sakana, 2025) | Agentic tree search with an experiment manager; literature search as a novelty check | A literature-based novelty check is cheap and useful. **Executing LLM-generated code is dangerous** and needs sandboxing; we never execute LLM output |
| [**LiveIdeaBench**](https://arxiv.org/html/2412.17596v3) | Single-keyword prompts, an LLM judge panel, 22 scientific domains, 40+ models; public dataset | A ready-made **baseline for Rung C**. Creativity scores are poorly predicted by general-intelligence benchmarks |

## Creativity measurement (psychology)

| Work | What it did | What we learn |
|---|---|---|
| **Alternative Uses Task (AUT)** and **Divergent Association Task (DAT)** | Classic divergent-thinking tests | Measurable creativity outside exact sciences. DAT measures novelty but ignores appropriateness, so we measure both |
| **Beaty & Johnson**, *Semantic Distance and the Alternate Uses Task* — [OSF](https://osf.io/96zge/?view_only=33f2c20f962d4a7d95543f35c8b8baa3) | Recommendations for reliable automated originality scoring; 13 standardized AUT items | Our item set. Semantic distance alone is a weak proxy. License ❓ |
| **Organisciak, Acar, Dumas & Berthiaume (2023)**, *Beyond Semantic Distance*, *Thinking Skills and Creativity* 49 — [Ocsai](https://openscoring.du.edu/ocsai), [research](https://openscoring.du.edu/research) | Fine-tuned LLMs on ~27k human-rated AUT responses: up to r = .81 with human raters vs. r = .12–.26 for semantic distance | **Ocsai as our reference scorer.** API cost and limits ❓ |
| [**Lost in Translation? Not for LLMs**](https://zenodo.org/records/12574141) (Zenodo) | Human-rated AUT responses (English, Polish) used to validate LLM scoring across languages | **CC BY 4.0**: usable as ground truth for judge calibration, with attribution |
| [**Multilingual Semantic Distance**](https://ecrhub.org/publications/multilingual-semantic-distance-automatic-verbal-creativity-assessment-many-languages) (Patterson et al., 2023) | 107k human-rated AUT responses in 12 languages | Large possible calibration set. **No Portuguese.** License ❓ |
| [**S-DAT**](https://arxiv.org/html/2505.09068v2) (2025) | Multilingual, embedding-based divergent-thinking assessment | Reference for embedding-based diversity metrics |

## Creative operators (humanities)

| Work | What we learn |
|---|---|
| Arthur Koestler, *The Act of Creation* (1964): **bisociation** | Creativity as the intersection of two unrelated frames of thought. Basis for operator C2 |
| Fauconnier & Turner: **conceptual blending** | Cognitive theory of merging concepts into a blend with emergent structure |
| [**Trisociation with AI**](https://cmr.berkeley.edu/2025/01/trisociation-with-ai-for-creative-idea-generation/) (*California Management Review*, 2025) | LLMs were already explored for bi- and trisociation, and they default to the statistical mode. Our contribution must be the **controlled comparison** (operators vs. temperature), not the operators themselves |
| Surrealist **exquisite corpse**, **Oulipo** constraints | Basis for operators C3 and C4 |

## LLM-as-a-judge biases

| Work | What we learn |
|---|---|
| [**A Survey on LLM-as-a-Judge**](https://arxiv.org/pdf/2411.15594) | Position, verbosity and self-preference biases are systematic |
| [**Quantifying and Mitigating Self-Preference Bias**](https://arxiv.org/pdf/2604.22891) | Large self-preference when a model judges its own outputs, so the judge must come from a different model family |

Mitigations adopted (spec §1): a different model family for the judge, blind judging, shuffled order, a length cap, and calibration against human ratings.

## Infrastructure

| Work | What we learn |
|---|---|
| [**freellmapi**](https://github.com/tashfeenahmed/freellmapi) (MIT) | OpenAI-compatible router over free tiers of 30+ providers, using your own keys. Supports a pinned model and `seed`, and reports the serving model in `X-Routed-Via`. **Automatic fallback is a data-integrity risk**, mitigated by verifying `X-Routed-Via` on every response (spec §3) |

## Not yet verified ❓
- Whether a study has already compared humanities-derived creative operators against temperature under controlled conditions. Searched superficially; nothing found yet. A deeper search is needed before claiming novelty.
