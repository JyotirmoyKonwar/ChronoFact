# ChronoFact: plan to reach a conference-ready submission

Branch: `Bench`. Target venues: ACL / EMNLP / NAACL (ARR cycle) or NeurIPS / ICLR (datasets and benchmarks or main track).

This plan turns the current system and LongMemEval-S evaluation into a paper reviewers can accept. It is ordered by
priority: phase 0 must land before any new number is reported, because it changes the headline comparison.

Legend: **[code]** change in this repo · **[run]** compute job · **[write]** paper text · **[human]** manual annotation.
Effort is a rough estimate on the reported setup (1 GPU for vLLM, 64 CPU cores); a full write-path ingest is ~3.7 h.

---

## Phase 0. Fix the confounded comparison (blocker)

The current headline compares ChronoFact and RAG under different answer and packing settings:

- `chronofact/pipeline.py:30`: `MODE = {"MODEL5": "reason", "RAG_plain": "short"}`. ChronoFact answers with
  step-by-step reasoning (512 output tokens); RAG must answer in one short phrase (128 tokens).
- `chronofact/reader.py` `Retriever.retrieve`: ChronoFact packs turns with the `skip` packer, RAG with `stop`.

Tasks

- [ ] **[code]** Make answer mode and packer per-system settings in `config.yaml` (e.g. `eval.systems.<name>.mode`,
      `.packer`) instead of hard-coded in `pipeline.MODE` and `reader.retrieve`. Keep the old values reachable for
      reproducing the README numbers.
- [ ] **[code]** Add system `RAG_reason` = RAG_plain context + `reason` mode + `skip` packer (identical answer step to
      ChronoFact). Also `MODEL5_short` (ChronoFact context, `short` mode) to show the 2×2.
- [ ] **[code]** Extend `SYSTEMS` handling in `pipeline.py`, `scripts/report.py`, `scripts/audit.py` and the paired
      bootstrap so any number of systems is reported against a chosen reference.
- [ ] **[run]** Retrieve + answer + judge for the new systems on the existing snapshots (no re-ingest needed).
- [ ] **[write]** Headline result becomes ChronoFact vs `RAG_reason`. Report the 2×2 (context × answer mode) so the
      memory effect and the prompting effect are separated.

Done when: the report has a table with all four cells, and the main claim is stated against the matched baseline.

---

## Phase 1. Ablations (what each component contributes)

The README itself says the lead is "the combined effect". A paper needs per-component evidence.

### 1a. Read-path ablations (reuse existing snapshots; retrieve + answer + judge only)

- [ ] **[code]** Add switches in `reader.py` / `config.yaml` `read:` and register each as a system variant:
  - `no_time_scope`: `question_mode` always returns `now`-less "all facts" (no `time_filter`).
  - `no_history_lines`: drop the "previously X, a to b" suffix in `fact_line`.
  - `no_validity_dates`: fact lines without "since / to / no longer current".
  - `facts_only`: no evidence turns (`facts_budget_frac = 1.0`, no turn packing).
  - `turns_only`: already `RAG_reason`.
  - `no_fact_gate`: rank all eligible facts by cosine, no tau / seeds / topic gate.
  - `facts_budget_frac` sweep: 0.2, 0.35 (current), 0.5.
- [ ] **[run]** One retrieve/answer/judge pass per variant (5 checkpoints × 4 budgets).

### 1b. Write-path ablations (need a new ingest each, ~3.7 h + W2)

- [ ] **[code]** Switches in `writer.py` / `config.yaml` `write:`:
  - `no_verify`: skip quote check and NLI (keep every extracted fact).
  - `no_nli`: quote check only.
  - `no_supersede`: states are appended, never superseded (old value stays active).
  - `overwrite`: states are overwritten in place (no history kept), the Mem0-style behaviour.
  - `no_salience`: every user turn goes to extraction (cost and quality effect of the filter).
  - `no_w2`: evaluate on `v5` snapshots instead of `v5f` (no new ingest needed).
  - `no_label`: use the extractor's family/slot directly, skip the labeller and rich statements.
- [ ] **[code]** Results for each variant go to their own results directory (`CF_RESULTS=results_abl_<name>`) sharing
      the LLM / embedding / NLI caches (`CF_CACHE_DIR`) so unchanged calls are free.
- [ ] **[run]** Ingest + W2 + retrieve + answer + judge per variant. Budget: ~6 variants × ~5 h.
- [ ] **[code]** `scripts/report.py`: ablation table (overall + per question type, Δ vs full system with paired CIs).

Done when: every component in the method section has a row in the ablation table with a CI.

---

## Phase 2. Stronger and published baselines

