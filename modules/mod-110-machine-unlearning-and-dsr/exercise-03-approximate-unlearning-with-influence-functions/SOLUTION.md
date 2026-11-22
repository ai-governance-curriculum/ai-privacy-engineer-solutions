# Exercise 03 — Approximate Unlearning with Influence Functions — Reference Solution

## 1. Solution overview

This is an **implementation exercise** on **approximate** unlearning
via influence functions. Where SISA ([[exercise-02]]) rewrites
training so that exact erasure is cheap, influence-function
unlearning takes a *trained* model and *moves its parameters* to
approximate the parameters that would have been obtained had the
record been absent. The cost is a single Hessian-vector-product
sequence, not a retrain. The trade-off is strict: the approximation
is only reliable under convex or near-convex losses, with a
well-conditioned Hessian, and modest-size parameter counts; deep
non-convex models get only a coarse approximation and need the
claims in the audit trail scaled accordingly.

The deliverable has three parts:

1. **A reference implementation** of the classical Koh–Liang
   (ICML 2017) influence-function update: parameter shift
   `Δθ = + (1/n) · H_θ^{-1} · ∇_θ ℓ(z, θ)` where `z` is the record
   to forget. Hessian-vector products are computed via LiSSA
   (Agarwal–Bullins–Hazan, 2017), with an explicit damping term
   for numerical stability.
2. **A controlled benchmark**: on a small convex model
   (logistic regression on a tabular dataset), compare the
   influence-function-unlearned model against (a) a full retrain
   that omits the record and (b) the untouched original. Report
   parameter distance, predictive-distribution distance (KL /
   total variation), and a membership-inference attack success
   rate as the operational quality metric ([[mod-105]]).
3. **A scope + limits note** stating the regime in which the
   approximation is defensible and the regime in which it is not,
   including the Basu–Pope–Feizi (ICLR 2021) warning that
   influence functions on deep networks are fragile.

The implementation lines up with the Koh–Liang (2017) and
Guo–Goldstein–Hannun–van der Maaten (ICML 2020, "Certified Data
Removal") formulations and keeps the governance framing (NIST AI
RMF, MITRE ATLAS, OWASP ML Top 10) identical to the rest of the
module.

## 2. Worked implementation

### 2.1 Construction recap

The empirical risk minimiser on training set `D = {z_1, …, z_n}` is

        θ* = argmin_θ (1/n) Σ_i ℓ(z_i, θ).

Removing a single point `z_k` and retraining gives
`θ*_{-k} = argmin_θ (1/n) Σ_{i ≠ k} ℓ(z_i, θ)`. The classical
influence function, first-order Taylor-expanded around `θ*`, is

        I_up,params(z_k)  =  − H_{θ*}^{-1}  · ∇_θ ℓ(z_k, θ*),

so that to first order

        θ*_{-k}  ≈  θ*  +  (1/n) · H_{θ*}^{-1} · ∇_θ ℓ(z_k, θ*).

The unlearning procedure is literally to apply that update, with
`H_θ` the empirical Hessian of the loss at `θ*`. The two
computational problems are

1. **Hessian-vector products (HVPs)** — use autograd twice (Pearlmutter
   trick) so you never form the Hessian.
2. **Inverse-HVP** — iterate LiSSA. With damping `λ`, let
   `H̃ = (1 − λ) I + λ H`; LiSSA computes `H̃^{-1} v` as a truncated
   Neumann series:

        u_0 = v,  u_t = v + (I − H̃) u_{t−1}.

For well-conditioned convex problems LiSSA converges in a few hundred
HVPs; the number of HVPs is the dominant cost and the main thing to
instrument.

### 2.2 Reference implementation (self-contained)

