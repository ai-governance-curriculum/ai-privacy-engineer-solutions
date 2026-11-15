# Exercise 02 — DP-SGD Training with Opacus — Reference Solution

## 1. Solution overview

This is an **implementation exercise**. The learner must take a standard
PyTorch training loop and convert it to a DP-SGD training loop using
Opacus, then demonstrate that (a) the produced model has a defensible
(ε, δ) guarantee at a stated unit of privacy, (b) the pipeline uses
Poisson subsampling, per-sample gradient clipping, and Gaussian noise
addition as DP-SGD requires, and (c) the release manifest captures what
a reviewer needs to reproduce the bound.

The assessable output is three things:

1. A **training script** that wraps the model, optimiser, and data loader
   in Opacus's `PrivacyEngine` and runs end-to-end.
2. A **privacy report** produced by the Opacus accountant at the end of
   training, including ε, δ, the clip norm, the noise multiplier, the
   sample rate, and the number of steps.
3. A **manifest** suitable for inclusion in a release package (unit of
   privacy, neighbouring relation, accountant used, and any deviations
   from the defaults).

The implementation below follows the Opacus 1.x API. Where a design
choice has multiple correct answers (e.g., clip norm, choice of
accountant), the solution picks one and names the alternatives.

## 2. Worked implementation

### 2.1 Minimal DP-SGD training script

```python
# train_dp.py
import torch
from torch import nn, optim
from torch.utils.data import DataLoader
from opacus import PrivacyEngine
from opacus.utils.batch_memory_manager import BatchMemoryManager

# --- 1. Hyperparameters tied to the privacy claim ---
TARGET_EPSILON = 8.0          # user-level, reported with delta below
TARGET_DELTA   = 1e-5         # delta << 1/N
MAX_GRAD_NORM  = 1.0          # per-sample L2 clip
EPOCHS         = 10
LOGICAL_BATCH  = 512          # sampling rate = LOGICAL_BATCH / N
PHYSICAL_BATCH = 64           # fits in GPU memory; BMM accumulates

def build_model() -> nn.Module:
    # Replace with the target architecture. Opacus validates the model;
    # layers without a per-sample-gradient hook (e.g. BatchNorm) must be
    # replaced (GroupNorm) or the engine will raise before training.
    return nn.Sequential(
        nn.Flatten(),
        nn.Linear(784, 256),
        nn.GroupNorm(32, 256),
        nn.ReLU(),
        nn.Linear(256, 10),
    )

def train(train_dataset, val_dataset, device: str = "cuda") -> dict:
    model = build_model().to(device)
    optimizer = optim.SGD(model.parameters(), lr=0.5, momentum=0.0)
    criterion = nn.CrossEntropyLoss()

    # Opacus requires a DataLoader that yields Poisson-subsampled batches.
    # make_private_with_epsilon rewrites the loader to use Poisson sampling.
    data_loader = DataLoader(
        train_dataset,
        batch_size=LOGICAL_BATCH,
        shuffle=True,
        num_workers=2,
    )

    privacy_engine = PrivacyEngine(accountant="prv")
    model, optimizer, data_loader = privacy_engine.make_private_with_epsilon(
        module=model,
        optimizer=optimizer,
        data_loader=data_loader,
        target_epsilon=TARGET_EPSILON,
        target_delta=TARGET_DELTA,
        epochs=EPOCHS,
        max_grad_norm=MAX_GRAD_NORM,
    )

    model.train()
    for epoch in range(EPOCHS):
        # BatchMemoryManager breaks the logical Poisson batch into physical
        # micro-batches without altering the privacy accounting.
        with BatchMemoryManager(
            data_loader=data_loader,
            max_physical_batch_size=PHYSICAL_BATCH,
            optimizer=optimizer,
        ) as memory_safe_loader:
            for batch in memory_safe_loader:
                x, y = batch[0].to(device), batch[1].to(device)
                optimizer.zero_grad()
                logits = model(x)
                loss = criterion(logits, y)
                loss.backward()
                optimizer.step()

        eps = privacy_engine.get_epsilon(delta=TARGET_DELTA)
        print(f"epoch={epoch+1} eps={eps:.2f} delta={TARGET_DELTA}")

    return {
        "epsilon": privacy_engine.get_epsilon(delta=TARGET_DELTA),
        "delta": TARGET_DELTA,
        "max_grad_norm": MAX_GRAD_NORM,
        "noise_multiplier": optimizer.noise_multiplier,
        "sample_rate": data_loader.sample_rate,
        "steps": privacy_engine.accountant.history,
        "accountant": "prv",
    }
```

