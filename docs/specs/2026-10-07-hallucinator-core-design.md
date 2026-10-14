# Hallucinator Core — Design Spec (Rung B: engine + experiment)

- **Date:** 2026-10-07
- **Author:** Hugo Pessoni
- **Status:** Draft — sections 1–2 approved in design review; sections 3–6 proposed, awaiting review
- **Related:** [`docs/research/references.md`](../research/references.md)

## Status markers used in this document

| Marker | Meaning |
|---|---|
| 🟢 **Confirmed** | Decided by the author during design review |
| 🟡 **Suggestion** | Proposed by the assistant, not yet explicitly approved |
| ❓ **Pending** | Open question, to be resolved before or during the implementation plan |

Nothing marked 🟡 or ❓ should be treated as a project decision.

---

## 0. Context and goals

**Origin.** Hallucinator Core started as the idea that LLM "hallucinations" might contain non-obvious ideas worth examining instead of discarding — the way surrealism or quantum physics looked absurd before being understood.

**What the literature already covers** (see references): generator + verifier loops (FunSearch, AlphaEvolve/OpenEvolve), multi-agent hypothesis generation with tournament ranking (Google AI co-scientist), LLM research ideation at scale (Si et al.), and the finding that LLM ideas look more novel at ideation time but lose more value once executed (Ideation–Execution Gap). Raising temperature correlates only weakly with novelty and strongly with incoherence.

**What this project contributes.** A controlled comparison: do *structured* creative operators taken from the humanities (bisociation, exquisite corpse, Oulipo constraints, analogy) outperform *random* noise (high temperature) — measured against human-calibrated judges with controlled biases?

**Goals** 🟢
1. A well-run experiment with an honest, measured answer — positive or negative. This is the success criterion.
2. Learning: the author wants to understand every component.
3. Documentation produced alongside the code is a first-class deliverable (technical, decisions, tests, ideas, references).
4. Discovering genuinely new ideas is a *possible outcome*, not a success criterion.

**Scope ladder** 🟢
- **Rung B (this spec):** Alternative Uses Task (AUT), with public human-rated ground truth.
- **Rung C (future spec):** open ideas per knowledge area (science, art, philosophy), reusing the judges calibrated in Rung B, with LiveIdeaBench as a baseline.
- **Platform (future spec):** a public web experience ("Try it =)"). Explicitly out of scope here.

**Constraints** 🟢
- Solo project, no deadline.
- Zero cost for now: local models (Ollama) and free tiers via `freellmapi`. Paid APIs later, by swapping provider profiles only.
- All repository content in English.
- Author's machine: RTX 5060 (8 GB VRAM), 32 GB RAM, Ryzen 5 5500. One ~8B quantized model fits in VRAM at a time.

---

## 1. Experiment design (Rung B) — 🟢 approved

### Research question
Do humanities-derived creative operators produce **more original** alternative uses than high-temperature sampling, **without losing appropriateness**?

Originality alone rewards nonsense; the standard definition of creativity in psychology is *original **and** appropriate*. Both are measured.

### Preregistration
Hypotheses, metrics, thresholds and the analysis plan are written and committed to `docs/preregistration.md` **before** any experimental run. Git history proves the plan predates the data and prevents choosing the analysis after seeing results (p-hacking).

### Task
AUT using the 13 standardized items selected by Beaty & Johnson: belt, brick, broom, bucket, candle, clock, comb, knife, lamp, pencil, pillow, purse, sock.

### Conditions
Same generator model in every condition.

| ID | Condition | Temperature | Prompting |
|---|---|---|---|
| C0 | Baseline | 0.7 | Direct prompt: "unusual uses for X" |
| C1 | High temperature | 1.3–1.5 (❓ exact value) | Same prompt as C0 |
| C2 | Bisociation | 0.7 | X crossed with a randomly sampled distant domain |
| C3 | Exquisite corpse | 0.7 | Chain of agents, each continuing without seeing the whole |
| C4 | Oulipo constraint | 0.7 | A forced constraint (e.g. "the use may not involve weight or shape") |
| C5 | Analogy | 0.7 | "How would a [octopus / archivist / glacier] use X?" |

