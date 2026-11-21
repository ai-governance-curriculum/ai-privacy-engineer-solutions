# Exercise 06 — Gradient Reconstruction Attack & Defence — Reference Solution

## 1. Solution overview

This is an **implementation + defence exercise**. The learner stands
up a small FL-style setting, runs a gradient-inversion (gradient
reconstruction) attack against a single client's update to recover
training inputs, and then applies the recommended defences to show
quantitatively that they break the attack. The point is to let a
reviewer see — not just read — that an unaggregated per-client
gradient is sensitive data.

The assessable artifacts:

1. **An attack script** that implements Deep Leakage from Gradients
   (DLG, Zhu et al. 2019) or Inverting Gradients (Geiping et al.
   2020) against a known model, reconstructing (image, label) from a
   single batch gradient.
2. **A defence runner** that re-runs the attack under each of
   (a) secure aggregation (sum of ≥ 8 clients, so the attacker sees
   only the sum — see [[exercise-04]]), (b) DP-FL at the aggregator
   (Gaussian noise + clipping — see [[exercise-05]]), (c) larger
   local batch size (gradient is averaged over more rows), and
   (d) multiple local steps before reporting (what FedAvg actually
   does).
3. **A result table** reporting attack PSNR (or MSE and
   reconstruction-accuracy of labels) under each setting, with a
   plot or image grid of reconstructions.
4. **A written conclusion** that maps the attack to its OWASP ML
   Top 10 entry (LLM02-ish / "sensitive information disclosure" via
   gradients), to the relevant MITRE ATLAS techniques
   (`AML.T0024 Exfiltration via ML Inference API`,
   `AML.T0025 Exfiltration via Cyber Means`) and to NIST AI RMF
   Measure/Manage.

Canonical citations: Zhu, Liu, Han, *Deep Leakage from Gradients*
(NeurIPS 2019); Geiping, Bauermeister, Dröge, Moeller, *Inverting
Gradients — How easy is it to break privacy in federated learning?*
(NeurIPS 2020); the DP-FL defence follows McMahan et al. (2018) and
NIST SP 800-226.

## 2. Worked implementation

### 2.1 Attack: Deep Leakage from Gradients (DLG)

DLG assumes the attacker knows the model architecture and weights
(true in FL: the server knows the global model) and observes a single
client's gradient `∇L(θ; x, y)`. The attacker optimises a dummy
`(x', y')` so that its gradient *matches* the observed one. For small
models and small batches, the reconstruction converges to the real
`(x, y)` up to a permutation.

```python
# dlg_attack.py
from __future__ import annotations

import torch
from torch import nn, optim


def build_victim_model() -> nn.Module:
    # Small CNN from the DLG paper. Keep it small — the attack is
    # harder on big batched gradients from deep nets (see §4 of
    # Geiping et al. 2020).
    return nn.Sequential(
        nn.Conv2d(3, 12, 5, padding=2), nn.Sigmoid(),
        nn.Conv2d(12, 12, 5, padding=2), nn.Sigmoid(),
        nn.Conv2d(12, 12, 5, padding=2), nn.Sigmoid(),
        nn.Flatten(),
        nn.Linear(12 * 32 * 32, 10),
    )


def compute_gradient(model: nn.Module, x: torch.Tensor, y: torch.Tensor) -> list[torch.Tensor]:
    model.zero_grad()
    out = model(x)
    loss = nn.CrossEntropyLoss()(out, y)
    loss.backward()
    return [p.grad.detach().clone() for p in model.parameters()]


def dlg_attack(
    model: nn.Module,
    target_grads: list[torch.Tensor],
    image_shape: tuple = (1, 3, 32, 32),
    num_classes: int = 10,
    iters: int = 300,
    lr: float = 1.0,
    device: str = "cpu",
) -> tuple[torch.Tensor, torch.Tensor]:
    """Return a reconstruction (x', y') whose gradient approximately
    matches `target_grads` under `model`."""
    model.to(device).eval()
    # Freeze model weights — we optimise inputs.
    for p in model.parameters():
        p.requires_grad_(True)  # needed for autograd through grad

    x_hat = torch.randn(image_shape, device=device, requires_grad=True)
    # Soft label; one-hot via gradient descent on logits.
    y_hat = torch.randn(image_shape[0], num_classes, device=device, requires_grad=True)
    opt = optim.LBFGS([x_hat, y_hat], lr=lr)

    target_grads = [g.to(device) for g in target_grads]

    def closure():
        opt.zero_grad()
        pred = model(x_hat)
        y_soft = torch.softmax(y_hat, dim=-1)
        loss = -(y_soft * torch.log_softmax(pred, dim=-1)).sum(dim=-1).mean()
        dummy_grads = torch.autograd.grad(
            loss, model.parameters(), create_graph=True, retain_graph=True,
        )
        grad_diff = sum(
            ((dg - tg) ** 2).sum() for dg, tg in zip(dummy_grads, target_grads)
        )
        grad_diff.backward()
        return grad_diff

    for _ in range(iters):
        opt.step(closure)

    return x_hat.detach(), y_hat.detach().argmax(dim=-1)
```