Key points the reviewer verifies:

- **`PrivacyEngine(accountant="prv")`** picks the Privacy Random Variable
  accountant, which is typically tighter than RDP at the same noise. The
  alternatives are `"rdp"` (robust default) and `"gdp"` (Gaussian DP).
- **`make_private_with_epsilon(...)`** reparameterises `noise_multiplier`
  from a *target ε* at fixed (epochs, δ, sample_rate). The learner
  commits to the privacy claim up front; Opacus derives the σ.
- **`BatchMemoryManager`** allows large *logical* Poisson batches (which
  are what the accountant sees) while backprop runs on smaller *physical*
  micro-batches. It does **not** alter the accounting.
- **`MAX_GRAD_NORM = 1.0`** is the per-sample L2 clip. The DP-SGD proof
  bounds sensitivity through clipping, not through ordinary weight decay.
- **BatchNorm → GroupNorm.** BatchNorm mixes samples across the batch
  statistics and Opacus rejects it. GroupNorm, LayerNorm, or InstanceNorm
  are DP-compatible.

### 2.2 Privacy report produced at training end

```json
{
  "mechanism": "DP-SGD (Abadi et al. 2016), subsampled Gaussian",
  "unit_of_privacy": "user",
  "neighbouring_relation": "add/remove one user (unbounded DP)",
  "epsilon": 8.00,
  "delta": 1e-5,
  "accountant": "PRV (Opacus 1.x)",
  "noise_multiplier": 0.95,
  "max_grad_norm": 1.0,
  "sample_rate": 0.00853,
  "steps": 2346,
  "epochs": 10,
  "library": "opacus==<pin>",
  "seed": 0,
  "dataset_size_N": 60000,
  "delta_vs_N_check": "1e-5 << 1/60000 ≈ 1.67e-5 — within guidance",
  "subsampling": "Poisson (make_private_with_epsilon rewrites DataLoader)"
}
```

*Note:* `noise_multiplier` and `steps` above are illustrative; the actual
values come from the accountant at run time. Record the version pin for
Opacus — accountant conversions are library-versioned.

### 2.3 Release manifest fields (minimum)

| Field | Value source |
|---|---|
| Mechanism | "DP-SGD" + citation |
| Neighbouring relation | Add/remove user, or replace record — must match the data loader's grouping key |
| Unit of privacy | The entity the DataLoader groups on before Poisson sampling |
| ε, δ | Final `get_epsilon(delta)` call |
| Accountant | `PrivacyEngine(accountant=...)` value + library version |
| σ (noise_multiplier) | Taken from the optimizer attribute |
| Clip norm C | `max_grad_norm` passed in |
| Sample rate q | `data_loader.sample_rate` |
| Steps T | Accountant's `history` length |
| Dataset size N | From the training set |
| δ ≪ 1/N check | Explicit numeric comparison |
| Seed | For reproducibility of the *training*, not of the privacy bound |

## 3. Validation steps

Run these before declaring the training "done":

1. **Model validator passes.** `ModuleValidator.validate(model, strict=True)`
   returns `[]`. If it raises, replace the offending layers (typical
   culprit: `nn.BatchNorm2d`).
2. **Poisson sampling is actually in effect.** After `make_private_*`,
   the data loader class is `DPDataLoader` and `sample_rate` is
   `logical_batch_size / dataset_size`, not `1 / num_batches`.
3. **Per-sample gradients are being computed.** During one step, confirm
   `param.grad_sample` exists for each parameter before the optimizer
   step clips and noises it.
4. **ε grows monotonically across epochs.** Log `get_epsilon(delta)`
   after every epoch; a flat or decreasing curve means the accountant is
   not seeing the steps — usually a loader-reset bug.
5. **Reproducibility.** Rerun with the same seed and library version; ε
   must be bit-identical and loss trajectory very close (exact-equal if
   determinism flags are set).
6. **Negative test — turn noise off.** Set `noise_multiplier=0` in a
   throwaway run; ε should diverge (the accountant will raise or return
   infinity). If it returns a finite small ε, the accountant is
   misconfigured.
