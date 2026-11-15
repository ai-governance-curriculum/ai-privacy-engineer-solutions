# Exercise 02 — Carlini LiRA Attack Engineering — Reference Solution

## 1. Solution overview

This is an **implementation exercise** that sharpens the generic shadow-MIA
of exercise-01 into the **Likelihood Ratio Attack (LiRA)** from Carlini,
Chien, Nasr, Song, Terzis, Tramèr, "Membership Inference Attacks From First
Principles," IEEE S&P 2022
(<https://arxiv.org/abs/2112.03570>). The learner delivers:

1. An **online** LiRA implementation that, for each candidate example `x`,
   trains `N` IN shadow models (containing `x`) and `N` OUT shadow models
   (not containing `x`), fits two per-example Gaussians over a calibrated
   statistic, and reports a per-example likelihood-ratio score.
2. An **offline** LiRA implementation that reuses one shared pool of
   shadow models and splits them per example at scoring time — far cheaper,
   with the expected loss in attack strength documented.
3. A **log-log ROC plot** and a table of TPR at FPR ∈ {10%, 1%, 0.1%,
   0.01%}. The whole point of LiRA is to be the attack that still has
   non-trivial TPR when FPR is extremely small; reporting macro accuracy
   defeats the exercise.

LiRA realises OWASP ML Top-10 **ML03 Model Inversion / Membership
Inference** (<https://owasp.org/www-project-machine-learning-security-top-10/>)
and MITRE ATLAS **AML.T0024 Exfiltration via ML Inference API**
(<https://atlas.mitre.org/>) under a stronger adversary than
exercise-01: the attacker can train shadow models and query the target with
a logit oracle.

## 2. Worked implementation

### 2.1 What makes LiRA different from the Shokri attack

Shokri-style attacks train **one classifier** over all shadow outputs, so
the attacker's decision boundary is *population-average*. LiRA instead
asks, for each specific `x`, "is the model's observed signal more likely
under the IN distribution for *this* example or the OUT distribution for
*this* example?" The attack is **per-example parametric**: two Gaussians per
example in a calibrated statistic.

### 2.2 The calibrated statistic (do not use raw loss)

Carlini et al. 2022 §4.2 use a *logit-transformed* confidence:

```
phi(f, x, y) = log( p_y(f, x) / (1 - p_y(f, x)) )
```

where `p_y(f, x)` is the softmax probability the model `f` assigns to the
true class `y`. This stabilises variance across easy/hard examples and makes
the Gaussian model plausible. Using raw loss or raw softmax collapses
tails and drops low-FPR TPR by a lot — this is the single biggest
implementation error.

### 2.3 Online LiRA — reference pseudocode

```python
# lira_online.py
import numpy as np

def logit_confidence(p, y):
    p_y = np.clip(p[np.arange(len(y)), y], 1e-7, 1 - 1e-7)
    return np.log(p_y / (1.0 - p_y))

def fit_gaussian(xs):
    return float(np.mean(xs)), float(np.std(xs) + 1e-6)

def online_lira(target_model, candidates, train_in_model_fn, train_out_model_fn,
                N=16):
    """
    candidates: iterable of (x, y) rows the attacker wants to score.
    train_in_model_fn(x, y, seed) trains a shadow model whose training set
        is sampled from a shared pool AND is forced to contain (x, y).
    train_out_model_fn(x, y, seed) trains a shadow model whose training set
        is sampled from the same pool but is forced NOT to contain (x, y).
    """
    scores, labels = [], []
    for (x, y) in candidates:
        phi_in, phi_out = [], []
        for s in range(N):
            m_in  = train_in_model_fn(x, y, seed=s)
            m_out = train_out_model_fn(x, y, seed=s + 10_000)
            phi_in.append(logit_confidence(m_in.predict_proba([x]),  np.array([y]))[0])
            phi_out.append(logit_confidence(m_out.predict_proba([x]), np.array([y]))[0])
        mu_in, sd_in   = fit_gaussian(phi_in)
        mu_out, sd_out = fit_gaussian(phi_out)
        phi_target = logit_confidence(target_model.predict_proba([x]), np.array([y]))[0]
        llr = _log_normal(phi_target, mu_in, sd_in) - _log_normal(phi_target, mu_out, sd_out)
        scores.append(float(llr))
    return np.array(scores)

def _log_normal(x, mu, sd):
    return -0.5 * ((x - mu) / sd) ** 2 - np.log(sd)
```

Attack cost (online): `2 · N · |candidates|` shadow trainings. For N=16 and
1 000 candidates that is 32 000 shadow models. In classroom use, hold
`|candidates|` to the hundreds or move to offline.

### 2.4 Offline LiRA — the cheap variant

Train **one shared pool** of `M` shadow models on bootstrap samples from the
shadow dataset. At scoring time, for each candidate `(x, y)` partition the
pool by *did this shadow include `x` or not*. If `M` is large and sampling
is unbiased, each example will have ~M/2 IN and ~M/2 OUT shadows almost for
free.

```python
# lira_offline.py
def offline_lira(target_model, candidates, pool, membership_table):
    """
    pool: list of shadow models, each with a fixed `id`
    membership_table: dict {(x_hash, y) -> set of pool_ids containing (x,y)}
    """
    scores = []
    for (x, y) in candidates:
        key = (hash_x(x), int(y))
        in_ids  = membership_table.get(key, set())
        out_ids = set(range(len(pool))) - in_ids
        phi_in  = [logit_confidence(pool[i].predict_proba([x]), np.array([y]))[0] for i in in_ids]
        phi_out = [logit_confidence(pool[i].predict_proba([x]), np.array([y]))[0] for i in out_ids]
        if len(phi_in) < 4 or len(phi_out) < 4:
            scores.append(0.0); continue
        mu_in, sd_in   = fit_gaussian(phi_in)
        mu_out, sd_out = fit_gaussian(phi_out)
        phi_t = logit_confidence(target_model.predict_proba([x]), np.array([y]))[0]
        scores.append(_log_normal(phi_t, mu_in, sd_in) - _log_normal(phi_t, mu_out, sd_out))
    return np.array(scores)
```

Expected cost (offline): `M` shadow trainings total — independent of the
number of candidates. Carlini et al. 2022 §5.4 report that offline LiRA
recovers most of the online attack's TPR at low FPR when `M` is large
(their experiments use M in the low hundreds).

### 2.5 Reporting (required format)

The report delivered to the defender must contain all four elements:

1. **Attack configuration table:** N (online) or M (offline), shadow
   training config, candidate set size, target architecture, target training
   recipe.
2. **ROC on log-log axes.** Linear axes hide the low-FPR regime.
3. **TPR @ FPR ∈ {10%, 1%, 0.1%, 0.01%} table.** Carlini et al. 2022 §5
   reports at exactly these cuts; this exercise follows the same convention.
4. **Per-example score histogram**, overlaid with the fitted IN and OUT
   Gaussians for 2–3 illustrative examples (one "easy win," one "hard,"
   one "ambiguous").

<!-- Caveat: numeric TPR values must come from the learner's own
measurements; do not restate TPR@low-FPR from any paper as if it applied to
their target model. -->

### 2.6 Handover

Fit the LiRA result into the module's downstream flow:

- feed per-example scores to **exercise-06 defender triage** so the top-K
  most-leaked examples drive the sampling strategy for any downstream
  decision (unlearning request priorities, DP-SGD budget renegotiation,
  dataset recall).
- if the target handles GDPR Art. 9 special categories or HIPAA 45 CFR
  164.514(b) identifiers, cross-reference the high-score examples with the
  catalog sensitivity tag before escalating.

## 3. Validation steps

1. **Statistic check.** Compute both `phi` (logit) and raw loss on the same
   shadow outputs; the logit statistic should have noticeably lower variance
   per example. If it does not, the logit transform is wrong — typically a
   missing clip on `p_y`.
2. **Gaussian-fit check.** For 20 randomly chosen examples, plot a QQ plot
   of `phi_in` vs a normal distribution. The fit does not need to be perfect
   (Carlini et al. 2022 Figure 2 shows the mild deviations) but should be
   roughly linear. If it is heavy-tailed everywhere, increase `N`.
3. **Online ≥ offline.** For the same N shadow models per example, online
   LiRA must have TPR ≥ offline at every FPR cut in the report. If offline
   wins, the IN/OUT partition in offline is corrupt.
4. **LiRA ≥ Shokri.** LiRA must match or beat exercise-01's shadow-MIA at
   **low FPR** (1% and 0.1%). It is allowed to lose at macro AUC.
5. **Hash disjointness.** The shadow dataset and the target training set
   must be disjoint. Compute hash(x) ∩ hash(x) = ∅ and show it.
6. **Reproducibility.** A single entry point script, fixed seed per shadow,
   and a `scores.csv` + `metrics.json` emitted to a run directory.

## 4. Rubric or review checklist

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| LiRA statistic is logit-of-confidence (not raw loss / softmax) | 2 | Documented in the code |
| Online LiRA implemented with N ≥ 8 per example | 2 | |
| Offline LiRA implemented with M ≥ 64 shadow models | 1 | |
| Per-example IN and OUT Gaussian fits produced and sanity-checked | 1 | QQ plot or histogram |
| TPR reported at FPR ∈ {10%, 1%, 0.1%, 0.01%} | 2 | Log-log ROC plot required |
| LiRA beats exercise-01 Shokri attack at FPR ≤ 1% | 1 | Else investigate |
| Reproducibility: seeded, scripted, emits `metrics.json` | 1 | |

Automatic failures:

- Reporting AUC or accuracy only. LiRA's claim to fame is low-FPR
  performance; without that cut, the exercise is not complete.
- Using raw loss as the statistic. The low-FPR numbers will be meaningless.
- Offline partition that places `x` in both IN and OUT sets.
- Shadow / target data overlap.

## 5. Common mistakes

- **Treating LiRA as "shadow MIA but bigger."** The non-obvious upgrade is
  the **per-example Gaussian**. You cannot train one global attack model
  and call it LiRA.
- **Using raw loss or raw softmax.** The logit-of-confidence statistic is
  the point. The paper includes the derivation (Carlini et al. 2022 §4.2);
  if you skip it the low-FPR TPR collapses.
- **Too-small N / M.** With N=2 online or M=10 offline, the Gaussian fit
  degenerates and the scores are dominated by noise. Floor is N≥8 or
  M≥64; the paper's results use higher.
- **Mixing architectures across the shadow pool.** The IN/OUT Gaussian
  comparison is only meaningful if shadows share architecture and training
  recipe with the target. If you don't know the target's recipe, say so —
  architectural mismatch is a documented capability boundary, not a
  reporting footnote.
- **Forgetting the baselines.** LiRA must be reported against both the
  Shokri-style baseline (exercise-01) and the loss-threshold baseline
  (Yeom et al. 2018), so the defender can see the privacy risk
  independently of attack sophistication.
- **Running LiRA on a DP-trained target and calling it "zero risk."** A
  low TPR at low FPR for a particular (model, dataset) run is not a proof;
  it is a *measurement*. The report must state the epsilon claim if DP was
  used and route to mod-106 for the formal argument.

## 6. References

- Nicholas Carlini, Steve Chien, Milad Nasr, Shuang Song, Andreas Terzis,
  Florian Tramèr. "Membership Inference Attacks From First Principles."
  IEEE Symposium on Security and Privacy, 2022.
  <https://arxiv.org/abs/2112.03570>
- Reza Shokri, Marco Stronati, Congzheng Song, Vitaly Shmatikov.
  "Membership Inference Attacks Against Machine Learning Models." IEEE
  S&P, 2017. <https://doi.org/10.1109/SP.2017.41>
- Samuel Yeom, Irene Giacomelli, Matt Fredrikson, Somesh Jha. "Privacy Risk
  in Machine Learning: Analyzing the Connection to Overfitting." IEEE CSF,
  2018. <https://doi.org/10.1109/CSF.2018.00027>
- Sasi Kumar Murakonda, Reza Shokri. "ML Privacy Meter: Aiding Regulatory
  Compliance by Quantifying the Privacy Risks of Machine Learning." PPML
  Workshop, NeurIPS 2020. <https://arxiv.org/abs/2007.09339>
- OWASP Machine Learning Security Top 10, ML03.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS, AML.T0024 Exfiltration via ML Inference API.
  <https://atlas.mitre.org/techniques/AML.T0024>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
