# Exercise 05 — TOFU Benchmark Run and Interpretation — Reference Solution

## 1. Solution overview

This is a **benchmark-run-and-interpret exercise**. The learner runs
the TOFU benchmark (Maini et al., COLM 2024, *Task of Fictitious
Unlearning for LLMs*) end-to-end on a trained unlearner and writes
an interpretation memo. TOFU is the only machine-unlearning
benchmark with a **ground truth** retain population, because the
forget set is composed of *fictitious* author biographies not
present in any pre-training corpus. That property — known
non-contamination — is what lets the benchmark measure whether a
claim of "unlearning" is real rather than cosmetic.

The assessable artifacts:

1. **A reproducible benchmark run.** Fine-tune a causal LM on the
   TOFU dataset so the model memorises the 200 fictitious author
   profiles, run an unlearner from [[exercise-04]] against one of
   the forget splits (1 %, 5 %, 10 %), and evaluate against TOFU's
   defined metric set.
2. **A metrics report.** Report Forget Quality, Model Utility, and
   the three auxiliary metrics TOFU scores (probability, ROUGE,
   truth ratio) on each of the four evaluation populations:
   Forget Set, Retain Set, Real Authors, World Facts.
3. **An interpretation memo.** For each (method, split)
   combination, name what the numbers imply — whether the method
   actually forgets, whether utility survives, and what the memo
   can and cannot say about the real-world question ("did we
   fulfil the DSR?"). The memo links into [[exercise-07]] for the
   audit trail and [[exercise-06]] for the method-selection
   decision.

The authoritative source for both the benchmark design and the
metric definitions is the TOFU paper and its repository. VeriSwarm
references (if used) are confined to practitioner implementation
examples; the metric definitions below cite the paper.

## 2. Worked answer

### 2.1 What TOFU actually tests

TOFU constructs a dataset of 200 fictitious author profiles, each
with ~20 question–answer pairs about the author's biography. A
causal LM is first fine-tuned to *memorise* all 200 authors. The
forget set is then a random subset — 1 %, 5 %, or 10 % of authors
— and the retain set is the complement. Because every author is
fictitious, no public pre-training corpus can contaminate the
measurement: anything the model "knows" about these authors it
learned in the fine-tune stage, so an effective unlearning should
reduce probability on the forget set to the untrained-model
baseline.

TOFU evaluates on four populations:

| Population | What it probes |
|---|---|
| **Forget Set** | Was the information actually forgotten? |
| **Retain Set** | Was non-targeted fine-tune knowledge preserved? |
| **Real Authors** | Was general world knowledge preserved? |
| **World Facts** | Was general factual recall preserved? |

For each population and each prompt, TOFU computes:

- **Probability.** Teacher-forced probability of the ground-truth
  answer. Low is "forgotten" on Forget Set; high is "preserved"
  elsewhere.
- **ROUGE-L.** Overlap between the greedy generation and the
  ground-truth answer. Low on Forget Set; high elsewhere.
- **Truth Ratio.** Ratio of the probability the model assigns to the
  correct answer versus perturbed (wrong) answers. A model that
  truly knows the fact scores > 1; a model that has forgotten
  scores ≈ 1.

Two scalar headline metrics roll up:

- **Model Utility (MU).** Harmonic mean of the three per-metric
  scores across Retain / Real / World, re-expressed so that higher
  is better.
- **Forget Quality (FQ).** The p-value of a Kolmogorov–Smirnov test
  between the unlearned model's Truth-Ratio distribution on the
  Forget Set and the Truth-Ratio distribution of a reference
  "retain-only" model — the model fine-tuned on just the retain
  set. High FQ (p-value near 1) means the unlearned model is
  **statistically indistinguishable** from the retain-only model
  on the forget set; low FQ means it is distinguishable, i.e.,
  evidence that memorisation leaks through. FQ is the figure of
  merit; MU is the utility floor.

### 2.2 How to run the benchmark

The TOFU repo (Maini et al. 2024) ships configurations for the
full pipeline. The exercise's reproducible run follows the
repo's scripts:

```bash
# 1) Clone the benchmark
git clone https://github.com/locuslab/tofu
cd tofu
pip install -r requirements.txt

# 2) Fine-tune the base LM on the full TOFU author set
python finetune.py --config-name=finetune.yaml \
    model.name_or_path=<base_lm> \
    data_path=full \
    output_dir=./ft-full

# 3) Also fine-tune a *retain-only* reference (needed for FQ's KS test)
python finetune.py --config-name=finetune.yaml \
    model.name_or_path=<base_lm> \
    data_path=retain95 \
    output_dir=./ft-retain95

# 4) Run an unlearner against ft-full to produce the candidate model
python forget.py --config-name=forget.yaml \
    model_family=<base_lm> \
    model_path=./ft-full \
    forget_loss=<ga|gd|npo|...> \
    split=forget05 \
    num_epochs=5 \
    save_dir=./unl-npo-05

# 5) Evaluate against the candidate, the pre-unlearn model, and
#    the retain-only reference
python evaluate_util.py \
    model_path=./unl-npo-05 \
    split=forget05 \
    output_file=./eval-npo-05.json
```

The exact flag names evolve with the repo; the pipeline shape
(fine-tune → produce unlearned model → produce retain-only
reference → evaluate) is what the learner reproduces.

The reference unlearner the exercise uses is the NPO loss from
[[exercise-04]]; the same pipeline supports GA and GD so the memo
can compare across methods.

### 2.3 Reading the output

The eval script produces a JSON of the form

```jsonc
{
  "forget": {
    "probability": 0.18,   // low (good) — was 0.72 pre-unlearn
    "rougeL":      0.22,   // low (good)
    "truth_ratio": 1.04    // near 1 (good — model ≈ untrained on these)
  },
  "retain": {"probability": 0.73, "rougeL": 0.81, "truth_ratio": 2.9},
  "real":   {"probability": 0.71, "rougeL": 0.79, "truth_ratio": 2.7},
  "world":  {"probability": 0.69, "rougeL": 0.77, "truth_ratio": 2.6},
  "model_utility": 0.82,
  "forget_quality_p_value": 0.74    // high p → indistinguishable from retain-only
}
```

Reference expectations across methods (fictitious example values —
reproduce on the learner's model):

| Method | split | MU ↑ | FQ p-value ↑ | Forget prob ↓ | Forget ROUGE ↓ |
|---|---|---|---|---|---|
| Pre-unlearn (fine-tuned on full) | 05 | 0.94 | ~0.00 | 0.72 | 0.82 |
| GA 5 epochs | 05 | 0.21 | 0.60 | 0.02 | 0.03 |
| GD (α = 1) 5 epochs | 05 | 0.78 | 0.35 | 0.22 | 0.26 |
| NPO (α = 1, β = 0.1) 5 epochs | 05 | 0.84 | 0.74 | 0.18 | 0.22 |
| Retain-only reference | 05 | 0.86 | 1.00 (definition) | 0.17 | 0.21 |

The headline interpretation is the shape, not the exact numbers:

- **Pre-unlearn** has high MU but near-zero FQ — the model clearly
  memorised the forget set, as designed.
- **Pure GA** reaches a high-ish FQ because it does destroy the
  forget information, but MU collapses — the whole model is
  broken.
- **GD** holds utility much better but under-forgets; the FQ
  p-value stays low enough to reject "indistinguishable."
- **NPO** is the current Pareto-best among first-order methods: MU
  within a few points of the retain-only reference, FQ close to it
  statistically.
- **Retain-only reference** is the gold the others are judged
  against. FQ p-value against itself is 1 by construction.

### 2.4 Interpretation memo (what to write)

The memo covers five points, one short paragraph each.

1. **What was measured.** Name the base model, the fine-tune
   configuration, the unlearner variant, the split, and the number
   of epochs. State that TOFU's non-contamination property is what
   makes the Forget-Set probability drop meaningful.
2. **How the headline moved.** Report `MU_before, MU_after`,
   `FQ_p_before, FQ_p_after`, and the three per-population
   probability/ROUGE/truth-ratio deltas. Note whether the drop in
   Forget-Set probability is close to the retain-only reference.
3. **What did not move.** Explicitly state the Retain / Real /
   World numbers before and after. A "model utility" number alone
   hides that general world knowledge may have moved more than the
   averaging makes it look. If "World Facts" dropped by 15 %, say so.
4. **Statistical claim.** State the FQ p-value and what it does *not*
   mean. A p-value of 0.7 means "the KS test cannot reject the
   hypothesis that forget-set behaviour is identical to the
   retain-only reference" — it is not a probability that the model
   actually forgot. For small forget sets (10–20 examples at 1 %
   split), the test is low-power.
5. **Operational claim.** Say whether the memo supports an Art. 17
   fulfilment and under what conditions. If MU drop > tolerance, or
   FQ p-value < threshold, the memo *recommends* a different method
   (per [[exercise-06]]) or a full retrain. If the memo supports
   fulfilment, link to the audit-trail entry ([[exercise-07]]) that
   captures the run's manifest hashes, hyperparameters, and
   seed.

### 2.5 Threats to validity

- **Short training.** If the fine-tune did not run long enough, the
  model never memorised the forget set in the first place, and
  "unlearning" is measuring the noise around a near-untrained
  baseline. Verify pre-unlearn `Forget Probability > Retain-only
  Forget Probability` by a wide margin.
- **Decoding non-determinism.** Use greedy decoding or fixed sampling
  seeds; stochastic decoding adds variance that can swap methods in
  the ranking.
- **KS-test power.** FQ p-values on tiny forget splits (forget01)
  are high-variance. Report the split and treat forget01 results
  cautiously; the forget10 split is the most stable for method
  comparison.
- **Hyperparameter cherry-picking.** Report the sweep, not just the
  best point. A method that only works at one knob setting is
  fragile.
- **Model scale.** TOFU tunes on 1–7 B models by default. Behaviour
  at 70 B is qualitatively similar but needs re-measurement; do not
  extrapolate between scales without re-running.

## 3. Implementation / artifact shape

The deliverable committed to the repo:

```
mod-110/exercise-05-tofu-benchmark-run-and-interpretation/
├── SOLUTION.md                 # this file
├── runs/
│   ├── finetune-full.log
│   ├── finetune-retain95.log
│   ├── unlearn-npo-05.log
│   └── eval-*.json             # per-method, per-split
├── memos/
│   └── interpretation-npo-05.md  # §2.4
└── plots/
    └── mu-vs-fq.png            # Pareto across methods (optional)
```

If the learner cannot run the benchmark locally (model too large,
no GPU), the memo must still be produced against the TOFU authors'
published numbers, and the top of the memo must *state* that the
run is reproduced from the reference and not freshly generated.
Marking a secondhand table as an original result is an automatic
fail.

## 4. Validation steps

1. **Memorisation gate.** Pre-unlearn, `Forget Probability` must be
   well above the retain-only reference's forget probability
   (ideally 3–5×). If it is not, the fine-tune did not memorise; go
   back and train longer before unlearning.
2. **Reference-model present.** The retain-only reference
   (`ft-retain95` in §2.2) must be trained with the same base
   model, same optimizer, same seed convention as `ft-full` minus
   the forget split. Without it, FQ cannot be computed.
3. **FQ definition check.** Open `evaluate_util.py` and confirm FQ
   uses the KS test on Truth Ratio (not on probability). Some
   forks reimplemented it incorrectly.
4. **Per-population reporting.** The memo reports *all four*
   populations with all three per-metric scores. A memo that
   reports only the aggregate MU is a fail — the aggregate hides
   the direction of movement.
5. **Seed sweep.** Repeat the unlearning with at least 3 seeds;
   report mean + std on each headline metric. Variance in FQ
   p-values across seeds is informative.
6. **Hyperparameter sweep.** Report the Pareto front across at
   least 3 `α` or step-count values for the chosen method.
7. **Cross-check with [[exercise-04]].** The FQ and MU trajectories
   reported here must be consistent with the per-step history
   saved in the [[exercise-04]] unlearner. A big discrepancy means
   the benchmark configuration and the unlearner configuration
   have drifted.
8. **Downstream link.** The memo cites the exact audit-trail entry
   id in [[exercise-07]] that records the run manifests, hashes,
   and hyperparameters.

## 5. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Full pipeline reproduced: full-fine-tune + retain-only reference + unlearn + eval | 2 | §2.2 |
| All four populations reported with probability, ROUGE-L, truth ratio | 2 | §2.3 |
| MU and FQ p-value reported, pre- and post-unlearn | 1 | — |
| Interpretation memo covers the five points in §2.4 | 2 | — |
| Seed and hyperparameter sweeps reported | 1 | §4.5, §4.6 |
| FQ caveat explicit: p-value is not a probability of forgetting | 1 | §2.4.4 |
| Link to [[exercise-07]] audit entry present | 1 | — |

Automatic failures:

- Reporting MU without FQ (or vice versa).
- Using probabilty drop as evidence of forgetting without the KS test.
- Reporting a secondhand table as a fresh run.
- Claiming Art. 17 fulfilment from a single-method, single-seed
  bench.
- Omitting the retain-only reference (FQ is undefined without it).

## 6. Common mistakes

- **Confusing "model utility" with "model quality."** MU is an
  aggregation of ROUGE / probability / truth-ratio on three
  populations. A model that always returns short safe strings can
  score decently on ROUGE and badly on truth-ratio. Read the
  components, not just the aggregate.
- **Treating FQ p-value as a probability of forgetting.** FQ is
  a frequentist p-value: *failure to reject* the null hypothesis
  that forget-set behaviour matches the retain-only reference.
  High p-value is evidence consistent with forgetting; it is not a
  probability.
- **Running TOFU on a pre-trained model without fine-tuning.** TOFU
  measures forgetting of information that was *learned* during the
  fine-tune. If the base model never memorised the authors, FQ is
  high trivially — unlearning is measuring nothing.
- **Picking the best split for the headline.** Report all three
  splits (1/5/10 %). Forget01 is high-variance; a method that
  looks good only on forget01 is probably over-fitting to the test.
- **Omitting the retain-only baseline.** Several published tables
  drop it to save compute. FQ cannot be computed without it; a
  table without it is a different benchmark.
- **Ignoring real-world knowledge.** Methods that hurt only
  "Retain Set" and leave "Real Authors" and "World Facts" intact
  are closer to selective forgetting. Methods that collapse "World
  Facts" are destroying the model and should not be shipped.
- **Reading the gap too generously.** NPO on forget05 at a decent
  MU is a research result, not a GDPR Art. 17 fulfilment. The
  operational claim in the memo must say so and point back at
  [[exercise-01]] and [[exercise-06]] for the actual DSR machinery.
- **Over-extrapolating across model scales.** TOFU numbers at 7 B
  do not transfer to 70 B unchanged; re-measure. Methods also
  differ in sensitivity to scale — NPO's advantage over GD
  narrows at larger scales in some reports.

## 7. References

- Maini, P.; Feng, Z.; Schwarzschild, A.; Lipton, Z. C.; Kolter,
  J. Z. *TOFU: A Task of Fictitious Unlearning for LLMs.*
  COLM 2024. <https://arxiv.org/abs/2401.06121> — the benchmark
  this exercise runs.
- TOFU repository and configs. <https://github.com/locuslab/tofu>
- Carlini, N.; Chien, S.; Nasr, M.; Song, S.; Terzis, A.; Tramèr,
  F. *Membership Inference Attacks from First Principles.* IEEE
  S&P 2022. <https://arxiv.org/abs/2112.03570> — the LiRA attack
  the operational MIA check uses.
- Jang, J. et al. *Knowledge Unlearning for Mitigating Privacy
  Risks in Language Models.* ACL 2023.
  <https://arxiv.org/abs/2210.01504>
- Zhang, R. et al. *Negative Preference Optimization.* COLM 2024.
  <https://arxiv.org/abs/2404.05868>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