7. **δ sanity.** `delta_vs_N_check` field in the manifest must show
   δ < 1/N; otherwise the (ε, δ) claim is below the NIST SP 800-226
   recommendation.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Poisson sampling in use (DPDataLoader, correct `sample_rate`) | 2 | Uniform shuffling fails |
| Per-sample clipping with explicit `max_grad_norm` | 1 | — |
| `PrivacyEngine` attached with an explicitly chosen accountant | 1 | PRV, RDP, or GDP — stated |
| Model validator passes (no BatchNorm; replaced with GroupNorm/LayerNorm) | 1 | — |
| Target ε committed up front via `make_private_with_epsilon`, not reverse-engineered | 1 | — |
| Manifest records δ, N, and performs δ ≪ 1/N check | 1 | — |
| Manifest names the unit of privacy and the neighbouring relation | 1 | User-level unless justified |
| Final privacy report includes σ, C, q, T, and accountant library version | 2 | Reviewer can reproduce the bound |

Automatic failures:

- Using `batch_size`/uniform shuffling and claiming DP-SGD.
- Reporting only per-step ε and multiplying by steps.
- Omitting δ, or using δ ≥ 1/N.
- Using BatchNorm layers.
- Clipping *after* summing the mini-batch gradient instead of per-sample.

## 5. Common mistakes

- **Leaving BatchNorm in the model.** BatchNorm uses batch statistics,
  which couple samples and break the per-sample-gradient contract.
  Opacus's `ModuleValidator` will flag this; the fix is GroupNorm or
  LayerNorm, not disabling the check.
- **Shuffling instead of Poisson sampling.** Opacus's accountants assume
  Poisson subsampling with rate `q = L / N`. If the loader is a plain
  shuffled `DataLoader`, the subsampling-amplification argument does not
  hold and the reported ε is optimistic.
- **Using `make_private` with `noise_multiplier` tuned by hand and
  reporting a round ε.** Prefer `make_private_with_epsilon`, which picks
  σ from a target ε. Committing to the ε first is the auditable path.
- **Treating `BatchMemoryManager` as a performance knob that changes
  privacy.** It does not. The logical Poisson batch is unchanged; only
  the physical micro-batch size differs.
- **Hand-rolling the clip-then-sum-then-noise step.** Use Opacus's
  optimizer wrapper. Hand-rolled versions often forget to noise after
  summing *per-sample* clipped grads, or clip on the mini-batch
  gradient (which is wrong).
- **Comparing ε across libraries without comparing accountants.** Opacus
  PRV ε and TF Privacy ε at the same σ are usually close but not
  identical. Fix the accountant before comparing numbers.
- **Not pinning library versions.** Accountant implementations evolve;
  the reported ε is the number *that version* produced. Record the pin.
- **Forgetting user-level grouping.** If one user contributes many rows,
  "user-level DP" needs the DataLoader to group by user before sampling,
  not sample rows independently.

## 6. References

- Abadi, M. et al. *Deep Learning with Differential Privacy.* ACM CCS 2016.
  <https://arxiv.org/abs/1607.00133>
- Yousefpour, A. et al. *Opacus: User-Friendly Differential Privacy
  Library in PyTorch.* arXiv:2109.12298.
  <https://arxiv.org/abs/2109.12298>
- Opacus documentation. <https://opacus.ai/>
- Opacus FAQ on accountants and layer compatibility.
  <https://opacus.ai/docs/faq>
- Mironov, I. *Rényi Differential Privacy.* CSF 2017.
  <https://arxiv.org/abs/1702.07476>
- Mironov, I.; Talwar, K.; Zhang, L. *Rényi Differential Privacy of the
  Sampled Gaussian Mechanism.* arXiv:1908.10530.
  <https://arxiv.org/abs/1908.10530>
- Gopi, S.; Lee, Y. T.; Wutschitz, L. *Numerical Composition of
  Differential Privacy.* NeurIPS 2021 (PRV accountant).
  <https://arxiv.org/abs/2106.02848>
- NIST Special Publication 800-226, *Guidelines for Evaluating
  Differential Privacy Guarantees* (final, 2025).
  <https://csrc.nist.gov/pubs/sp/800/226/final>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
