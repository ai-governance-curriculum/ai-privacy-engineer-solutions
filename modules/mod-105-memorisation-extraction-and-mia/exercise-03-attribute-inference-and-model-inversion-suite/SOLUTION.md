# Exercise 03 — Attribute Inference and Model Inversion Suite — Reference Solution

## 1. Solution overview

This is an **implementation + design** exercise. The learner builds a small
suite of two closely related — and often conflated — attacks, then produces
a decision memo that explains the difference to a non-ML reviewer.

- **Attribute Inference Attack (AIA):** given a target model, a record
  with some attributes known, infer an unknown sensitive attribute of the
  same record. Canonical reference: Fredrikson, Lantz, Jha, Lin, Page,
  Ristenpart, "Privacy in Pharmacogenetics: An End-to-End Case Study of
  Personalized Warfarin Dosing," USENIX Security 2014.
- **Model Inversion Attack (MI):** given a target model (and typically a
  label), reconstruct a *representative input* for that label. Canonical
  references: Fredrikson, Jha, Ristenpart, "Model Inversion Attacks that
  Exploit Confidence Information and Basic Countermeasures," ACM CCS 2015;
  and (for deep nets) Zhang, Jia, Chen, Lyu, Wu, Zhao, "The Secret Revealer:
  Generative Model-Inversion Attacks Against Deep Neural Networks," IEEE
  CVPR 2020.