Running the attack end-to-end on one sample:

```python
# run_attack.py
import torch
from torchvision import datasets, transforms
from dlg_attack import build_victim_model, compute_gradient, dlg_attack


def _psnr(a: torch.Tensor, b: torch.Tensor) -> float:
    mse = ((a - b) ** 2).mean().item()
    if mse == 0.0:
        return float("inf")
    return 10 * torch.log10(torch.tensor(1.0 / mse)).item()


def main():
    torch.manual_seed(0)
    model = build_victim_model()
    tfm = transforms.Compose([
        transforms.Resize(32),
        transforms.ToTensor(),
    ])
    ds = datasets.CIFAR10("./data", train=True, download=True, transform=tfm)
    x, y = ds[0]
    x = x.unsqueeze(0)
    y = torch.tensor([y])

    target_grads = compute_gradient(model, x, y)
    x_hat, y_hat = dlg_attack(model, target_grads)
    print("label recovered:", y_hat.item(), "truth:", y.item())
    print("PSNR:", _psnr(x_hat.clamp(0, 1), x))

if __name__ == "__main__":
    main()
```

On a single-sample CIFAR-10 image with this toy model the attack
reaches PSNR ≈ 20–30 dB — visually recognisable. This is the
baseline against which defences are compared.

### 2.2 Defence 1 — larger local batch

Replace the single-sample gradient with a batch gradient:

```python
# run_defence_batch.py — same as run_attack.py with:
from torch.utils.data import DataLoader, Subset
loader = DataLoader(Subset(ds, range(64)), batch_size=64)
x, y = next(iter(loader))
```

Rerun; the PSNR drops sharply because the gradient is now an average
over 64 samples and the inversion has to disambiguate which combination
of rows produced it. DLG typically fails above batch ≈ 8 on this
architecture (Zhu et al. 2019 §4.2; Geiping et al. 2020 shows this
is model-dependent).

### 2.3 Defence 2 — multiple local steps (FedAvg)

```python
# run_defence_fedavg.py
model = build_victim_model()
loader = DataLoader(ds_small, batch_size=8)
opt = optim.SGD(model.parameters(), lr=0.1)

start_state = {k: v.detach().clone() for k, v in model.state_dict().items()}
model.train()
for _ in range(5):                             # 5 local steps
    for xb, yb in loader:
        opt.zero_grad()
        loss = nn.CrossEntropyLoss()(model(xb), yb)
        loss.backward()
        opt.step()

delta = {k: (v - start_state[k]) for k, v in model.state_dict().items()}
# Attacker now sees `delta`, not a single gradient. DLG must back out
# which weight-trajectory produced `delta` — a much harder problem.
```

Even without any cryptography, DLG does not transparently invert a
FedAvg *delta* that accumulates multiple SGD steps on a batched
dataset. This is one reason "plain FedAvg" is often framed as
*more* private than "share raw gradients" — though this is not a
substitute for secure aggregation or DP.

### 2.4 Defence 3 — Gaussian noise + clipping (DP-FL)

```python
# run_defence_dp.py
C, sigma = 1.0, 1.0
grads = compute_gradient(model, x, y)
flat = torch.cat([g.view(-1) for g in grads])
norm = flat.norm()
scale = (C / max(norm.item(), 1e-8)).__min__(1.0) if False else min(C / max(norm.item(), 1e-8), 1.0)
grads = [g * scale for g in grads]
grads = [g + torch.randn_like(g) * sigma * C for g in grads]
# Then re-run the attack with these as target_grads.
```

Rerun the attack with the noised, clipped gradient; PSNR collapses
toward the noise floor. The exact ε of this one step is small (one
round, q = 1), but the point of the drill is to show that *any*
calibrated noise with a correct clip destroys the reconstruction.

### 2.5 Defence 4 — secure aggregation (sum of many clients)

