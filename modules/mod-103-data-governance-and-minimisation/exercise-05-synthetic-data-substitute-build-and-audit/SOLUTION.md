# Exercise 05 — Synthetic-Data Substitute Build and Audit — Reference Solution

## 1. Solution overview

This is a **design + implementation exercise**. The learner is asked to
substitute a sensitive training or evaluation corpus with a **synthetic
dataset** and then **audit** whether the substitution actually buys the
privacy property it claims. The target properties are, in order:

1. **Fitness for purpose.** The synthetic dataset supports the
   downstream task to within an agreed utility budget.
2. **Reduction of per-individual re-identification risk** relative to
   the source.
3. **Membership-inference and attribute-inference resistance** at a
   stated threshold.
4. **Honest documentation** of what the synthetic data can and cannot
   be used for.

The assessable output has four parts:

1. A **build plan** — source scoping, generator family choice,
   training protocol, and release artifacts.
2. A **worked example** — a small tabular corpus, a generator pipeline
   outline, and a synthetic sibling.
3. An **audit plan** — utility tests, privacy tests, and leakage
   tests, each with a concrete metric and a pass/fail threshold.
4. A **disclosure sheet** — what reviewers and downstream users are
   told, including the generator's lineage and known failure modes.

An "I ran a GAN and shipped the output" answer fails because it does
not audit the privacy claim. An "I removed names" answer fails because
redaction is not synthesis. A synthetic-data release without a stated
privacy threshold and a stated utility budget fails because it has
nothing to pass or fail against.

## 2. Worked answer

### 2.1 Decide whether synthetic data is the right tool

Synthetic data is attractive but expensive. Before building, check:

- **Is the source actually sensitive?** If the source can be used with
  consent or aggregated differently, synthesis may be unnecessary. A
  minimisation review (exercise 03) comes first.
- **Is the downstream task tolerant to distributional error?** Style
  transfer, class-balance augmentation, and schema prototyping tolerate
  synthesis well. Rare-event detection, fraud analysis, and
  individual-level analytics do not.
- **Is there a cheaper alternative?** Differentially private release
  of aggregates (histograms, marginals) may serve the purpose without
  generating full records.
- **Is the regulatory frame compatible?** Under GDPR, synthetic data
  that cannot be linked back to individuals is not personal data. The
  question is "cannot be linked" — a conclusion only an audit can
  support. Claiming "synthetic therefore anonymous" without an audit
  is the error the EDPB opinion on anonymisation warns against.

If synthesis is warranted, continue.

### 2.2 Build plan

**Step 1 — Source scoping.**

- Pick the narrowest source slice that can support the task.
- Record the source corpus (`dataset_id`, `version`, `freeze_date`),
  the lawful basis for processing it to train a generator, and the
  retention clock.
- Decide whether the generator is trained once and shipped (and the
  source destroyed) or kept for refresh. The former gives a stronger
  privacy posture; the latter is cheaper.

**Step 2 — Generator family.**

Choose by data type and risk tolerance:

- **Tabular, structured:** Bayesian-network generator, PrivBayes-style
  mechanism, or a DP-synthesiser (DP-WGAN, DP-GMM). Preference order:
  DP-synthesiser > PrivBayes > plain Bayesian net > plain GAN. Plain
  GANs for tabular data memorise aggressively and are not a safe
  default.