- [ ] **[code]** Baseline adapter interface: `ingest(history) -> state`, `context(question, as_of, budget) -> text`,
      so external systems plug into the same checkpoints, budgets, answerer and judge.
- [ ] **[code]** **Mem0** (open-source, run with Llama-3.1-8B via the same vLLM endpoint and Contriever or its default
      embedder, documented).
- [ ] **[code]** **Zep / Graphiti** (open-source Graphiti with the same LLM). This is the closest prior work
      (bi-temporal edges with valid/invalid times), so it matters most.
- [ ] **[code]** **A-MEM** and/or **MemGPT / Letta** (at least one; pick by ease of running locally).
- [ ] **[code]** **LongMemEval's strongest RAG recipe**: session/round keys with fact-augmented keys, time-aware query
      expansion, chain-of-note reading.
- [ ] **[code]** **Long-context baseline**: whole history in context. Raise `--max-model-len` in
      `scripts/serve_llama.sh` (Llama 3.1 supports 128k; check GPU memory) or run it on a subset.
- [ ] **[code]** **Oracle**: only the evidence sessions in context (upper bound).
- [ ] **[run]** All baselines under the same budgets where applicable (long-context and oracle are budget-free; report
      their token counts).
- [ ] **[write]** Table placing ChronoFact against published LongMemEval-S numbers (cite the source, note judge and
      answerer differences).

Done when: ChronoFact is compared with at least Mem0, Zep/Graphiti, strong RAG, long-context and oracle under one protocol.

---

## Phase 3. Generality: benchmarks, backbones, judge

### 3a. More benchmarks
- [ ] **[code]** `chronofact/data.py`: a loader interface so a dataset yields `Session` / `Question` the same way.
- [ ] **[code]** **LoCoMo** loader (10 long conversations, QA with temporal and multi-hop categories). Note: two
      speakers, not user/assistant; decide whose facts are stored and document it.
- [ ] **[code]** **LongMemEval-M** (much longer histories) on a subset, to support the "bounded as memory grows" claim.
- [ ] Optional: MemoryAgentBench or a small purpose-built temporal-update stress set.
- [ ] **[run]** Full pipeline on each.

### 3b. More backbones
- [ ] **[run]** Answerer swap: Llama-3.1-70B (or 3.3-70B) and one API model, on existing contexts (answer + judge only).
- [ ] **[run]** Write-path swap: at least one larger extractor on a subset (e.g. 100 histories) to show the method is not
      tied to 8B quirks (see `label_kind_from_family` workaround in `config.yaml`).

### 3c. Judge
- [ ] **[code]** Add LongMemEval's official judge prompt and a GPT-4o (or equivalent strong) judge profile. Keep the
      Llama judge as a secondary column.
- [ ] **[human]** Expand the hand audit from 50 to ~300 verdicts, stratified by system and question type; report
      agreement and Cohen's κ for each judge.

Done when: results hold on ≥ 2 benchmarks, ≥ 2 answerers, and the main judge is the community-standard one.

---

## Phase 4. Measure the headline claims directly (intrinsic memory quality)

Current claims count rejections and operations; they do not measure correctness.

- [ ] **[code]** Export a sample of write decisions for annotation: per history, stored facts with their quote and
      source turn, rejected facts with reason, and SUPERSEDE / CLOSE events with old and new fact
      (`ops_log`, `facts`, `episodes` tables).
- [ ] **[human]** Annotate ~30 histories (2 annotators, adjudicated):
  - **Fact precision**: is each stored fact supported by the user's words? (tests "never stores what was not said")
  - **Fact recall**: from the turns, list the facts that should be stored; how many were captured?
  - **Rejection precision**: of NLI rejections, how many were actually true facts (over-strictness cost)?
  - **Supersession precision / recall**: correct closes, wrong closes, missed updates.
  - **Time resolution accuracy**: is `valid_from` correct for relative expressions ("two weeks ago")?
- [ ] **[code]** Report these with and without NLI (`no_verify` ablation) to show the verification trade-off.
- [ ] **[code]** `forget` test at scale: after `Store.forget`, no retrieval for the deleted content returns it
      (automatic check over all facts of a sample of histories).
- [ ] **[run]** Memory-growth curve: context tokens, retrieval latency and accuracy vs history length (checkpoints
      already exist; add LongMemEval-M).

Done when: every claim in the README "Why ChronoFact" table is backed by a measured number or removed.

---

## Phase 5. Weak spots and error analysis

ChronoFact loses on abstention (57.5 vs 81.7), on what the assistant said (66.1 vs 78.2) and is low on preference (16.2).