Operators run at the **baseline temperature** so any effect is attributable to the operator, not to noise (variable isolation).

### Metrics
- **Originality:** Ocsai as the human-calibrated reference scorer, plus our LLM judge.
- **Appropriateness:** LLM judge — "would this actually work?"
- **Diversity:** mean pairwise semantic distance within a condition (tests the homogenization problem).
- **Validity rate:** share of coherent responses.

### Judge calibration (runs first)
The LLM judge scores a public human-rated AUT dataset (CC BY 4.0). We publish its correlation with human ratings and with Ocsai. ❓ The minimum acceptable correlation (e.g. r ≥ 0.5) is fixed in the preregistration. If the judge fails, this is documented and Ocsai becomes the primary originality metric.

### Bias controls
- Judge from a **different model family** than the generator (avoids self-preference bias).
- **Blind** judging: the judge never sees the condition.
- Randomized order.
- Response length capped (neutralizes verbosity bias).

### Statistics
Bootstrap confidence intervals and effect sizes, not p-values alone. The report answers "how large is the difference and how certain are we".

### Pending ❓
- Responses per item × condition (≈30 → ≈2,300 generations), to be computed in the plan.
- Whether the Ocsai API is free and its rate limits.
- Exact high-temperature value for C1.
- Licenses of every dataset before anything is committed (see section 4).

---

## 2. Engine architecture — 🟢 approved

**Orchestration:** LangGraph 🟢 (chosen by the author over a plain-Python pipeline; rationale and consequences to be recorded in an ADR).

```
experiments/b1.yaml  (items, conditions, N, roles → provider profiles)
        │
        ▼
┌─ Graph 1: GENERATE ──────────────────────────────┐
│ planner → fan-out (item × condition × n) →       │
│ operator nodes → generator LLM → generations.jsonl│
└──────────────────────────────────────────────────┘
        ▼
┌─ Graph 2: CURATE ────────────────────────────────┐
│ normalize → length cap → dedup (embeddings) →    │
│ validity filter → curated.jsonl                  │
└──────────────────────────────────────────────────┘
        ▼
┌─ Graph 3: JUDGE ─────────────────────────────────┐
│ blind + shuffle → judge panel in parallel:       │
│   • LLM judge (originality + appropriateness)    │
│   • Ocsai (reference scorer)                     │
│   • semantic distance                            │
│ → aggregate → scores.jsonl                       │
└──────────────────────────────────────────────────┘
        ▼
  analysis (plain Python, no graph) → report.md + plots
```

### Components

