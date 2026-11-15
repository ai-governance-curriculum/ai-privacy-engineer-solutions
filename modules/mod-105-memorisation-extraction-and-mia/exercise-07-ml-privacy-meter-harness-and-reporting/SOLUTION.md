# Exercise 07 — ML Privacy Meter Harness and Reporting — Reference Solution

## 1. Solution overview

This is an **implementation + reporting** exercise. The learner wires the
attacks from exercises 01–02 (and, optionally, 03/04) into a repeatable
harness anchored on **ML Privacy Meter**, the open-source tool from the
Shokri lab (Murakonda, Shokri, "ML Privacy Meter: Aiding Regulatory
Compliance by Quantifying the Privacy Risks of Machine Learning," PPML
Workshop, NeurIPS 2020, <https://arxiv.org/abs/2007.09339>; repo at
<https://github.com/privacytrustlab/ml_privacy_meter>). The output is a
one-page **Privacy Risk Report** the privacy engineer can hand to an
auditor without translating.

Deliverables:

1. A configured, re-runnable **ML Privacy Meter harness** against a named
   target model, fed by the same evaluation splits used in exercise-01 /
   02 so the numbers are directly comparable.
2. A **report generator** that emits a Markdown + PDF report with the
   required fields (§2.3).
3. A **compare-across-runs** workflow: when the target model is retrained,
   the harness re-runs and the report diff is checked into the model
   registry.

The capability this exercises realises corresponds to OWASP ML Top-10
**ML03 Model Inversion / Membership Inference**
(<https://owasp.org/www-project-machine-learning-security-top-10/>) and
to the measurement obligations under NIST AI RMF **MEASURE 2.3** and
**MEASURE 2.7** (<https://www.nist.gov/itl/ai-risk-management-framework>).

## 2. Worked implementation

### 2.1 Harness topology

```
dataset split (pre-registered)
  ├── target training set   -> train target_model
  ├── reference training pool -> train k reference/shadow models
  └── audit set
       ├── known members (subset of target train)
       └── known non-members (held-out from the same distribution)

target_model + reference_models + audit_set
  -> ml_privacy_meter AttackRunner
     -> score vectors
     -> metrics {AUC, balanced acc, TPR@FPR=1%, TPR@FPR=0.1%, TPR@FPR=0.01%}
     -> per-example scores CSV
  -> report_generator
     -> report.md  + report.pdf  + metrics.json
```

The crucial property: the **same split manifest** is used for exercise-01,
exercise-02, and this exercise. Different splits → different attack numbers
→ the report's regressions become meaningless.

### 2.2 Minimal end-to-end script

The ML Privacy Meter API has changed across releases. The following is a
version-agnostic skeleton; pin the version in `requirements.txt` when
submitting.

```python
# run_meter.py
import json, pathlib, yaml
# ml_privacy_meter is the upstream package; pin to the version used in CI.
# Examples from the repo at https://github.com/privacytrustlab/ml_privacy_meter
# show the attack / audit flow.

def load_config(path): return yaml.safe_load(open(path))

def build_target(cfg): ...        # returns target model
def build_reference_pool(cfg): ... # returns list of shadow models
def build_audit_set(cfg): ...     # returns (members, non_members) with labels

def run(cfg_path="meter.yaml"):
    cfg = load_config(cfg_path)
    target = build_target(cfg)
    refs   = build_reference_pool(cfg)
    audit  = build_audit_set(cfg)
    # from ml_privacy_meter import audit as meter_audit
    # scores = meter_audit.run_attack(target, refs, audit, cfg["attack"])
    scores = ...  # per-example attack score vector
    y_true = ...  # 1 for member, 0 for non-member
    metrics = _compute_metrics(scores, y_true)
    out = pathlib.Path(cfg["out_dir"])
    out.mkdir(parents=True, exist_ok=True)
    (out / "metrics.json").write_text(json.dumps(metrics, indent=2))
    _emit_report(out / "report.md", cfg, metrics, scores, y_true)

def _compute_metrics(scores, y_true):
    import numpy as np
    from sklearn.metrics import roc_auc_score, roc_curve
    auc = float(roc_auc_score(y_true, scores))
    fpr, tpr, _ = roc_curve(y_true, scores)
    def tpr_at(cap):
        idx = max(np.searchsorted(fpr, cap, side="right") - 1, 0)
        return float(tpr[idx])
    return {
        "auc": auc,
        "balanced_accuracy": float(((scores > np.median(scores)) == y_true).mean()),
        "tpr@fpr=0.01":  tpr_at(0.01),
        "tpr@fpr=0.001": tpr_at(0.001),
        "tpr@fpr=0.0001": tpr_at(0.0001),
        "n_members": int(y_true.sum()),
        "n_nonmembers": int(len(y_true) - y_true.sum()),
    }
```

`meter.yaml` pins everything a reviewer needs to reproduce:

```yaml
target:
  model_id: rank-v2
  dataset: clicks-2026Q3
  split_manifest: s3://.../splits/clicks-2026Q3-v1.json
reference:
  pool_size: 16
  training_recipe: configs/recipe-v2.yaml
attack:
  name: lira           # or shokri_shadow
  n_shadow_per_example: 16
  statistic: logit_confidence
out_dir: runs/2026-11-14-rank-v2
```

### 2.3 Report template (one page, required)

The Markdown report must include these sections, in this order:

```
# Privacy Risk Report — <model_id> @ <date>

## Scope
- Model: <name, version, architecture>
- Dataset: <name, version, sensitivity tier>
- Split manifest: <path, hash>

## Attack configuration
- Attack: <shokri_shadow | lira_online | lira_offline>
- Reference pool size: <N>
- Statistic: <logit_confidence>
- Harness: ML Privacy Meter <pinned version>

## Headline metric
| Metric | Value | Threshold | Status |
| AUC | 0.XX | — | info |
| Balanced accuracy | 0.XX | — | info |
| TPR @ FPR=1% | 0.XX | <from registry> | pass/fail |
| TPR @ FPR=0.1% | 0.XX | <from registry> | pass/fail |
| TPR @ FPR=0.01% | 0.XX | <from registry> | pass/fail |

## Log-log ROC
<image: roc_loglog.png>

## Per-class breakdown
<table of TPR@FPR=0.1% by class>

## Comparison to previous run
| Metric | Previous | Current | Δ |
| ... |

## Capability gaps
- <what was not tested>

## Recommendation
<accept / mitigate / block> — <one sentence rationale>

## Appendix
- Config file hash
- Random seed
- Reference-pool training commit
```

<!-- Caveat: the thresholds in the "Headline metric" table must
come from the registry in exercise-06 §2.3 for the product in question;
do not restate any paper's numbers as the product's threshold. -->

### 2.4 Run-over-run comparison

The report generator writes `metrics.json` into a run directory keyed by
model version. The comparison step is a simple diff:

```python
# compare.py
import json, sys
def compare(prev_path, curr_path):
    prev = json.load(open(prev_path)); curr = json.load(open(curr_path))
    for k in ("auc", "tpr@fpr=0.01", "tpr@fpr=0.001", "tpr@fpr=0.0001"):
        delta = curr[k] - prev[k]
        print(f"{k}: {prev[k]:.4f} -> {curr[k]:.4f} ({delta:+.4f})")
    # Non-zero exit if any low-FPR TPR regressed by more than 20% relative.
    for k in ("tpr@fpr=0.001", "tpr@fpr=0.0001"):
        if prev[k] > 0 and (curr[k] - prev[k]) / prev[k] > 0.20:
            print(f"REGRESSION on {k}")
            sys.exit(1)
```

Wire this to CI so a retraining PR that silently increases low-FPR MIA
risk fails the check.

### 2.5 Integration with the rest of the module

- Findings exceeding threshold in this report populate the triage table
  in exercise-06.
- If DP-SGD or deduplication is applied as a mitigation, re-run this
  harness and attach the new report to the mitigation sweep.
- For LLMs, extend the harness with the exercise-04 discoverable /
  extractable metrics and the exercise-05 exposure metric. ML Privacy
  Meter is primarily designed for classifier MIA; the LLM-side numbers
  live alongside, not inside, it.

## 3. Validation steps

1. **Reproducibility.** Running `run_meter.py` twice with the same seed
   and split manifest produces identical `metrics.json` and identical
   per-example scores. If not, a non-deterministic path exists.
2. **Split-manifest reuse.** The split manifest hash in this report
   equals the one used by exercises 01 and 02. Grep the manifest hash in
   the three `metrics.json` files; they must match.
3. **Reference pool hygiene.** None of the reference-pool training sets
   intersect the target training set. Compute the intersection of
   training-record hashes and confirm it is empty.
4. **Log-log ROC produced.** Linear-axis ROC hides the regime that
   matters. The report fails if only linear axes are plotted.
5. **Comparison runs.** Train a second target with heavier regularisation
   and re-run. TPR@FPR=0.1% must drop meaningfully (Yeom et al. 2018 is
   the mechanism argument). If it does not, the harness is not actually
   connected to the target.
6. **CI gate behaves.** Deliberately regress the target (shorter
   regularisation, more epochs) and confirm the comparison exits non-zero.

## 4. Rubric or review checklist

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| ML Privacy Meter version pinned in `requirements.txt` | 1 | |
| Config file (YAML) names model, dataset, split manifest, attack | 1 | Reviewable artefact |
| Harness emits `metrics.json`, `report.md`, log-log ROC image | 2 | |
| Low-FPR metrics (1%, 0.1%, 0.01%) all present in the report | 2 | Carlini et al. 2022 convention |
| Run-over-run comparison script and CI gate wired | 2 | Fails on low-FPR regression |
| Split manifest reused across exercises 01/02/07 | 1 | Hash check |
| Capability gaps named in the report | 1 | Non-English, multimodal, etc. |

Automatic failures:

- Reporting macro accuracy or AUC without low-FPR TPR.
- No reference-pool / target disjointness check.
- No CI gate on run-over-run regression for the privacy metric.
- Split manifest differs between exercise-01 and this exercise, making
  any comparison across exercises meaningless.

## 5. Common mistakes

- **Installing ML Privacy Meter unpinned.** The API surface has changed
  across releases. Pin the version and record it in the report.
- **Treating "the tool ran" as "privacy was measured."** A report with
  `{"auc": 0.52}` is technically green but almost certainly means the
  attack is broken or the target is fully generalising. Investigate 0.5s
  before shipping them.
- **Dropping the low-FPR cuts.** A report that only shows AUC hides the
  one number the auditor actually cares about: how reliably a determined
  attacker can identify *some* members with almost no false positives.
- **Rebuilding splits each run.** Random splits on every invocation make
  results incomparable. Pin the split manifest.
- **CI gate on AUC alone.** AUC can be stable while low-FPR TPR spikes
  (Carlini et al. 2022 §5 shows both patterns). Gate on the low-FPR
  metric.
- **Reporting per-run MIA without run-over-run tracking.** The single
  biggest value of the harness is the **trend**. A one-off report gets
  filed; a trend forces action when it moves.
- **Reporting without a sensitivity tier.** The same MIA number is a
  blocker for one dataset and informational for another; the report must
  carry the tier.

## 6. References

- Sasi Kumar Murakonda, Reza Shokri. "ML Privacy Meter: Aiding Regulatory
  Compliance by Quantifying the Privacy Risks of Machine Learning." PPML
  Workshop, NeurIPS 2020. <https://arxiv.org/abs/2007.09339>
- ML Privacy Meter, open-source implementation (Shokri Lab, NUS).
  <https://github.com/privacytrustlab/ml_privacy_meter>
- Reza Shokri et al. "Membership Inference Attacks Against Machine
  Learning Models." IEEE S&P 2017. <https://doi.org/10.1109/SP.2017.41>
- Nicholas Carlini et al. "Membership Inference Attacks From First
  Principles." IEEE S&P 2022. <https://arxiv.org/abs/2112.03570>
- Samuel Yeom et al. "Privacy Risk in Machine Learning: Analyzing the
  Connection to Overfitting." IEEE CSF 2018.
  <https://doi.org/10.1109/CSF.2018.00027>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS. <https://atlas.mitre.org/>
- NIST AI Risk Management Framework (AI RMF 1.0), MEASURE 2.3 and 2.7.
  <https://www.nist.gov/itl/ai-risk-management-framework>