```python
# influence_unlearn.py — pedagogical reference. Not production: see §5.
from __future__ import annotations

import math
from dataclasses import dataclass
from typing import Callable, Iterable, List, Tuple

import torch
from torch import nn


# ---------- loss plumbing --------------------------------------------------
def params_as_vec(model: nn.Module) -> torch.Tensor:
    return torch.cat([p.detach().view(-1) for p in model.parameters()])


def set_params_from_vec(model: nn.Module, vec: torch.Tensor) -> None:
    offset = 0
    for p in model.parameters():
        numel = p.numel()
        p.data.copy_(vec[offset:offset + numel].view_as(p))
        offset += numel


def grad_params(
    loss: torch.Tensor, params: List[torch.Tensor], create_graph: bool
) -> List[torch.Tensor]:
    return list(torch.autograd.grad(loss, params, create_graph=create_graph))


def flat(tensors: List[torch.Tensor]) -> torch.Tensor:
    return torch.cat([t.contiguous().view(-1) for t in tensors])


# ---------- Hessian-vector product ----------------------------------------
def hvp(
    loss_fn: Callable[[], torch.Tensor],
    params: List[torch.Tensor],
    v: torch.Tensor,
) -> torch.Tensor:
    """Pearlmutter HVP: (∇² loss) · v, flattened."""
    loss = loss_fn()
    g = flat(grad_params(loss, params, create_graph=True))
    gv = (g * v).sum()
    hv = flat(grad_params(gv, params, create_graph=False))
    return hv


# ---------- LiSSA inverse-HVP ----------------------------------------------
@dataclass
class LissaConfig:
    damping: float = 0.01          # λ: avoid ill-conditioned H
    scale: float = 25.0            # divide H by `scale` so Neumann converges
    depth: int = 250               # truncation depth
    repeats: int = 3               # average over independent runs


def inverse_hvp_lissa(
    full_loss_fn: Callable[[], torch.Tensor],
    params: List[torch.Tensor],
    v: torch.Tensor,
    cfg: LissaConfig = LissaConfig(),
) -> torch.Tensor:
    """Compute (H + λI)^{-1} v by LiSSA Neumann series, averaged over repeats."""
    est = torch.zeros_like(v)
    for _ in range(cfg.repeats):
        u = v.clone()
        for _ in range(cfg.depth):
            h_u = hvp(full_loss_fn, params, u)
            # u ← v + (I − (H + λI)/scale) u    [Neumann iteration on H/scale]
            u = v + (1 - cfg.damping) * u - h_u / cfg.scale
        est = est + u / cfg.scale
    return est / cfg.repeats


# ---------- influence-function unlearning ---------------------------------
def unlearn_influence(
    *,
    model: nn.Module,
    full_loss_fn: Callable[[], torch.Tensor],          # mean loss on D
    forget_loss_fn: Callable[[], torch.Tensor],        # loss on z_k alone
    n_train: int,
    lissa_cfg: LissaConfig = LissaConfig(),
) -> dict:
    """Shift parameters by (1/n) · H^{-1} · ∇ℓ(z_k, θ*)."""
    params = [p for p in model.parameters() if p.requires_grad]
    grad_zk = flat(grad_params(forget_loss_fn(), params, create_graph=False)).detach()

    inv = inverse_hvp_lissa(full_loss_fn, params, grad_zk, lissa_cfg)
    delta = inv / n_train

    theta = params_as_vec(model) + delta
    set_params_from_vec(model, theta)
    return {
        "grad_norm": float(grad_zk.norm()),
        "delta_norm": float(delta.norm()),
        "lissa_depth": lissa_cfg.depth,
        "lissa_repeats": lissa_cfg.repeats,
    }
```

The `full_loss_fn` closure must return the mean loss across the
training set at the current parameters; `forget_loss_fn` must return
the loss on the single forget record. Both are closures over
dataloaders rather than globals so the caller can swap datasets for
the benchmark.

### 2.3 Benchmark harness

The harness trains three models on the same data and compares them:
`M_full` (trained on `D`), `M_minus_k` (trained on `D \ {z_k}`,
gold reference), and `M_unlearned` (produced by applying the
influence update to `M_full`).