Secure aggregation (see [[exercise-04]]) hides every single client's
delta from the server. The server only sees `Σ y_i`. The attacker
then has to invert a *sum* of per-client gradients, with no per-
client disaggregation available. Even without DP this blocks DLG
cleanly on ≥ 8 honest participants under the Bonawitz threat model:
no single client's `y_i` is ever revealed.

```python
# run_defence_secagg.py (pedagogical)
grad_sum = None
for cid in range(8):
    xc, yc = ds[cid]
    g = compute_gradient(model, xc.unsqueeze(0), torch.tensor([yc]))
    grad_sum = g if grad_sum is None else [a + b for a, b in zip(grad_sum, g)]
# Attacker sees grad_sum. DLG must now recover 8 inputs from one summed
# gradient — strictly harder, and usually impossible on reasonable models.
```

### 2.6 Result table (template the learner fills in)

| Setting | Batch | Local steps | σ (DP) | PSNR ↓ | Label match ↓ |
|---|---|---|---|---|---|
| Baseline (1 sample, 1 step) | 1 | 1 | 0 | ~25 dB | 1/1 |
| Defence 1: batch 64 | 64 | 1 | 0 | ~10 dB | Chance |
| Defence 2: FedAvg (5 steps, batch 8) | 8 | 5 | 0 | ~8 dB | Chance |
| Defence 3: DP (C=1, σ=1, 1 step) | 1 | 1 | 1 | ~5 dB | Chance |
| Defence 4: SecAgg (sum of 8 clients, 1 step each) | 1/client | 1 | 0 | ~5 dB | Chance |
| Defence 2+4 combined | 8 | 5 | 0 | floor | Chance |

Exact PSNR numbers are architecture-dependent. The pedagogical
pattern is: *baseline is visually recognisable; every defence
collapses PSNR to a level where the reconstruction is unusable*.

### 2.7 Threat-model mapping (for the writeup)

- **OWASP ML Top 10** — "sensitive information disclosure via model
  gradients" falls under the OWASP ML-01 / ML-02 category of training-
  data exposure via model behaviour. In the FL deployment context,
  the surfaced finding is: *per-client gradients are PII-equivalent
  for the data they were computed on*.
- **MITRE ATLAS** — relevant techniques for this attack:
  - `AML.T0024 Exfiltration via ML Inference API` (if the gradient
    is observable via an exposed update endpoint);
  - `AML.T0025 Exfiltration via Cyber Means` (if the attacker
    passively captures round traffic);
  - `AML.T0020 Data from Local System` (on a compromised client).
- **NIST AI RMF** — Measure/Manage: the organisation must *measure*
  gradient leakage before claiming the FL deployment protects
  training data, and *manage* it by composing secure aggregation,
  DP-FL, and defensive FedAvg settings.

## 3. Validation steps

1. **Attack baseline reproduces.** Run `run_attack.py` with a fixed
   seed; the reconstruction visually resembles the input and PSNR
   is in the 15–30 dB range for the toy architecture.
2. **Batch defence is monotone.** Sweep batch size `{1, 4, 8, 16, 64}`
   and record PSNR; PSNR should strictly decrease with batch size
   on the same seed.
3. **DP defence collapses attack.** For `σ ∈ {0.1, 0.5, 1.0, 2.0}`
   with `C = 1`, record PSNR. For `σ ≥ 0.5` the reconstruction
   should look like noise; for `σ = 0` it should match the baseline.
4. **FedAvg defence collapses attack.** Run with `local_steps ∈
   {1, 2, 5}`; the delta-attack should fail cleanly at `local_steps ≥ 2`
   in the toy setup.
5. **Secure aggregation defence collapses attack.** Attack the sum
   of `k` gradients for `k ∈ {1, 2, 4, 8}`; PSNR should drop with
   `k`. For `k ≥ 8` on the toy model the reconstruction is visually
   indistinguishable from noise.
6. **No cherry-picking.** Report mean PSNR over at least 5 different
   target images per setting, not a single best example. The point
   is a *distribution* of reconstruction quality.
7. **Defence combinations.** Run DP + FedAvg + SecAgg together; this
   should produce PSNR at or below the DP-only floor. If a reviewer
   asks "which defence actually did the work," the sweep above
   answers it.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Baseline DLG/Inverting-Gradients attack recovers a visually recognisable image on a known model | 2 | PSNR must be in the published range |