- **Text:** fine-tune a smaller language model under DP-SGD, or
  use a prompt-conditioned generator *without* training on the raw
  text (e.g., "generate 10 support-dialog openings in the style of the
  genre," conditioned on category labels rather than record samples).
- **Image:** DP diffusion / DP GAN; without DP, image generators have
  demonstrable memorisation on small training sets.
- **Timeseries:** condition on marginals and reconstructed spectra
  rather than per-sequence copies.

If differential privacy is used, record **(ε, δ)** with the composition
accounting (not just a per-step claim). ε values are only comparable
when the composition and the adjacency definition are stated.

**Step 3 — Training protocol.**

- Train in the privacy-plane account (same isolation discipline as the
  vault — exercise 04). The raw corpus does not leave this plane.
- Fix random seeds; pin model code; snapshot the training config.
- Produce a **generator artifact** that the privacy plane signs. The
  artifact is what crosses the trust boundary, not the raw data.
- If DP is used, lock ε at model release and refuse to let later
  sampling compute tighter bounds; later sampling is post-processing
  and does not loosen the bound, but it must not be used to claim one.

**Step 4 — Release artifacts.**

- The **synthetic dataset** (`dataset_id.syn`), with the same datasheet
  schema as exercise 01, explicitly marked `source_is_synthetic = true`
  and linked to the generator artifact.
- The **generator card** — generator family, training hyperparameters,
  DP accountant (if used), stability tests, warnings.
- The **audit report** (see 2.4).
- The **ML-BOM slice** linking the synthetic dataset to its generator
  and the generator to its training corpus (see exercise 06).

### 2.3 Worked implementation — small tabular corpus

Scenario: a tabular corpus `cust-demand-v2` with 50k rows and
columns `(region, age_bucket, product_tier, monthly_orders,
avg_order_value, is_lapsed)`. The downstream task is to train a demand
forecaster. Age, region, and order value are joint-identifying for a
subset of rows.

Build outline (illustrative):

```python
# pipelines/synth/cust_demand.py
from synthcity.plugins import Plugins

source = load_frozen_dataset("cust-demand-v2", version="v2.0.0")
source = enforce_minimum_bucket_size(source, by=["region","age_bucket"], k=50)

generator = Plugins().get("dpgan", epsilon=3.0, delta=1e-5)
generator.fit(source)

synthetic = generator.generate(count=200_000)

sign_and_publish(
    synthetic,
    dataset_id="cust-demand-v2.syn",
    generator_artifact_id="gen-dpgan-cust-demand-v2",
    source_dataset_id="cust-demand-v2",
    source_version="v2.0.0",
    epsilon=3.0, delta=1e-5,
)
```

The `enforce_minimum_bucket_size` call is the first privacy control:
remove cells that would otherwise show up as a single row in the
training signal. The DP mechanism (ε=3.0, δ=1e-5) is the second. The
audit in 2.4 is the third — the one that decides whether these were
enough.

### 2.4 Audit plan

Three audit tracks, run as separate CI jobs with explicit thresholds.

**Track A — Utility.**

- **Downstream-task utility.** Train the downstream model on the
  synthetic data; evaluate on a held-out *real* test set. Record delta
  vs training on the real data. Pass threshold is task-specific and
  agreed in advance (e.g., ≤ 2 percentage-point drop in RMSE or ≤ 5%
  AUROC drop).
- **Marginal fidelity.** Compare per-column histograms and pairwise
  correlations between real and synthetic. Use a bounded statistic
  (total-variation distance, Jensen-Shannon divergence); set a per-
  column threshold.
- **Cell-size fidelity.** Compare joint-bucket counts for the columns
  most likely to carry identifying information. Note where synthetic
  smooths out a cell size that was small in the source — that is
  *good* for privacy and bad for rare-event utility.

**Track B — Privacy (record-level).**

- **Nearest-neighbour distance distribution.** For each synthetic row,
  find the nearest real row and compute a distance. If the synthetic
  nearest-neighbour distribution is shifted *left* of a reference
  (real-to-real holdout) distribution, the generator is reproducing
  training rows. Threshold: no more than X% of synthetic rows closer
  than any real-to-real nearest-neighbour baseline.
- **Exact-duplicate test.** Count synthetic rows that match a real row
  on an identifying tuple. Threshold: 0 for narrow identifiers,
  small-and-justified otherwise. Document any exception.
- **Outlier-preservation test.** Check whether rare values appear in
  the synthetic data without aggregation. Any rare value reproduced
  verbatim is a leak.

**Track C — Privacy (attack-based).**

- **Membership-inference attack.** Train a shadow model and evaluate
  AUC of membership inference against the generator's training
  members versus non-members. Threshold: AUC ≤ 0.6 (closer to 0.5 is
  better). Document the attacker's budget.
- **Attribute-inference attack.** For each sensitive attribute,
  attempt to infer it from the remaining columns in the synthetic
  data and compare to inference from a baseline (real-column marginal
  distribution). Threshold: inference accuracy ≤ baseline + ε'
  (chosen per attribute).
- **Model-inversion probe.** For a text or image generator,
  deliberately prompt for memorised strings (exact prefix completion,
  canary test). Threshold: no recovery of any canary token with
  probability above a documented baseline.

Each track's thresholds live in `audits/<dataset_id>.syn/thresholds.yaml`
and are set *before* the audit runs, not after. An audit that moves
thresholds post-hoc has lost its value.

### 2.5 Disclosure sheet (hand-off)

For each release, publish a short sheet containing:

- **Source corpus** and version, with explicit "not personal data of
  individuals outside the source" claim and the reasoning.
- **Generator family and parameters**, including ε, δ, and composition.
- **Known uses** — tasks for which utility was audited.
- **Prohibited uses** — rare-event detection, individual-level
  analytics, any task that requires out-of-distribution coverage not
  demonstrated in audit.
- **Known failure modes** — e.g., smooths out small cells, inflates
  tails, loses seasonality beyond one cycle.
- **Residual risk** — the specific attack-based bounds and the
  conditions under which they hold (attacker budget, auxiliary data
  assumed).
- **Lineage** — pointers to the datasheet, the generator card, the
  audit report, and the ML-BOM slice.

The disclosure sheet is the artifact that lets a downstream team decide
whether the synthetic dataset is fit for *their* purpose. Without it,
synthetic data silently drifts into inappropriate uses.

## 3. Validation steps

1. **Pre-registration check.** Thresholds were set before the audit
   ran. The thresholds file predates the audit report by commit date
   / signed timestamp.
2. **Utility sanity.** A downstream task trained on the synthetic data
   outperforms a "no-data" baseline (e.g., predict the mean). If it
   does not, the synthetic data is not useful and the audit is moot.
3. **Nearest-neighbour test visibility.** The audit shows the full
   distribution (histogram), not just a single statistic.
4. **Canary test.** For text or sequence data, a canary string was
   inserted into the training set; the audit tested for recovery and
   reported the result.
5. **DP accounting.** If DP was used, the audit report includes the
   composition and the accountant used; ε and δ are stated per the
   same adjacency definition.
6. **Reproducibility.** The build is reproducible from the pinned
   source, pinned generator code, and pinned seed. A reviewer can
   re-run it.
7. **Policy check.** The release is marked `source_is_synthetic = true`
   in the catalog and the downstream use list matches the datasheet's
   "intended uses" section.
8. **Boundary check.** The raw source did not leave the privacy plane
   during training; only the generator artifact and the synthetic
   output did.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Source scoping with lawful basis, retention, and plane isolation | 1 | Training outside an isolated plane fails |
| Generator family chosen with a stated reason, DP parameters named when used | 2 | "A GAN" without justification fails |
| Utility audit with a pre-registered threshold and a real holdout | 1 | Post-hoc thresholds fail |
| Record-level privacy audit: NN distribution, duplicates, outliers | 2 | One-metric audits fail |
| Attack-based privacy audit: MIA and AIA with documented attacker budget | 2 | Missing either fails this item |
| Disclosure sheet with prohibited uses, known failure modes, residual risk | 1 | Missing "prohibited uses" fails |
| Lineage links to datasheet, generator card, audit report, ML-BOM slice | 1 | Missing ML-BOM link is an item fail |

Automatic failures:

- Claiming "synthetic therefore anonymous" without a record-level and
  attack-based audit.
- Setting thresholds after the audit ran.
- Reporting ε without the composition or the adjacency definition.
- Training the generator in the application plane.
- Releasing without a disclosure of prohibited uses.

## 5. Common mistakes

- **"Synthetic therefore anonymous."** The EDPB and Article 29 WP
  guidance treat this as a conclusion, not a label. If the data can be
  linked back by any plausible means, it is still personal data.
- **No utility baseline.** Shipping a generator because "it ran" with
  no held-out real evaluation means there is no evidence the data is
  useful.
- **DP parameters without composition.** "ε = 3" tells you little
  without the composition, the adjacency definition, and the
  attacker's view.
- **Generator trained on *all* features.** Train on the smallest slice
  the downstream task needs. More features in the generator means
  more memorisation surface.
- **Treating image / text as if they were tabular.** Memorisation
  dynamics differ. Image generators memorise exact crops; text
  generators memorise exact strings. The audit set must differ.
- **Audit metric picked after the fact.** A single post-hoc metric
  ("looks fine") is not an audit.
- **Canary-less text synthesis.** For any text generator, a canary
  test is cheap and highly informative. Skipping it is unexplained.
- **Shipping without prohibited-use list.** "Use responsibly" is not a
  policy. Enumerate the uses the audit does not cover.

## 6. References

- Article 29 Data Protection Working Party, Opinion 05/2014 on
  Anonymisation Techniques (WP216). <https://ec.europa.eu/justice/article-29/documentation/opinion-recommendation/index_en.htm>
- Regulation (EU) 2016/679 (General Data Protection Regulation),
  Recital 26 on anonymous information, Arts. 4(1), 4(5), 5, 25, 32,
  35. <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- NIST SP 800-188 (final) — De-identification of Government Datasets
  Containing Personally Identifiable Information. <https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-188.pdf>
- NIST AI Risk Management Framework (AI RMF 1.0), Govern/Measure
  functions. <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST AI 600-1 — Generative AI Profile, risks related to
  memorisation and training-data provenance.
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
- OWASP Machine Learning Security Top 10 — ML04 Membership Inference
  Attack. <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — techniques related to ML Training Data Manipulation
  and Model Inversion (current IDs on the ATLAS site).
  <https://atlas.mitre.org/>
- VeriSwarm Trust Center — practitioner reference on synthetic-data
  documentation practices. <https://veriswarm.ai/trust>