```python
# bench.py — convex-loss regime for a reliable influence approximation.
import copy
import torch
from torch import nn

from influence_unlearn import (
    unlearn_influence, LissaConfig,
)


def logistic_regression(dim: int, num_classes: int) -> nn.Module:
    return nn.Sequential(nn.Linear(dim, num_classes))


def train_to_convergence(
    model: nn.Module, X: torch.Tensor, y: torch.Tensor,
    *, l2: float = 1e-3, iters: int = 1000, lr: float = 0.5,
):
    opt = torch.optim.LBFGS(model.parameters(), lr=lr, max_iter=iters, tolerance_grad=1e-9)
    loss_fn = nn.CrossEntropyLoss()

    def closure():
        opt.zero_grad()
        logits = model(X)
        loss = loss_fn(logits, y) + l2 * sum((p ** 2).sum() for p in model.parameters())
        loss.backward()
        return loss

    opt.step(closure)
    return model


def kl_between(model_a: nn.Module, model_b: nn.Module, X: torch.Tensor) -> float:
    with torch.no_grad():
        pa = torch.softmax(model_a(X), dim=-1)
        pb = torch.softmax(model_b(X), dim=-1) + 1e-12
        return float((pa * (pa.clamp_min(1e-12).log() - pb.log())).sum(dim=-1).mean())


def bench(X: torch.Tensor, y: torch.Tensor, k: int):
    dim, C = X.size(1), int(y.max()) + 1

    # --- M_full ---
    m_full = train_to_convergence(logistic_regression(dim, C), X, y)

    # --- M_minus_k (gold) ---
    mask = torch.ones(X.size(0), dtype=torch.bool); mask[k] = False
    m_minus = train_to_convergence(logistic_regression(dim, C), X[mask], y[mask])

    # --- M_unlearned ---
    m_unl = copy.deepcopy(m_full)
    l2 = 1e-3

    def full_loss():
        logits = m_unl(X)
        return (nn.CrossEntropyLoss()(logits, y)
                + l2 * sum((p ** 2).sum() for p in m_unl.parameters()))

    def forget_loss():
        logits = m_unl(X[k:k+1])
        return nn.CrossEntropyLoss()(logits, y[k:k+1])

    report = unlearn_influence(
        model=m_unl,
        full_loss_fn=full_loss,
        forget_loss_fn=forget_loss,
        n_train=X.size(0),
        lissa_cfg=LissaConfig(damping=0.01, scale=25.0, depth=250, repeats=3),
    )
    print(report)
    print("‖θ_full − θ_minus‖₂  =", float((params_vec(m_full) - params_vec(m_minus)).norm()))
    print("‖θ_unl  − θ_minus‖₂  =", float((params_vec(m_unl)  - params_vec(m_minus)).norm()))
    print("KL(M_unl || M_minus) =", kl_between(m_unl, m_minus, X))
    print("KL(M_full|| M_minus) =", kl_between(m_full, m_minus, X))


def params_vec(m):
    return torch.cat([p.detach().view(-1) for p in m.parameters()])
```

Expected results, on logistic regression with `n = 2000` and L2
regularisation `λ = 10⁻³`:

- `‖θ_full − θ_minus‖₂` is small but non-zero (the baseline drift
  from removing one of 2000 points).
- `‖θ_unl − θ_minus‖₂` is roughly an order of magnitude smaller
  than `‖θ_full − θ_minus‖₂`. The ratio is the figure of merit —
  it says how much of the "retrain gap" the influence update
  closes.
- `KL(M_unl || M_minus)` should be an order of magnitude smaller
  than `KL(M_full || M_minus)`. If it is not, LiSSA has not
  converged — increase `depth` or `repeats`.

### 2.4 When the approximation is defensible

The Taylor expansion assumes the loss is twice differentiable and
the empirical minimiser is a true minimiser (not just a stationary
point). The regimes that satisfy this are:

- Strictly convex losses with explicit L2 regularisation (logistic
  regression, linear regression, ridge, SVMs with smooth loss).
- Small- to medium-parameter models where LiSSA converges within a
  tractable `depth × repeats` budget.
- Single-record or small-batch forgetting; multi-record forgetting
  requires either a sum of influences (first-order approximation)
  or iterating the procedure (Guo et al. 2020).