| At least three defences implemented (batch, FedAvg steps, DP, SecAgg — pick ≥ 3) | 2 | Each defence re-runs the attack |
| Result table has at least 5 PSNR rows + reconstruction image grid | 2 | Mean over ≥ 5 targets |
| OWASP + ATLAS + NIST AI RMF mapping stated in the writeup | 1 | Three frameworks named with specific entries |
| Cross-links to [[exercise-04]] (secure aggregation) and [[exercise-05]] (DP-FL) | 1 | Reviewer should be able to navigate the module |
| Writeup says explicitly what the attack *does not* require (no sample; attacker only needs model + gradient) | 1 | This is the finding |
| Writeup names the threat model the attack belongs to (honest-but-curious server / passive observer) | 1 | — |

Automatic failures:

- Reporting "defence works" with a single target image.
- Reporting DP as a defence without also reporting the (ε, δ) and
  accountant used — the reader must be able to tell whether the
  defence came at zero privacy cost.
- Treating multi-step FedAvg as a *privacy guarantee*. It is an
  attack-mitigation on current inversion attacks, not a bound. The
  writeup must say so.
- Running the attack against a model the attacker does not actually
  have access to in the stated threat model (e.g., a proprietary
  model) and reporting that as evidence of real-world risk.

## 5. Common mistakes

- **Claiming FedAvg "solves" leakage.** DLG and Inverting-Gradients
  publications show the attack works against single-step updates and
  small batches. FedAvg with many local steps and non-trivial batch
  size *empirically* breaks naive gradient inversion — but newer
  attacks (Geiping et al. 2020, Yin et al. 2021) adapt. Only secure
  aggregation and DP-FL have formal guarantees.
- **Reporting PSNR without an image grid.** PSNR is a summary; a
  reviewer wants to see one strong reconstruction (baseline) and
  three noisy ones (defences) to judge whether the attack is *real*
  in their system.
- **Choosing the easy architecture and generalising.** The toy CNN
  from Zhu et al. (2019) is intentionally attack-friendly. On
  ResNet-50, baseline DLG often fails even without defences. State
  the architecture used and resist "my model is safe because DLG
  didn't work on it."
- **Attacking the wrong gradient.** The attack needs the *loss
  gradient* with respect to parameters; grabbing `param.data -
  param_before.data` (a weight delta) and feeding it to DLG does not
  work directly without the step-size inverse. Either run single-step
  DLG on the true gradient, or run Inverting-Gradients on the
  weight-delta with the correct formulation.
- **Omitting the label-recovery step.** Zhu et al.'s DLG jointly
  recovers `(x, y)`; some implementations only recover `x` with a
  guessed `y`. That is a different attack; name it.
- **Mislabelling the threat model.** DLG assumes the attacker knows
  the model weights. In an honest-but-curious server scenario, this
  is true — the server maintains the global model. In a client-side
  threat model, it is only true if the attacker has a client role.
  The writeup must say which.
- **Skipping the combination test.** DP + SecAgg + FedAvg is the
  recommended real-world stack. A writeup that tests each defence
  alone but not together does not show the *production* story.
- **Reporting the attack as "proof FL is unsafe."** The attack is a
  *measurement*. It proves per-client gradients carry recoverable
  information in specific settings. The policy conclusion depends on
  what defences the deployment layers on, which is the whole point
  of the exercise.

## 6. References

- Zhu, L.; Liu, Z.; Han, S. *Deep Leakage from Gradients.* NeurIPS 2019.
  <https://arxiv.org/abs/1906.08935>
- Geiping, J.; Bauermeister, H.; Dröge, H.; Moeller, M. *Inverting
  Gradients — How easy is it to break privacy in federated learning?*
  NeurIPS 2020. <https://arxiv.org/abs/2003.14053>
- Yin, H. et al. *See through Gradients: Image Batch Recovery via
  GradInversion.* CVPR 2021. <https://arxiv.org/abs/2104.07586>
- Zhao, B.; Mopuri, K. R.; Bilen, H. *iDLG: Improved Deep Leakage
  from Gradients.* arXiv:2001.02610.
  <https://arxiv.org/abs/2001.02610>
- McMahan, H. B.; Ramage, D.; Talwar, K.; Zhang, L. *Learning
  Differentially Private Recurrent Language Models.* ICLR 2018.
  <https://arxiv.org/abs/1710.06963>
- Bonawitz, K. et al. *Practical Secure Aggregation for
  Privacy-Preserving Machine Learning.* ACM CCS 2017.
  <https://eprint.iacr.org/2017/281>
- Kairouz, P.; McMahan, H. B.; et al. *Advances and Open Problems in
  Federated Learning.* FnTML 2021. <https://arxiv.org/abs/1912.04977>
- NIST Special Publication 800-226, *Guidelines for Evaluating
  Differential Privacy Guarantees* (final, 2025).
  <https://csrc.nist.gov/pubs/sp/800/226/final>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
