# Exercise 01 — Shadow-Model MIA End to End — Reference Solution

## 1. Solution overview

This is an **implementation exercise** with an attached reporting artifact. The
learner builds the Shokri-style shadow-model membership inference attack
(Shokri, Stronati, Song, Shmatikov, "Membership Inference Attacks Against
Machine Learning Models," IEEE S&P 2017) end-to-end against a target
classifier, then produces an evaluation that the defender can act on.

The assessable artifact is a short pipeline plus a one-page report. The
pipeline must show:

1. A **target model** trained on a private split `D_target_train`.
2. **k shadow models** of the same architecture, each trained on a disjoint
   shadow split with a known in-distribution / out-of-distribution label.
3. An **attack model** (per class, or one global) trained on the shadow
   models' softmax outputs labelled `in`/`out`.
4. An **evaluation** of the attack on the held-out target data, reporting
   **balanced accuracy, AUC, and TPR at low FPR (1%, 0.1%)** — not accuracy
   alone — because the privacy-relevant question is "how reliably can the
   attacker pick out *some* training members with few false positives?"
   (Carlini et al., "Membership Inference Attacks From First Principles,"
   IEEE S&P 2022 makes the TPR-at-low-FPR argument explicit and this
   exercise follows the same reporting convention.)

The MIA capability this exercise realises corresponds to OWASP Machine
Learning Security Top 10 risk **ML03: Model Inversion / Membership
Inference** (see <https://owasp.org/www-project-machine-learning-security-top-10/>)
and to MITRE ATLAS technique **AML.T0024 Exfiltration via ML Inference API /
Membership Inference** (see <https://atlas.mitre.org/>).

## 2. Worked implementation

### 2.1 Threat model, written down

- **Adversary capability:** black-box query access to the target model,
  receiving softmax (or at minimum top-k) probabilities per query. No access
  to weights, gradients, loss, or training set.
- **Adversary knowledge:** the data distribution and the model family. This
  matches the "shadow training with similar distribution" assumption in
  Shokri et al. 2017, §IV.
- **Attack goal:** given a candidate record `(x, y)`, decide whether `x` was
  in the target model's training set.
- **Defender assumption:** the defender wants a published number for the
  MIA risk against a particular (model, dataset) pair as part of a pre-release
  privacy review — see NIST AI RMF **MEASURE 2.7** ("AI system security and
  resilience … are evaluated and documented")
  (<https://www.nist.gov/itl/ai-risk-management-framework>).

### 2.2 Reference pipeline (Python / PyTorch-style pseudocode)

Any framework works — the code below is deliberately minimal.

```python
# shadow_mia.py
import numpy as np
from sklearn.model_selection import StratifiedShuffleSplit
from sklearn.metrics import roc_auc_score, roc_curve

RNG = np.random.default_rng(0)

def train_shadow(x, y, arch_factory, train_cfg):
    """Train one shadow model and return (model, in_idx, out_idx)."""
    sss = StratifiedShuffleSplit(n_splits=1, test_size=0.5, random_state=RNG.integers(1e9))
    (in_idx, out_idx), = sss.split(x, y)
    model = arch_factory()
    model.fit(x[in_idx], y[in_idx], **train_cfg)
    return model, in_idx, out_idx

def build_attack_examples(shadow_model, x_in, y_in, x_out, y_out):
    """Return (softmax_vec, true_label, member_flag) per example."""
    p_in  = shadow_model.predict_proba(x_in)
    p_out = shadow_model.predict_proba(x_out)
    rows = []
    for p, y in zip(p_in, y_in):   rows.append((p, int(y), 1))
    for p, y in zip(p_out, y_out): rows.append((p, int(y), 0))
    return rows

def run_shadow_mia(x_pool, y_pool, target_model, arch_factory, train_cfg,
                   k=16, per_class=True):
    attack_rows = []
    for _ in range(k):
        shadow, in_idx, out_idx = train_shadow(x_pool, y_pool, arch_factory, train_cfg)
        attack_rows += build_attack_examples(
            shadow, x_pool[in_idx], y_pool[in_idx],
            x_pool[out_idx], y_pool[out_idx])

    # One attack model per class (Shokri §IV), or a single global model.
    from sklearn.ensemble import GradientBoostingClassifier
    attackers = {}
    if per_class:
        for c in np.unique(y_pool):
            rows_c = [(p, m) for (p, y, m) in attack_rows if y == c]
            if len(rows_c) < 50: continue
            P = np.stack([r[0] for r in rows_c]); m = np.array([r[1] for r in rows_c])
            attackers[c] = GradientBoostingClassifier().fit(P, m)
    else:
        P = np.stack([r[0] for r in attack_rows])
        m = np.array([r[2] for r in attack_rows])
        attackers["*"] = GradientBoostingClassifier().fit(P, m)
    return attackers

def score_target(attackers, target_model, x_eval, y_eval, true_member_flag,
                 per_class=True):
    probs = target_model.predict_proba(x_eval)
    scores = np.zeros(len(x_eval))
    for i, (p, y) in enumerate(zip(probs, y_eval)):
        clf = attackers.get(int(y)) if per_class else attackers["*"]
        if clf is None:
            scores[i] = p[int(y)]  # fallback: raw confidence baseline
        else:
            scores[i] = clf.predict_proba(p.reshape(1, -1))[0, 1]
    return _metrics(scores, true_member_flag)

def _metrics(scores, y_true):
    auc = roc_auc_score(y_true, scores)
    fpr, tpr, _ = roc_curve(y_true, scores)
    def tpr_at(fpr_cap):
        idx = np.searchsorted(fpr, fpr_cap, side="right") - 1
        return float(tpr[max(idx, 0)])
    return {"auc": float(auc),
            "tpr@fpr=0.01": tpr_at(0.01),
            "tpr@fpr=0.001": tpr_at(0.001)}
```

### 2.3 Baseline that any serious MIA must beat

Report the **confidence-threshold baseline**: label a sample a member iff
`max softmax > τ`. Tune `τ` on shadow data, then evaluate on the target.
Yeom et al. ("Privacy Risk in Machine Learning: Analyzing the Connection to
Overfitting," IEEE CSF 2018) showed this is already a strong attack for
over-fit models. If the shadow-model attack does not beat this baseline, the
pipeline is wired wrong (the most common bug is a train/eval label leak
between the shadow pool and the target pool — see §5).

### 2.4 Example results table (format the learner must emit)

The one-page report must include a table of this shape. Numbers are
**placeholders to show formatting**, not claims about any real model.

<!-- Caveat: the placeholders below must be replaced with the learner's own
measured numbers from their target model and shadow runs; do not cite these
as facts. -->

| Attack | AUC | Balanced acc | TPR @ FPR=1% | TPR @ FPR=0.1% |
|---|---|---|---|---|
| Confidence-threshold baseline | 0.XXX | 0.XXX | 0.XXX | 0.XXX |
| Shokri shadow MIA (k=16, per-class) | 0.XXX | 0.XXX | 0.XXX | 0.XXX |

Plot a ROC curve on **log-log** axes (per Carlini et al. 2022, §3.2) so the
low-FPR regime is readable.

### 2.5 Handover: what the defender does with this

The report ends with a one-paragraph recommendation that routes to one of:

- **Accept** — residual MIA TPR@FPR=0.1% is below the published threshold
  for the use case (set in the privacy review; see mod-111).
- **Mitigate** — route to mod-105 exercise-06 (triage + mitigation sweep)
  and mod-106 (DP-SGD budget).
- **Block release** — target model shows high TPR at very low FPR and
  processes sensitive-category data (GDPR Art. 9 in the EU; HIPAA PHI in the
  US).

## 3. Validation steps

Learners can self-check without the instructor:

1. **Sanity: shadow == target.** Train a shadow with the same seed, data
   split, and config as the target and evaluate the attack against it. The
   AUC must be noticeably higher than 0.5. If it is ~0.5, the pipeline is
   broken.
2. **Sanity: perfectly generalising target.** Train the target with heavy
   regularisation + dropout + early stopping so the train/test gap approaches
   zero, and re-run. AUC should drop toward 0.5 (Yeom et al. 2018 predicts
   this). If it does not, there is a feature leak.
3. **Hold-out hygiene.** The target training set and the shadow data pool
   must be **disjoint**. Compute the intersection of hashes; it must be
   empty. This is the single most common bug and inflates AUC.
4. **Per-class check.** If the attack AUC is dominated by one class, state
   it — attacks that only work on a minority class still matter and the
   report must not average them away.
5. **Report low-FPR numbers.** A reviewer must be able to answer "how many
   members can the attacker name with only 1-in-1000 false positives?" from
   the report. If the report only shows accuracy or AUC, it fails.

## 4. Rubric or review checklist

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Threat model written out (capability, knowledge, goal) | 1 | Named adversary, named API surface |
| k ≥ 8 shadow models, disjoint pools from the target | 2 | Hash-overlap check shown |
| Attack trained on shadow softmax (not on labels alone) | 1 | Feature vector reproducible |
| Confidence-threshold baseline reported alongside | 1 | Must beat the baseline on at least one metric |
| AUC + balanced accuracy + TPR@FPR=1% + TPR@FPR=0.1% all reported | 2 | Log-log ROC plot required |
| Per-class breakdown present | 1 | Not just the macro number |
| Handover recommendation (accept / mitigate / block) explicit | 1 | Tied to a threshold |
| Code is re-runnable from a single entry point with a fixed seed | 1 | Reproducibility |

Automatic failures:

- Reporting accuracy only. The accuracy of an MIA on a balanced dataset is
  dominated by the uninteresting negatives and hides the privacy risk.
- Shadow pool overlaps the target training set.
- Attack "beats" baseline by less than noise and the learner claims success
  anyway — missing confidence intervals is a fail.

## 5. Common mistakes

- **Shadow / target leakage.** The shadow pool and the target training set
  are drawn from the same larger dataset. If the splits are not disjoint,
  shadow models see target members and the attack appears far stronger than
  it is in deployment. Always compute the hash-intersection.
- **Using one shadow model.** A single shadow model gives a biased attacker.
  Shokri et al. 2017 used dozens; this exercise sets a floor of k=8 for
  classroom runs and recommends k=16–32.
- **Reporting accuracy or AUC only.** Both saturate at low FPR regimes that
  matter for privacy. Carlini et al. 2022 and the ML Privacy Meter paper
  (Murakonda & Shokri, "ML Privacy Meter: Aiding Regulatory Compliance by
  Quantifying the Privacy Risks of Machine Learning," 2020 PPML workshop)
  both argue for TPR-at-low-FPR.
- **Training the attack model only on the target's own confidences.** That
  is not an MIA; that is calibration. The attack model must be trained on
  shadow outputs with ground-truth membership labels.
- **Averaging over classes.** A class-imbalanced leak (e.g., the attack
  works on a rare label with 90% TPR@1%FPR) is a critical finding even if
  the macro number is boring.
- **Not re-training shadows when the target architecture changes.** The
  shadow pool is tied to (architecture, hyper-parameters), not to the
  dataset alone.

## 6. References

- Reza Shokri, Marco Stronati, Congzheng Song, Vitaly Shmatikov.
  "Membership Inference Attacks Against Machine Learning Models." IEEE
  Symposium on Security and Privacy, 2017.
  <https://doi.org/10.1109/SP.2017.41>
- Samuel Yeom, Irene Giacomelli, Matt Fredrikson, Somesh Jha. "Privacy Risk
  in Machine Learning: Analyzing the Connection to Overfitting." IEEE
  Computer Security Foundations Symposium, 2018.
  <https://doi.org/10.1109/CSF.2018.00027>
- Nicholas Carlini, Steve Chien, Milad Nasr, Shuang Song, Andreas Terzis,
  Florian Tramèr. "Membership Inference Attacks From First Principles."
  IEEE Symposium on Security and Privacy, 2022.
  <https://arxiv.org/abs/2112.03570>
- Sasi Kumar Murakonda, Reza Shokri. "ML Privacy Meter: Aiding Regulatory
  Compliance by Quantifying the Privacy Risks of Machine Learning." PPML
  Workshop, NeurIPS 2020. <https://arxiv.org/abs/2007.09339>
- OWASP Machine Learning Security Top 10, ML03 Model Inversion / Membership
  Inference. <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS, Technique AML.T0024 Exfiltration via ML Inference API.
  <https://atlas.mitre.org/techniques/AML.T0024>
- NIST AI Risk Management Framework (AI RMF 1.0), MEASURE function.
  <https://www.nist.gov/itl/ai-risk-management-framework>