The regimes where it is **not** defensible, per Basu–Pope–Feizi
(ICLR 2021):

- Deep non-convex networks: the Hessian has many near-zero
  eigenvalues; LiSSA diverges; even when it converges, the
  first-order approximation fails to capture the retraining
  trajectory. Expect noisy influence values and audit certificates
  that cannot be defended.
- Models without explicit regularisation: `H` is singular; damping
  masks the issue without fixing it.
- Early-stopped models where `θ*` is not an actual minimiser.

For the deep-network regime, see [[exercise-04]] (gradient-ascent
unlearning for LLMs) and [[exercise-02]] (SISA for exact erasure).

### 2.5 Certified removal (optional extension)

Guo et al. 2020 extend the influence-function update by adding a
Gaussian noise term calibrated so that the resulting model is
indistinguishable from a retrained model under a chosen
`(ε, δ)`-differential-privacy budget — a *certified data removal*.
The noise scale is set by the Lipschitz constant of the loss and
the smallest Hessian eigenvalue. This is the construction the
privacy team cites when an Art. 17 refusal needs a positive
guarantee instead of "best-effort approximation."

## 3. Validation steps

1. **Baseline closure test.** On the convex benchmark,
   `‖θ_unl − θ_minus‖₂ / ‖θ_full − θ_minus‖₂` should be
   well below 1 — typically ≤ 0.2 on a well-regularised LR
   problem. A ratio near 1 means the influence update is a no-op;
   LiSSA has not converged.
2. **HVP-gradient sanity.** Compute `hvp(loss, params, θ)`;
   with `v = θ`, the result should equal `H · θ`. Compare to a
   finite-difference estimate on a small model. Discrepancy >
   10⁻³ means the autograd graph is leaking.
3. **LiSSA convergence curve.** Plot `‖u_t − u_{t−1}‖` across LiSSA
   iterations; the curve should fall at least an order of magnitude
   before truncation. If it is flat, `scale` is too small (H is
   being scaled into divergence) or `damping` is too high (bias is
   dominating).
4. **Membership-inference drop.** Use a trained black-box MIA
   ([[mod-105]]) against the forget record. Attack success rate on
   `M_full` should be substantially above chance; on `M_unl` and
   `M_minus_k` it should be near chance and roughly matched. If
   `M_unl` retains high MIA success, the forgetting is not
   operational and the audit certificate cannot cite the removal.
5. **Non-convex blow-up.** Repeat the benchmark on a small MLP
   (two hidden layers, ReLU, no regularisation). The ratio
   `‖θ_unl − θ_minus‖₂ / ‖θ_full − θ_minus‖₂` typically does not
   drop below 1 — this is the Basu–Pope–Feizi finding and must
   be reported in the scope note (§2.4).
6. **Determinism.** Fix `torch.manual_seed`. Two `unlearn_influence`
   invocations on the same `M_full` must produce the same `Δθ` up to
   LiSSA stochasticity; if they diverge wildly, the RNG is leaking
   via dataloader shuffling.
7. **Audit certificate.** The output `report` dict must contain
   `grad_norm`, `delta_norm`, LiSSA depth, LiSSA repeats, loss type,
   and regularisation. These fields feed the audit trail
   ([[exercise-07]]).

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Pearlmutter HVP implemented correctly (double-backprop, no explicit Hessian) | 2 | §2.2 |
| LiSSA inverse-HVP with damping and convergence diagnostics | 2 | §2.2, §3.3 |
| Influence update applied as `θ ← θ + (1/n) · H^{-1} · ∇ℓ(z_k)` | 1 | sign and `1/n` scaling |
| Benchmark compares `M_full`, `M_minus_k`, `M_unl` on parameter distance and KL | 2 | §2.3 |
| MIA-based operational metric reported | 1 | §3.4 |
| Scope / limits note cites Basu–Pope–Feizi for non-convex caveats | 1 | §2.4 |
| Audit-certificate dict populated and linked into [[exercise-07]] | 1 | — |

Automatic failures:

