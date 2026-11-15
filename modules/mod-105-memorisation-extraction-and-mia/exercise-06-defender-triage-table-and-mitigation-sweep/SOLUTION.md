# Exercise 06 — Defender Triage Table and Mitigation Sweep — Reference Solution

## 1. Solution overview

This is a **design exercise** that consumes the attack outputs of
exercises 01–05 and produces the artefact a privacy engineer actually ships
back to a product team: a **triage table** mapping findings to candidate
mitigations, and a **mitigation sweep** that quantifies the privacy /
utility trade-off for the top candidates.

The deliverable has two parts:

1. **Triage table.** One row per finding (per attack × per asset × per
   sensitivity tier), scored by severity and routed to a candidate
   mitigation. Severity scoring follows NIST AI RMF MEASURE 2.3 "Risk
   measurements include accuracy, precision, recall, and other measures
   relevant to the use case"
   (<https://www.nist.gov/itl/ai-risk-management-framework>) and is
   anchored to the low-FPR cuts defined in exercises 01–02.
2. **Mitigation sweep.** A short, re-runnable experiment comparing the
   top three candidate mitigations on the same target against the same
   evaluation harness, with a decision recorded against an acceptance
   criterion signed off during planning (not after seeing the numbers).

The findings routed here correspond to OWASP Machine Learning Security
Top 10 **ML03 Model Inversion / Membership Inference** and **ML02 Data
Poisoning / Memorisation** (<https://owasp.org/www-project-machine-learning-security-top-10/>)
and to MITRE ATLAS techniques **AML.T0024 Exfiltration via ML Inference
API** and related reconstruction techniques (<https://atlas.mitre.org/>).

## 2. Worked implementation

### 2.1 Triage table — required columns

| Finding ID | Attack | Asset | Sensitivity | Metric | Observed | Threshold | Severity | Candidate mitigations | Owner · Due |
|---|---|---|---|---|---|---|---|---|---|

- **Finding ID:** stable identifier (`MOD105-F-0001` etc.).
- **Attack:** which exercise produced it (`01-shadow-MIA`, `02-LiRA`,
  `03-AIA`, `03-MI`, `04-LLM-extract`, `05-canary`).
- **Asset:** named model + dataset version.
- **Sensitivity:** classification tag from the data catalog. The
  severity gate depends on this. Example tiers: `public`, `internal`,
  `pii-basic`, `special-category` (GDPR Art. 9 or CPRA SPI), `regulated`
  (HIPAA PHI, FERPA, PCI).
- **Metric and threshold:** the metric from the source exercise with the
  pre-registered acceptance threshold; see §2.3 for the registry.
- **Severity:** derived from (gap × sensitivity) — see §2.4.
- **Candidate mitigations:** named interventions from §2.5. More than
  one per row is normal.
- **Owner · Due:** single named engineer, dated commitment.

### 2.2 Worked example row

| Finding ID | Attack | Asset | Sensitivity | Metric | Observed | Threshold | Severity | Candidate mitigations | Owner · Due |
|---|---|---|---|---|---|---|---|---|---|
| `MOD105-F-0003` | `02-LiRA` | `rank-v2` on `clicks-2026Q3` | `pii-basic` | TPR@FPR=0.001 | 0.12 | 0.03 | High | 1. DP-SGD fine-tune (mod-106), 2. training-set dedup + minimisation (mod-103), 3. prediction clipping / top-k-only API | A. Engineer · 2026-12-15 |

<!-- Caveat: the observed/threshold values are illustrative only;
real rows must come from the learner's measured runs. -->

### 2.3 Metric and threshold registry

The threshold is set **before the attack runs** so the report cannot
back-fit a passing threshold to a disappointing number. Default registry
(adjust per product):

| Attack | Metric | Threshold for `public` | for `pii-basic` | for `special-category` / `regulated` |
|---|---|---|---|---|
| 01-shadow MIA | TPR@FPR=0.01 | ≤ 0.20 | ≤ 0.05 | ≤ 0.02 |
| 02-LiRA | TPR@FPR=0.001 | ≤ 0.05 | ≤ 0.03 | ≤ 0.01 |
| 03-AIA | lift over marginal | ≤ 0.10 | ≤ 0.05 | ≤ 0.02 |
| 03-MI | train-NN / held-out-NN ratio | ≤ 1.3 | ≤ 1.1 | ≤ 1.05 |
| 04-discoverable | k-extractable rate, k=100 | ≤ 0.10 | ≤ 0.01 | ≤ 0.001 |
| 04-extractable | PII-hit yield / 10^6 tokens | ≤ 10 | ≤ 1 | 0 |
| 05-exposure | exposure above FP floor (σ) | ≤ 3 | ≤ 2 | ≤ 1 |

These thresholds are **defaults for a classroom exercise** and are *not*
an industry standard — they exist to make the exercise completable and
must be re-set per product in production. State this in the triage
report's preamble.

### 2.4 Severity rule (one line)

```
severity = f( gap_in_sigma_or_log2, sensitivity_tier, blast_radius )
```

Applied consistently:

- **Blocker** — any `regulated` tier finding above threshold.
- **High** — any `special-category` finding, or any `pii-basic` finding
  with gap > 2× threshold, or any public finding with gap > 10× threshold.
- **Medium** — otherwise above threshold.
- **Info** — below threshold but close (within 25%); documented, not
  blocked.

### 2.5 Candidate mitigation catalogue (routing targets)

Each row in the triage table routes to one or more of:

| Mitigation | What it changes | Where it lives | When it fits |
|---|---|---|---|
| DP-SGD fine-tune | gradient-noise mechanism bounding per-example influence; Abadi et al. "Deep Learning with Differential Privacy," ACM CCS 2016 <https://arxiv.org/abs/1607.00133> | mod-106 | High MIA or LiRA TPR at low FPR; LLM memorisation above threshold |
| PATE | teacher ensemble + noisy voting; Papernot et al. "Semi-supervised Knowledge Transfer for Deep Learning from Private Training Data," ICLR 2017 <https://arxiv.org/abs/1610.05755> | mod-106 | High-utility tabular models with enough teacher data |
| Dataset deduplication | collapse near-duplicates before training; Lee et al. 2022 <https://arxiv.org/abs/2107.06499> | mod-103 | LLM memorisation findings correlated with repetition count |
| Training-data minimisation | drop columns, drop records, drop features per purpose | mod-103 | AIA findings pointing at a sensitive column that is not actually needed |
| Output API narrowing | top-k only, label-only, cap decimal precision on softmax | this module | MIA/LiRA findings where the softmax tail is the attacker's signal |
| Confidence masking / temperature scaling | increase label noise at inference | this module | Partial mitigation; often insufficient alone |
| Regularisation + early stopping | reduce train/test gap | this module | Over-fit-driven MIA; Yeom et al. 2018 is the mechanism argument |
| Unlearning / retraining | remove specific records post-hoc | mod-110 | Specific extractions linked to specific records |
| Confidential compute / enclave | protect inference-time inputs | mod-108 | Attack surface is inference-side, not training-side |
| Rate limiting + abuse detection | cap queries per principal per unit time | this module | Shadow-MIA and LiRA need many queries |

### 2.6 Mitigation sweep — the experiment

Re-use the exercise-01 / 02 harness unchanged. For each top-candidate
mitigation, retrain the target (or wrap the serving layer) and re-run the
attack harness. Report:

```
sweep_results.csv
  mitigation, utility_metric, utility_value, attack_metric, attack_value, delta_utility, delta_attack
```

Plot **privacy–utility frontier**: `x = utility`, `y = attack TPR@FPR=0.001`
(or exposure, per attack family). Each candidate is a point. The
acceptance criterion drawn in planning is a horizontal line (attack metric
ceiling). Any point below the line and above the utility floor is
shippable; anywhere else is a reject.

```python
# sweep.py (skeleton)
def run_sweep(target_factory, attack_fn, utility_fn, mitigations):
    rows = []
    baseline = target_factory(mitigation=None)
    rows.append(("baseline", utility_fn(baseline), attack_fn(baseline)))
    for name, apply in mitigations.items():
        m = apply(target_factory())
        rows.append((name, utility_fn(m), attack_fn(m)))
    return rows
```

### 2.7 Decision record

For each finding whose top-chosen mitigation is applied, record (in the
sweep README or ticket):

- Pre-registered threshold and sensitivity tier.
- Baseline attack metric and utility.
- Mitigation candidate chosen; sweep result (attack metric, utility).
- Decision: accept / reject / escalate.
- Owner, date, review signoff.

## 3. Validation steps

1. **Threshold registered before attack numbers are read.** Reviewer
   checks the registry file's commit timestamp against the attack-report
   timestamp; if the registry post-dates the attack, severity is suspect.
2. **No "everyone" rows.** Every triage row has a single-named owner.
3. **Sensitivity tag matches the catalog.** The sensitivity column is not
   free-text — it comes from the data catalog tag. Join and check.
4. **Mitigation candidate list is non-empty for every High/Blocker row.**
   "We should probably look into this" is not a mitigation.
5. **Sweep uses the same attack harness as the finding.** You can't score
   one mitigation with Shokri MIA and another with LiRA — pick one per
   finding and keep it fixed.
6. **Utility is measured.** A privacy mitigation that drops utility to
   zero is not a mitigation. The sweep must report utility alongside
   attack metric.
7. **Decision is recorded even when the answer is accept-as-is.**
   Acceptance of a finding above threshold needs a documented reason — a
   downstream control, a business justification, a time-boxed waiver.

## 4. Rubric or review checklist

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Triage table has all columns, no blanks on any row | 2 | |
| Metric/threshold registry committed before attack runs | 2 | Timestamp check |
| Severity derived consistently from (gap × sensitivity) | 1 | Not case-by-case |
| Candidate mitigations named per row, from the catalogue | 1 | |
| Mitigation sweep runs ≥ 3 candidates per top finding | 2 | |
| Sweep plots the privacy–utility frontier | 1 | |
| Decision record stored per finding (accept / reject / escalate) | 1 | |

Automatic failures:

- Setting thresholds after seeing the attack numbers.
- Routing a Blocker finding to "we'll monitor it."
- Shipping a mitigation that drops utility to zero and calling it a win.
- Claiming a mitigation worked without re-running the same attack
  harness.

## 5. Common mistakes

- **Mixing severity and priority.** Severity is "how bad." Priority is
  "when we do it." A critical-severity finding can be deprioritised if a
  compensating control is in place, but the row must still say
  "critical."
- **Setting thresholds after seeing the numbers.** This is the single
  biggest failure mode of post-hoc risk work. Register thresholds in
  planning.
- **Over-indexing on DP-SGD.** DP-SGD is the headline mitigation but it
  is not free: utility costs, hyper-parameter sensitivity, and privacy
  accounting (see mod-106). A minimisation finding is often addressed
  more cheaply by dropping a column than by rebuilding the whole
  training stack.
- **Scoring MI "severity" the same way as MIA.** MI reconstructions are
  regulated-disclosure findings only when the train-vs-held-out ratio is
  elevated; MIA is a per-example disclosure question. Keep the scoring
  rules separate.
- **Forgetting the sensitivity tier.** The same TPR@FPR against a
  special-category dataset is far worse than against a public one. Do
  not report raw numbers without the tier.
- **Treating "rate limiting" as the whole answer for LiRA.** Rate limits
  slow attacker throughput but do not change per-query leakage.
  Combine with a mechanism that changes the signal (DP, output
  narrowing).
- **No decision record.** Triage without a decision is a ticket queue,
  not a program.

## 6. References

- Martín Abadi et al. "Deep Learning with Differential Privacy." ACM
  CCS, 2016. <https://arxiv.org/abs/1607.00133>
- Nicolas Papernot, Martín Abadi, Úlfar Erlingsson, Ian Goodfellow,
  Kunal Talwar. "Semi-supervised Knowledge Transfer for Deep Learning
  from Private Training Data." ICLR 2017.
  <https://arxiv.org/abs/1610.05755>
- Katherine Lee et al. "Deduplicating Training Data Makes Language
  Models Better." ACL 2022. <https://arxiv.org/abs/2107.06499>
- Samuel Yeom, Irene Giacomelli, Matt Fredrikson, Somesh Jha. "Privacy
  Risk in Machine Learning: Analyzing the Connection to Overfitting."
  IEEE CSF 2018. <https://doi.org/10.1109/CSF.2018.00027>
- Nicholas Carlini et al. "Membership Inference Attacks From First
  Principles." IEEE S&P 2022. <https://arxiv.org/abs/2112.03570>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS. <https://atlas.mitre.org/>
- NIST AI Risk Management Framework (AI RMF 1.0), MEASURE and MANAGE.
  <https://www.nist.gov/itl/ai-risk-management-framework>