| Module | Responsibility |
|---|---|
| `providers/` | Provider profiles; client factory (`ChatOpenAI` pointed at each profile's `base_url`); `X-Routed-Via` verification; retry with exponential backoff on rate limits; response cache keyed by hash(model, prompt, params, seed) |
| `operators/` | One implementation per condition (C0–C5) behind a common interface: `(item, seeded rng) → messages + sampling params`. Exquisite corpse is multi-call and is a **subgraph**. Domain lists, constraints and personas live in versioned data files, not in code |
| `graphs/` | The three phase graphs. Generation fans out with the LangGraph **`Send` API**, with a concurrency limit to respect free-tier quotas |
| `judges/` | The **judge panel** — the original "expert panel" idea in measurable form. In Rung B the experts are originality/appropriateness scorers; in Rung C they become domain experts with routing, behind the same interface |
| `calibration/` | Runs Graph 3 over the human-rated dataset and computes correlations. Runs **before** the experiment |
| `cli/` | `hallucinator generate | curate | judge | analyze --experiment experiments/b1.yaml` |
| `panel/` | Streamlit provider panel (section 3) |

### Why three graphs instead of one
Each phase persists its output. Changing the judge means re-running Graph 3 only, on the same `curated.jsonl` — a fair comparison, because generations are identical.

### Reproducibility inside LangGraph
- Typed graph state (Pydantic).
- A run **manifest** per execution (section 4).
- LangGraph **checkpointer**, so long runs resume after a crash.

### Embeddings
🟡 A small local embedding model (runs on CPU): free, deterministic, no quota. ❓ Exact model chosen in the plan.

---

## 3. Providers and panel — 🟡 proposed, awaiting review

### Decisions already made 🟢
- `freellmapi` is the default source for **both** the judge and the generator, while costs are zero. Ollama remains the local alternative.
- Swapping providers must not require code changes. Paid providers will be added later as new profiles.
- A **local web panel** is used to switch providers and keys.

### Provider profile 🟡
Defined in `config/providers.yaml`:

```yaml
profiles:
  freellmapi-judge:
    kind: openai-compatible        # ollama | openai-compatible | anthropic
    base_url: http://127.0.0.1:<port>/v1
    model: <pinned model id>
    api_key_env: FREELLMAPI_KEY    # name of the env var, never the key itself
    verify_routed_via: true
    allow_fallback: false
    max_rpm: <int>
    timeout_s: 60
roles:
  generator: freellmapi-generator
  judge: freellmapi-judge
  embedder: local-embedder
```

The experiment config may override roles. The generator and judge profiles must resolve to **different model families**; this is validated at start-up.

### freellmapi guardrails 🟡
- Runs as a **separate local service**, at a pinned version, and is **not** a dependency of this repository.
- Automatic fallback silently changes which model answered, which would contaminate the data. Every experimental response therefore has its `X-Routed-Via` header checked against the pinned model. On mismatch, the response is **discarded and logged**, never used silently.
- Each upstream provider's terms apply. Some free tiers may train on submitted data, which is acceptable here because the inputs are public datasets.

### Panel 🟢 Streamlit (🟡 details)
- Binds to `127.0.0.1` only.
- Scope: provider configuration only. Run monitoring is a possible later addition and out of scope now.
- Pages:
  - **Roles:** pick a profile per role.
  - **Profiles:** create, edit, delete.
  - **Keys:** always masked (`sk-...a3f9`).
- A **Test** button per role makes a real call and shows who answered (`X-Routed-Via`), latency, and remaining quota when the provider exposes it.
- Keys never appear in logs, error messages or manifests.
- ❓ Key storage: `.env` file (gitignored) vs. the OS credential store (Windows Credential Manager via `keyring`).

---

## 4. Data and reproducibility — 🟡 proposed, awaiting review

### Layout
```
data/items/aut_items.yaml          # the 13 AUT items
data/operators/*.yaml              # distant domains, constraints, personas
data/external/<dataset>/           # downloaded datasets + LICENSE/ATTRIBUTION (only if license allows)
experiments/b1.yaml                # experiment definition
runs/<run_id>/                     # gitignored working outputs
  manifest.json
  generations.jsonl
  curated.jsonl
  scores.jsonl
  report.md
results/<experiment_id>/           # committed, published results
.cache/                            # gitignored response cache (SQLite)
```

### Run manifest
- Git commit SHA.
- Hash of the experiment config.
- Resolved provider profiles (no keys).
- The **actual model per call** (from `X-Routed-Via`).
- Seeds.
- Hash of `uv.lock`.
- Dataset hashes.
- Start and end timestamps.

### Seeds and determinism
A global seed derives a per-task seed. Note: hosted providers do not guarantee bit-exact outputs even with a seed. Reproducibility therefore means:
1. Re-running the **analysis** from the saved outputs is exact.
2. Re-running the **generation** is statistically comparable, not identical.

The README must state this honestly.

### Datasets
Before any external dataset is committed or downloaded by a script:
- verify its license;
- add attribution;
- record the decision in an ADR.

Candidates:
- "Lost in Translation?" on Zenodo (CC BY 4.0 🟢).
- Beaty & Johnson on OSF (❓ license).
- Patterson et al. 2023, multilingual (❓ license).

### Tooling 🟡
- Python 3.12 or 3.13 (❓ which), pinned with `uv`. The system Python 3.14 is too new for some ML wheels.
- `uv.lock` is committed.

---

## 5. Living documentation and AGENTS.md — 🟡 proposed, awaiting review

### Decision already made 🟢
Documentation is the main deliverable and is produced alongside the code. AGENTS.md must improve itself with good practices learned during development.

### Structure
| File | Purpose |
|---|---|
| `README.md` | What, why, current results, how to run, honest limitations |
| `AGENTS.md` | Rules for coding agents (and humans): workflow, guardrails, definition of done |
| `docs/architecture.md` | Current architecture (kept in sync with the code) |
| `docs/adr/NNNN-<title>.md` | One file per decision: context, options, decision, consequences |
| `docs/preregistration.md` | Hypotheses and analysis plan, frozen before runs |
| `docs/experiments/<id>.md` | Lab notebook: what ran, with which manifest, results, interpretation |
| `docs/research/references.md` | Literature and what we learn from each work |
| `docs/ideas.md` | Idea backlog (Rung C, platform, new operators) — ideas, not decisions |
| `docs/lessons-learned.md` | Incidents and pitfalls, dated, with root cause |

### Self-improvement loop
1. A mistake, incident, or review finding happens.
2. A dated entry goes into `docs/lessons-learned.md`: what happened, root cause, fix.
3. If the lesson generalizes, it is **promoted to a rule in AGENTS.md**, linking back to the incident.

Rules are born from real incidents rather than invented up front, with the exception of a security baseline (section 6).

### Definition of done for every change
- Did it change a decision? → ADR.
- Did it produce a result? → experiment log.
- Did it reveal a pitfall? → lessons-learned entry.
- Did it change behavior? → `docs/architecture.md` / README updated.

A change that alters behavior without updating docs is incomplete.

### Pending ❓
Whether to use the author's `savc-pipeline` skill as the documentation backbone. It already implements status markers, incident-born guardrails and living docs.

AGENTS.md is written **after** this spec is approved, so that it reflects decided architecture only.

---

## 6. Testing and security — 🟡 proposed, awaiting review

### Testing
- `pytest`, with no network in CI.
- **Unit tests:**
  - operators (deterministic prompt construction for a given seed);
  - profile resolution and the family-mismatch check;
  - `X-Routed-Via` verification;
  - cache keys;
  - statistics functions against known values.
- **Graph tests** use LangChain's fake chat models (e.g. `FakeListChatModel`), so the wiring is tested without LLM calls.
- **Smoke experiment:** a tiny config (1 item × 2 conditions × n=2), run locally against Ollama before any real run.
- **Calibration** is itself a gated step with a preregistered threshold.
- `ruff` for lint and format, plus a type checker (❓ `mypy` or `pyright`).

### Security baseline
No document prevents CVEs; tooling does. AGENTS.md will *require* the following:

**Dependencies**
- `uv.lock` committed.
- `pip-audit` in CI.
- Dependabot for Python and GitHub Actions.
- GitHub Actions pinned by commit SHA.

**Secrets**
- Only in `.env` (gitignored) or the OS keyring; `.env.example` holds placeholders.
- Secret scanning via a `gitleaks` pre-commit hook and in CI.
- Log redaction.

**LLM output is untrusted data**
- Never executed (no `eval`/`exec`/shell).
- Never rendered as raw HTML (`unsafe_allow_html` stays off in Streamlit).
- Parsed into Pydantic models; invalid output is recorded as a failure, never coerced.

**Prompt injection**
- Dataset text and generations are interpolated into judge prompts. They are delimited and labeled as data.
- Judge output is structured and parsed.

**Panel**
- Localhost only.
- Masked keys.
- Never commits `.env`.

**Cost and quota guard**
- Max calls per run and max requests per minute per profile.
- A budget cap field, unused while everything is free, ready for paid providers.

**Third-party services**
- `freellmapi` at a pinned version, run outside the repo.

---

## Out of scope (Rung B)
- Public platform / web UI for end users.
- Rung C (open ideas per knowledge area, domain-expert routing, Elo tournaments).
- Evolutionary loops (OpenEvolve-style populations) — candidate for a future ADR.
- Collecting new human ratings (e.g. in Portuguese).

## Consolidated open questions ❓
1. Ocsai API: cost and rate limits.
2. Licenses of the Beaty & Johnson and Patterson et al. datasets.
3. Sample size per item × condition.
4. Exact C1 temperature.
5. Minimum judge–human correlation threshold.
6. Generator and judge models (different families), and the embedding model.
7. Python 3.12 vs 3.13; `mypy` vs `pyright`.
8. Key storage: `.env` vs OS keyring.
9. Use of `savc-pipeline` as the documentation backbone.