- [ ] **[human]** Error analysis: sample ~100 wrong answers across types, categorise (extraction miss, verification
      reject, wrong supersession, retrieval miss, packing cut, answer reasoning error, judge error).
- [ ] **[code]** Abstention: try a calibrated abstention rule (e.g. abstain when no gated fact and low top turn score)
      or an answer-prompt variant; report on held-out half (cross-fitted like `tau`).
- [ ] **[code]** Assistant content: store assistant commitments/recommendations as a separate fact source, or always
      reserve budget for assistant turns when the question references "you said / recommended".
- [ ] **[code]** Preferences: check extraction recall for preferences in the annotation of phase 4; adjust prompt if needed.
- [ ] **[write]** Qualitative examples: 3–4 worked cases (one win from supersession, one from time scoping, one failure).

---

## Phase 6. Cost and efficiency

- [ ] **[code]** Report per system: LLM calls and tokens at write time, wall time per turn (p50 / p95, already in
      `meta.json`), read latency, context tokens. Same table for Mem0 / Zep / long-context.
- [ ] **[write]** Cost–accuracy plot (accuracy vs write tokens and vs read tokens).

---

## Phase 7. Paper writing

- [ ] **[write]** Problem definition: temporal fact store (facts with subject, slot, value, valid_from, valid_to,
      provenance), operations (add, supersede, close, forget), and time-scoped queries (now, as of t, history).
- [ ] **[write]** Method: write path (salience, grounded extraction, quote + NLI verification, labelling, temporal
      update), W2 reconciliation, read path (time scope, hybrid retrieval, fact gate, budgeted packing), answering.
      Self-contained: no references to the internal design spec ("README 5.4", "Appendix C.x") or internal names
      ("Model4", "Model5", "W2" without definition).
- [ ] **[write]** Clear contribution statement vs Mem0, Zep/Graphiti, A-MEM, MemGPT: write-time grounding with
      verbatim quote + entailment, explicit validity intervals with kept history, question-time temporal scoping,
      strict token budgets. Feature comparison table.
- [ ] **[write]** Experimental setup (datasets, checkpoints, budgets, cross-fitting, bootstrap, judge), main results,
      ablations, intrinsic memory quality, generality, cost, error analysis.
- [ ] **[write]** Limitations: 8B extractor quirks, cost of write path, English only, user-centric memory, dependence on
      NLI model; tuning on test halves (cross-fitted, no separate dev set).
- [ ] **[write]** Ethics: personal data in memory, deletion (`forget`), consent; licences (Contriever is CC BY-NC;
      Llama 3.1 terms; LongMemEval filler from ShareGPT/UltraChat).
- [ ] **[write]** Figures: architecture, write path, read path, results by budget, results by question type,
      cost–accuracy, memory-growth curve.
- [ ] **[write]** Reproducibility checklist (venue-specific): seeds, pinned models (`model_pins.json`), hardware,
      run time, cached re-runs.

---

## Phase 8. Methodology hygiene and submission

- [ ] **[code]** Separate dev set: tune `tau`, `min_entail`, abstention rule on LongMemEval questions outside -S (or a
      held-out slice), or state clearly that cross-fitting on the test halves is the protocol.
- [ ] **[write]** Disclose that thresholds were calibrated with Muse Spark (proprietary) labels; show the `local`
      profile with README-default thresholds as a robustness row.
- [ ] **[write]** Multiple-comparison note for per-type results (Holm or report as exploratory).
- [ ] **[code]** Anonymised code release for double-blind review (anonymous.4open.science); strip author/GitHub names,
      hackathon reference and demo video link from the submitted README.
- [ ] **[code]** `scripts/report.py` emits LaTeX tables directly from `runs.parquet` so paper numbers are generated, not
      copied.

---

## Suggested order and rough compute

| order | phase | needs re-ingest? | rough compute |
|---|---|---|---|
| 1 | 0. matched baseline | no | ~3 h (answer + judge for 2 new systems) |
| 2 | 1a. read ablations | no | ~1.5 h per variant |
| 3 | 3c. strong judge + audit | no | API cost + annotation time |
| 4 | 1b. write ablations | yes | ~5 h per variant |
| 5 | 2. external baselines | their own ingest | 1–2 days engineering + runs |
| 6 | 4. intrinsic annotation | no | ~2–3 annotator-days |
| 7 | 3a/3b. LoCoMo, LongMemEval-M, bigger models | yes | 1–2 days |
| 8 | 5, 6. error analysis, cost | no | 1 day |
| 9 | 7, 8. writing, anonymisation | no | 1–2 weeks |

Minimum viable submission: phases 0, 1, 2 (Mem0 + Zep + long-context + oracle), 3a (LoCoMo), 3c, 4, and 7.