Both map to OWASP ML Top-10 **ML03 Model Inversion / Membership
Inference** (<https://owasp.org/www-project-machine-learning-security-top-10/>)
and MITRE ATLAS **AML.T0024** and **AML.T0025 Exfiltration via Cyber
Means** (<https://atlas.mitre.org/>). They are different *techniques* with
different *risks*; the decision memo must separate them.

Deliverables:

1. A working **AIA baseline** against a tabular classifier (attacker knows
   some columns, infers one hidden sensitive column).
2. A working **MI attack** against a simple image classifier (gradient- or
   generator-based reconstruction of a class-representative image).
3. A **"what did we actually learn?" memo** that prevents the common
   misreading of MI reconstructions.

## 2. Worked implementation

### 2.1 Attribute inference — reference approach

The simplest (and often strongest) AIA is a *marginal-plus-model* attack:

- Attacker has `(x_known, y)` pairs for many records and knows the true
  `x_sensitive` for enough of them to fit a surrogate.
- Attacker fits `p(x_sensitive | x_known, y, target_prediction)` using the
  target model's prediction as an extra feature.
- Attacker scores precision, recall, and — critically — **lift over the
  marginal baseline** `p(x_sensitive)`.

Why lift: an AIA that "achieves 85% accuracy" on a sensitive attribute
whose marginal is 85% has learned nothing. Yeom et al. 2018 §V.A formalises
this trap and the memo in §2.5 below walks the learner through it.

```python
# aia.py
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import classification_report

def marginal_baseline(y):
    return float(np.mean(y))  # for binary case; extend to multinomial

def fit_aia(x_known, target_pred, y_sens):
    X = np.hstack([x_known, target_pred.reshape(-1, 1)])
    return GradientBoostingClassifier().fit(X, y_sens)

def evaluate_aia(model, x_known, target_pred, y_sens):
    X = np.hstack([x_known, target_pred.reshape(-1, 1)])
    p = model.predict_proba(X)[:, 1]
    pred = (p > 0.5).astype(int)
    acc = (pred == y_sens).mean()
    base = max(np.mean(y_sens), 1 - np.mean(y_sens))  # majority-class baseline
    return {
        "accuracy": float(acc),
        "majority_baseline": float(base),
        "lift_over_majority": float(acc - base),
        "report": classification_report(y_sens, pred, output_dict=True),
    }
```

### 2.2 Model inversion — reference approach

For a small image classifier `f(x) → p(y)` the canonical gradient-based MI
(Fredrikson, Jha, Ristenpart, 2015) is:

```
x_0 ~ random init
for t = 1..T:
    L(x) = -log p_y(f(x)) + λ · R(x)       # R: a prior / regulariser
    x_{t+1} = x_t - η · ∂L/∂x | project to valid pixel range
return x_T
```

For deep nets, pair with a generator prior `G(z)` and optimise `z` (Zhang
et al. 2020 §3 "generative inversion"):

```python
# mi.py (sketch)
import torch

def invert_class(target_model, label, generator=None, steps=1000, lr=0.05, lam=1e-3):
    device = next(target_model.parameters()).device
    if generator is not None:
        z = torch.randn(1, generator.latent_dim, device=device, requires_grad=True)
        opt = torch.optim.Adam([z], lr=lr)
        for _ in range(steps):
            x = generator(z)
            logits = target_model(x)
            loss = torch.nn.functional.cross_entropy(logits, torch.tensor([label], device=device))
            loss = loss + lam * (z ** 2).mean()         # prior
            opt.zero_grad(); loss.backward(); opt.step()
        return generator(z).detach(), z.detach()
    else:
        x = torch.randn(1, *target_model.input_shape, device=device, requires_grad=True)
        opt = torch.optim.Adam([x], lr=lr)
        for _ in range(steps):
            logits = target_model(x.clamp(-1, 1))
            loss = torch.nn.functional.cross_entropy(logits, torch.tensor([label], device=device))
            loss = loss + lam * torch.var(x)             # smoothness prior
            opt.zero_grad(); loss.backward(); opt.step()
        return x.detach().clamp(-1, 1), None
```

### 2.3 Metrics the learner must report

For AIA: accuracy, precision, recall, F1, **lift over marginal**, lift
over a *shuffled-target* control (target predictions randomly re-assigned —
measures what the target itself contributed).

For MI: pixel MSE vs the nearest training image is **not a sufficient**
privacy metric on its own; it conflates "the attack found a representative
of the class" with "the attack extracted a specific training individual."
Report at minimum:

- A nearest-neighbour similarity to the *training set* (e.g., cosine on a
  pre-trained embedding).
- A nearest-neighbour similarity to the *held-out set* of the same label.
- The ratio of the two (Zhang et al. 2020 Figure 4 reports this comparison).

If the reconstruction looks more like the training set than like held-out
data of the same label, MI has extracted something class-specific; if the
ratio is ~1, MI only found a prototype.

### 2.4 Worked numbers in the memo

<!-- Caveat: fill the table with the learner's own measurements;
do not import numbers from papers as if they applied to this run. -->

| Attack | Target | Metric | Value |
|---|---|---|---|
| AIA (marginal baseline) | tabular classifier | majority-class accuracy | 0.XX |
| AIA (surrogate on targets' preds) | tabular classifier | accuracy | 0.XX |
| AIA lift over marginal | tabular classifier | | 0.XX |
| MI, no generator | image classifier | NN-sim to train / NN-sim to held-out | 0.XX / 0.XX |
| MI, with generator | image classifier | NN-sim to train / NN-sim to held-out | 0.XX / 0.XX |

### 2.5 Decision memo — required structure

Two single pages, in plain English, written for a product manager:

- **What AIA proved (and did not).** If lift-over-marginal is near zero,
  the AIA proved *nothing specific about the model* — the base rate was
  the problem (or the solution) all along. If lift is material, name the
  sensitive attribute and the categorical disclosure risk tier.
- **What MI proved (and did not).** The famous "Fredrikson 2015 face
  reconstruction" image is a class prototype. The memo must state this out
  loud, because stakeholders routinely misread an MI image as "the model
  memorised Alice's face." The right question is **ratio of train-NN to
  held-out-NN**. Call that out explicitly.
- **What to route to downstream modules.** If AIA lift > threshold, route
  to the triage sweep (exercise-06) and, for regulated attributes, to the
  DPIA (mod-111). If MI ratio > threshold on individual training images,
  escalate to the memorisation / extraction evaluation (exercise-04 on text
  side, and an image-side equivalent for reconstructions).

## 3. Validation steps

1. **AIA sanity: shuffled target.** Re-run the AIA with the target
   prediction column *randomly permuted*. The accuracy must drop. If it
   does not drop, the attack is reading `x_known` directly and attributing
   it to the target — the target model is not actually leaking anything new.
2. **AIA sanity: base-rate collapse.** If the sensitive attribute has a
   skewed marginal (e.g., 92/8), check majority-class baseline before
   reporting accuracy. Add lift-over-marginal to every row of the table.
3. **MI sanity: random init vs prior.** With no prior, MI should diverge
   or produce adversarial-looking garbage for most deep nets. With a
   generator prior, MI should produce coherent class prototypes. If a
   no-prior MI already produces coherent faces, the attack code has a
   silent regulariser baked in.
4. **MI sanity: train vs held-out comparison.** Compute nearest-neighbour
   similarity to both the training set and a held-out set of the same
   label. If both are comparable, the attack found a prototype, not a
   training record. Do not misreport it.
5. **Threat-model check.** Both attacks assume at least prediction access.
   State the API surface (softmax, top-k, label-only). Label-only AIA is
   possible but far weaker.

## 4. Rubric or review checklist

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| AIA implementation runs and emits a metrics table | 1 | |
| AIA reports **lift over marginal** (not accuracy alone) | 2 | Majority-class baseline must appear |
| AIA includes a **shuffled-target control** | 1 | Isolates target's contribution |
| MI implementation runs and emits reconstructions | 1 | |
| MI reports train-NN vs held-out-NN similarity ratio | 2 | The core privacy question |
| Decision memo names what MI proved and did not prove | 2 | Explicit anti-misreading paragraph |
| Routing to downstream modules (triage / DPIA) stated | 1 | |

Automatic failures:

- AIA reported as "high accuracy on sensitive attribute" without the
  marginal baseline.
- MI reconstruction presented as "evidence the model memorised Alice"
  without the train-vs-held-out ratio.
- Threat model not stated (no mention of what API surface was assumed).

## 5. Common mistakes

- **Reading an MI reconstruction as "the model memorised this person."**
  Model inversion yields class-representative inputs. The paper that made
  this attack famous (Fredrikson, Jha, Ristenpart, 2015) was clear about
  this and the field has argued over the interpretation ever since. The
  memo must preempt the misreading.
- **Confusing AIA with MIA.** AIA is "infer an unknown *column* of a
  known record." MIA is "decide if a record was in the training set." A
  defender looking at the wrong column of metrics will mis-prioritise
  mitigations.
- **Ignoring the marginal.** Yeom et al. 2018 §V.A is the standard warning:
  a correlation-based AIA can match class-skewed marginals and look
  "accurate" while contributing zero information beyond the base rate.
- **Comparing MI similarity only to the training set.** Without a
  held-out-same-label comparison, the number is meaningless.
- **Running MI with an overly strong prior and calling the result an
  attack.** If `G(z)` is a face generator, `G(z*)` will look like a face
  no matter what. Decompose the reconstruction's provenance in the memo:
  how much came from the prior, how much from the target's gradients?
- **Treating AIA as "just a correlation."** When the sensitive attribute
  is a GDPR Art. 9 special category (health, biometrics, race, religion,
  sexual orientation, trade-union membership, etc.) or a US sensitive
  personal information category under CPRA, a material-lift AIA is a
  regulatory finding regardless of ML sophistication.

## 6. References

- Matthew Fredrikson, Eric Lantz, Somesh Jha, Simon Lin, David Page, Thomas
  Ristenpart. "Privacy in Pharmacogenetics: An End-to-End Case Study of
  Personalized Warfarin Dosing." USENIX Security, 2014.
  <https://www.usenix.org/conference/usenixsecurity14/technical-sessions/presentation/fredrikson_matthew>
- Matthew Fredrikson, Somesh Jha, Thomas Ristenpart. "Model Inversion
  Attacks that Exploit Confidence Information and Basic Countermeasures."
  ACM Conference on Computer and Communications Security, 2015.
  <https://doi.org/10.1145/2810103.2813677>
- Yuheng Zhang, Ruoxi Jia, Hengzhi Chen, Siheng Lyu, Chenchen Wu, Dawn
  Song, Weiyan Zhao. "The Secret Revealer: Generative Model-Inversion
  Attacks Against Deep Neural Networks." IEEE CVPR, 2020.
  <https://arxiv.org/abs/1911.07135>
- Samuel Yeom, Irene Giacomelli, Matt Fredrikson, Somesh Jha. "Privacy Risk
  in Machine Learning: Analyzing the Connection to Overfitting." IEEE CSF,
  2018. <https://doi.org/10.1109/CSF.2018.00027>
- OWASP Machine Learning Security Top 10, ML03.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS, AML.T0024 and AML.T0025. <https://atlas.mitre.org/>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