- Claiming the method produces *exact* unlearning (it does not).
- Reporting only `‖Δθ‖` without comparing to the retrain gold
  (`M_minus_k`): the whole point is that the retrain gap is closed.
- Using the method on a deep network and omitting the
  Basu–Pope–Feizi caveat.
- Forgetting the `1/n` factor (doubles or halves the update
  magnitude; invalidates the approximation).
- Omitting L2 regularisation on the baseline LR problem, then being
  surprised that `H` is singular.

## 5. Common mistakes

- **Treating the approximation as a guarantee.** The classical
  influence function is a first-order Taylor expansion. Even when
  LiSSA converges perfectly, it does not reproduce `θ*_{-k}` — only
  its first-order Taylor neighbour. For an Art. 17 certificate,
  this is a defensible method only when paired with an MIA check
  (§3.4) and a scope statement.
- **LiSSA diverging on an unscaled Hessian.** The Neumann series
  for `H^{-1}` converges only when `‖I − H‖ < 1`. If `H`'s top
  eigenvalue > 2, the series blows up. The standard fix is to
  scale `H` by a constant `s` so that `H/s` has top eigenvalue
  < 1; the inverse-HVP is then `(1/s) · (H/s)^{-1} · v`. Forgetting
  the scaling is the single most common LiSSA bug.
- **LiSSA over-damped.** Large `damping` (e.g., 0.1+) biases the
  inverse-HVP toward the identity — the update looks like plain
  gradient ascent on the forget loss, not an influence correction.
  The MIA test will not drop, and the method is secretly running
  the [[exercise-04]] gradient-ascent recipe without the LLM
  safeguards.
- **Autograd graph leaks.** `create_graph=True` is needed for the
  Hessian step but must *not* be held across multiple LiSSA steps.
  If you don't `.detach()` between HVPs, memory explodes and the
  double-backprop path double-counts.
- **Forgetting the regularisation term in the loss closure.** If
  the training loss includes L2 but the `full_loss_fn` does not,
  LiSSA solves a different system from the one minimised during
  training and the influence update will be badly wrong.
- **Using the mean loss where the sum is expected (or vice versa).**
  Koh–Liang's derivation uses `(1/n) Σ ℓ`; some papers use `Σ ℓ`.
  The `1/n` factor in `θ ← θ + (1/n) · H^{-1} · ∇ℓ(z_k)` is tied to
  the mean-loss formulation. Mixing them scales the update by `n`.
- **Multi-record unlearning by summing updates.** The sum is a
  first-order approximation that breaks when the Hessian drifts.
  Iterate the procedure (apply update, recompute Hessian, repeat)
  or move to SISA.
- **Ignoring the inference-time attack surface.** After unlearning,
  the model's predictions on neighbours of `z_k` can still leak
  that `z_k` was unusual. ATLAS `AML.T0024` membership inference
  attacks are still worth running post-unlearn as the operational
  check.

## 6. References

- Koh, P. W.; Liang, P. *Understanding Black-box Predictions via
  Influence Functions.* ICML 2017.
  <https://arxiv.org/abs/1703.04730> — the formulation implemented
  here.
- Agarwal, N.; Bullins, B.; Hazan, E. *Second-Order Stochastic
  Optimization for Machine Learning in Linear Time.* JMLR 2017.
  <https://jmlr.org/papers/v18/16-491.html> — LiSSA, the
  inverse-HVP method used in §2.2.
- Basu, S.; Pope, P.; Feizi, S. *Influence Functions in Deep
  Learning Are Fragile.* ICLR 2021.
  <https://arxiv.org/abs/2006.14651> — the limit of the method on
  deep non-convex networks.
- Guo, C.; Goldstein, T.; Hannun, A.; van der Maaten, L.
  *Certified Data Removal from Machine Learning Models.*
  ICML 2020. <https://arxiv.org/abs/1911.03030> — the certified
  extension (§2.5).
- Bourtoule, L. et al. *Machine Unlearning.* IEEE S&P 2021.
  <https://arxiv.org/abs/1912.03817> — the exact alternative
  implemented in [[exercise-02]].
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
