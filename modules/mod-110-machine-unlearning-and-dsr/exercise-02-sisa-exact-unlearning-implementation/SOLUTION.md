# Exercise 02 — SISA Exact Unlearning Implementation — Reference Solution

## 1. Solution overview

This is an **implementation exercise**. The learner implements the
SISA architecture (**S**harded, **I**solated, **S**liced,
**A**ggregated) from Bourtoule et al., *Machine Unlearning*
(IEEE S&P 2021). SISA replaces monolithic training with a
training regime that makes exact unlearning cheap: training data is
split into isolated shards, each shard's training proceeds in
slices with checkpoints between slices, and inference aggregates
across the per-shard models. To unlearn a record, you identify the
one shard that contains it, drop the record, and retrain *from the
earliest slice that saw it* — not from scratch across all data.

The deliverable has three parts:

1. **A SISA framework**: shard assignment, per-shard sliced training
   with checkpoints, aggregation at inference, and an `unlearn(sample_id)`
   entry point that is provably equivalent to a re-sharded full
   retrain minus the forgotten record.
2. **A reference training on a small tabular dataset** (e.g.,
   UCI Adult or MNIST-flattened) with `S` shards and `R` slices per
   shard, parameterised so the exercise can be tuned.
3. **A benchmark** reporting: (a) retrain cost in epochs to unlearn
   a single record under SISA vs. a monolithic baseline, and
   (b) utility delta (accuracy) at a chosen `(S, R)`.

The construction is exact: after `unlearn`, the resulting constituent
model is indistinguishable from one that would have been trained on
"original data minus the forgotten record" with the same seeds. This
is the property that distinguishes SISA from the approximate methods
in [[exercise-03]] and [[exercise-04]], and that lets an organisation
claim Art. 17 fulfilment at the model level with a reproducible
certificate.

The implementation lines up with the Bourtoule et al. construction,
with the standards framing (NIST AI RMF, MITRE ATLAS, OWASP ML Top
10) kept the same as the rest of the module.

## 2. Worked implementation

### 2.1 Construction recap

- **S shards.** The training set `D` is partitioned into `S`
  disjoint shards `D_1, …, D_S`. Assignment is deterministic — a
  hash of a stable record identifier — so that `unlearn(id)` finds
  the right shard in O(1).
- **R slices per shard.** Each shard `D_s` is further split into
  `R` ordered slices `D_{s,1}, …, D_{s,R}`. Slice `r` is trained
  for `E_r` epochs on `D_{s,1} ∪ … ∪ D_{s,r}` starting from the
  checkpoint saved after slice `r − 1`. The checkpoint after each
  slice is retained.
- **Aggregation.** Serving is an ensemble over the `S` per-shard
  models. The paper gives two choices: label-vote (majority of top-1
  predictions) and logit-average (mean of logits). Label-vote is
  the SISA default because it bounds the information leaked by any
  single constituent model.
- **Unlearning.** Given a record `id`:
  1. find shard `s = hash(id) mod S`,
  2. find the earliest slice `r*` containing the record in `D_s`,
  3. restart training *from the checkpoint after slice `r* − 1`*
     with `D_{s,r*}` through `D_{s,R}` recomputed after removing the
     record,
  4. refresh the shard's final weights; the other `S − 1` shards
     are untouched.

The expected retraining work is roughly `(1/S) · ((R − r*) / R)` of
a full retraining run, so with `S = R = 10` a single-record erasure
costs ≤ 1% of a from-scratch run.

### 2.2 Reference implementation

```python
# sisa.py — pedagogical reference. Not production: see §5.
from __future__ import annotations

import copy
import hashlib
import os
import pickle
from dataclasses import dataclass
from pathlib import Path
from typing import Callable, Dict, List, Sequence, Tuple

import numpy as np
import torch
from torch import nn
from torch.utils.data import DataLoader, Subset


# ---------- types ----------------------------------------------------------
Sample = Tuple[np.ndarray, int]          # (features, label)
Record = Tuple[str, Sample]              # (stable_id, sample)


# ---------- deterministic sharding -----------------------------------------
def shard_of(record_id: str, num_shards: int) -> int:
    h = hashlib.sha256(record_id.encode()).digest()
    return int.from_bytes(h[:8], "big") % num_shards


def assign_shards(records: Sequence[Record], num_shards: int) -> List[List[Record]]:
    shards: List[List[Record]] = [[] for _ in range(num_shards)]
    for r in records:
        shards[shard_of(r[0], num_shards)].append(r)
    return shards


def assign_slices(shard: List[Record], num_slices: int) -> List[List[Record]]:
    """Deterministic, record-id-ordered slicing within a shard."""
    sorted_shard = sorted(shard, key=lambda r: r[0])
    out: List[List[Record]] = [[] for _ in range(num_slices)]
    for i, rec in enumerate(sorted_shard):
        out[i % num_slices].append(rec)
    return out


# ---------- storage layout -------------------------------------------------
@dataclass
class SisaPaths:
    root: Path

    def shard_dir(self, s: int) -> Path:  return self.root / f"shard-{s:02d}"
    def slice_ckpt(self, s: int, r: int) -> Path:
        return self.shard_dir(s) / f"after-slice-{r:02d}.pt"
    def final_ckpt(self, s: int) -> Path:
        return self.shard_dir(s) / "final.pt"
    def manifest(self, s: int) -> Path:
        return self.shard_dir(s) / "manifest.pkl"


# ---------- training primitives --------------------------------------------
def _make_loader(recs: Sequence[Record], batch_size: int) -> DataLoader:
    xs = torch.tensor(np.stack([r[1][0] for r in recs], axis=0), dtype=torch.float32)
    ys = torch.tensor([r[1][1] for r in recs], dtype=torch.long)
    ds = torch.utils.data.TensorDataset(xs, ys)
    return DataLoader(ds, batch_size=batch_size, shuffle=True)


def _train_one_slice(
    model: nn.Module,
    recs_through_r: Sequence[Record],
    *,
    epochs_for_slice: int,
    batch_size: int,
    lr: float,
    seed: int,
) -> None:
    torch.manual_seed(seed)
    opt = torch.optim.SGD(model.parameters(), lr=lr, momentum=0.9)
    loss_fn = nn.CrossEntropyLoss()
    loader = _make_loader(recs_through_r, batch_size=batch_size)
    for _ in range(epochs_for_slice):
        for x, y in loader:
            opt.zero_grad()
            loss_fn(model(x), y).backward()
            opt.step()


# ---------- shard training -------------------------------------------------
def train_shard(
    *,
    shard_records: List[List[Record]],         # slice-split shard
    model_factory: Callable[[], nn.Module],
    epochs_per_slice: int,
    batch_size: int,
    lr: float,
    paths: SisaPaths,
    shard_index: int,
    resume_from_slice: int = 0,                # used by unlearn
) -> None:
    """Train one shard through all R slices; snapshot after each slice."""
    paths.shard_dir(shard_index).mkdir(parents=True, exist_ok=True)
    if resume_from_slice == 0:
        model = model_factory()
    else:
        model = model_factory()
        model.load_state_dict(torch.load(paths.slice_ckpt(shard_index, resume_from_slice - 1)))

    accum: List[Record] = []
    for r in range(resume_from_slice):
        accum.extend(shard_records[r])

    R = len(shard_records)
    for r in range(resume_from_slice, R):
        accum.extend(shard_records[r])
        _train_one_slice(
            model,
            accum,
            epochs_for_slice=epochs_per_slice,
            batch_size=batch_size,
            lr=lr,
            seed=1000 * shard_index + r,  # deterministic per-shard-per-slice
        )
        torch.save(model.state_dict(), paths.slice_ckpt(shard_index, r))

    torch.save(model.state_dict(), paths.final_ckpt(shard_index))
    # Manifest records which record_ids appear in which slice.
    manifest = {r: [rec[0] for rec in shard_records[r]] for r in range(R)}
    with open(paths.manifest(shard_index), "wb") as f:
        pickle.dump(manifest, f)


# ---------- aggregation at inference ---------------------------------------
def predict(
    *,
    x: torch.Tensor,
    model_factory: Callable[[], nn.Module],
    paths: SisaPaths,
    num_shards: int,
    mode: str = "label-vote",        # or "logit-average"
) -> torch.Tensor:
    outs = []
    for s in range(num_shards):
        m = model_factory()
        m.load_state_dict(torch.load(paths.final_ckpt(s)))
        m.eval()
        with torch.no_grad():
            outs.append(m(x))

    if mode == "logit-average":
        return torch.stack(outs, dim=0).mean(dim=0).argmax(dim=-1)
    if mode == "label-vote":
        votes = torch.stack([o.argmax(dim=-1) for o in outs], dim=0)   # [S, B]
        B = votes.size(1)
        picks = []
        for b in range(B):
            vals, counts = torch.unique(votes[:, b], return_counts=True)
            picks.append(int(vals[counts.argmax()]))
        return torch.tensor(picks)
    raise ValueError(mode)


# ---------- unlearning -----------------------------------------------------
def unlearn(
    *,
    record_id: str,
    records: Sequence[Record],
    num_shards: int,
    num_slices: int,
    model_factory: Callable[[], nn.Module],
    epochs_per_slice: int,
    batch_size: int,
    lr: float,
    paths: SisaPaths,
) -> Dict[str, int]:
    """Exact unlearning of `record_id`. Returns a report."""
    s = shard_of(record_id, num_shards)
    shard = [r for r in records if shard_of(r[0], num_shards) == s and r[0] != record_id]
    slices = assign_slices(shard, num_slices)

    # Find earliest slice that contained the record in the ORIGINAL assignment.
    original_shard = [r for r in records if shard_of(r[0], num_shards) == s]
    original_slices = assign_slices(original_shard, num_slices)
    r_star = next(
        (i for i, sl in enumerate(original_slices) if record_id in {rec[0] for rec in sl}),
        None,
    )
    if r_star is None:
        return {"shard": s, "slice": -1, "resumed": -1, "status": "not-found"}

    train_shard(
        shard_records=slices,
        model_factory=model_factory,
        epochs_per_slice=epochs_per_slice,
        batch_size=batch_size,
        lr=lr,
        paths=paths,
        shard_index=s,
        resume_from_slice=r_star,
    )
    return {"shard": s, "slice_removed_at": r_star, "status": "ok"}
```

### 2.3 Benchmark harness

```python
# bench.py — reports retraining work saved by SISA and utility delta.
import time
from pathlib import Path

import numpy as np
import torch
from torch import nn
from sisa import (
    SisaPaths, assign_shards, assign_slices,
    train_shard, predict, unlearn,
)


def make_model():
    return nn.Sequential(nn.Linear(28*28, 128), nn.ReLU(), nn.Linear(128, 10))


def load_mnist_flat():   # left as a stub — any 2-class or 10-class toy works
    # returns records: list[(str_id, (np.ndarray, int))]
    ...


def bench(S=10, R=10, epochs_per_slice=1, lr=0.1, batch_size=256):
    records = load_mnist_flat()
    paths = SisaPaths(root=Path("./sisa-run"))
    shards = assign_shards(records, num_shards=S)

    # --- train all shards ---
    t0 = time.perf_counter()
    for s, sh in enumerate(shards):
        train_shard(
            shard_records=assign_slices(sh, R),
            model_factory=make_model,
            epochs_per_slice=epochs_per_slice,
            batch_size=batch_size, lr=lr,
            paths=paths, shard_index=s,
        )
    t_train = time.perf_counter() - t0

    # --- one unlearn ---
    target_id = records[0][0]
    t0 = time.perf_counter()
    report = unlearn(
        record_id=target_id, records=records,
        num_shards=S, num_slices=R,
        model_factory=make_model,
        epochs_per_slice=epochs_per_slice,
        batch_size=batch_size, lr=lr, paths=paths,
    )
    t_unlearn = time.perf_counter() - t0
    print(f"train={t_train:.2f}s, unlearn={t_unlearn:.2f}s, ratio={t_unlearn/t_train:.3f}")
    print(report)
```

For `S = R = 10`, the single-record unlearn should run in
approximately `(1/S) · ((R − r*) / R)` of the full-train time, up to
IO overhead. A record that lands in slice `r* = 0` of its shard
costs `1/S` of a full retrain; a record in slice `r* = R − 1` costs
`1/(S · R)`.

### 2.4 Why this is "exact"

Each shard's training is deterministic in the seed stream
`seed = 1000·s + r`. The checkpoint after slice `r − 1` is a function
only of `D_{s,1}, …, D_{s,r − 1}` and that seed stream. If the
forgotten record was first seen in slice `r*`, the checkpoints
`[0..r* − 1]` are bit-identical to the checkpoints of a hypothetical
training run on `D \ {record}`. Restarting training from slice
`r* − 1` under the same seed stream with the record removed from
slices `r*, …, R − 1` therefore produces the same shard model that a
clean-slate retrain would. The other `S − 1` shards never saw the
record, so their checkpoints are unaffected. SISA's claim to
"exact" unlearning is the statement that *after unlearn, the ensemble
is bit-identical to a from-scratch ensemble trained on `D minus the
record`*. Bourtoule et al. make this the formal claim of the paper.

## 3. Validation steps

1. **Bit-identity test (gold).** Train an ensemble on the full
   `D`, train a second ensemble on `D \ {r0}`, then unlearn `r0`
   from the first ensemble. Compare final-checkpoint hashes
   per shard. All `S` shards must match between the "unlearned" and
   the "fresh" ensemble, byte-for-byte. Failure of this test means
   the determinism is broken somewhere (e.g., dataloader shuffle or
   dropout seed leakage).
2. **Cost-savings test.** Measure wall-clock for a full ensemble
   train vs. one `unlearn`. The ratio must be within 25 % of the
   theoretical `(1/S) · ((R − r*) / R)`.
3. **Shard-assignment stability.** Change the data order in the
   input list; `shard_of(record_id, S)` must still return the same
   shard for the same id. (A shuffle-sensitive sharder breaks
   `unlearn`.)
4. **Aggregation sanity.** Run the ensemble with `label-vote` and
   `logit-average` on the same test batch. Both should produce
   well-defined labels; neither should error on ties — pick the
   first by argmax, documented.
5. **Utility floor.** Compare the ensemble's accuracy at
   `(S, R) = (10, 10)` against a monolithic baseline trained on the
   whole set. SISA utility is expected to be lower than monolithic by
   a few percentage points on small models; the ratio is the cost of
   the unlearning property and is quantifiable.
6. **No-cross-shard-leak test.** Assert that shards' training loops
   only ever see their own `D_s` — a smoke `grep` for any code path
   that combines shards during training (as opposed to during
   aggregation) is enough to catch the obvious bug.
7. **Manifest integrity.** After `unlearn(r0)`, the manifest for the
   affected shard must not contain `r0` in any slice. A manifest that
   still names the removed record is a provenance bug and defeats
   the audit story ([[exercise-07]]).

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Deterministic `shard_of` and `assign_slices` (hash-based, data-order-invariant) | 2 | §2.2 |
| Per-shard sliced training with checkpoint-after-each-slice | 2 | enables resume |
| `unlearn()` finds correct shard and slice, resumes from `r* − 1`, retrains only that shard | 2 | §2.2 |
| Bit-identity test present and passing | 1 | §3.1 |
| Aggregation supports label-vote and logit-average | 1 | — |
| Cost-savings reported with theoretical ratio | 1 | §3.2 |
| Manifest / provenance updated after unlearn | 1 | audit prerequisite |

Automatic failures:

- Retraining the whole ensemble on `unlearn()` (defeats the point).
- Sharing a dataloader shuffle across shards (breaks determinism
  and the exact-unlearning claim).
- Claiming "exact" unlearning without the bit-identity test.
- Keeping the removed record in the manifest or the slice files after
  `unlearn`.
- Using a non-deterministic hash (Python `hash` is salted per
  process; `sha256` is the correct primitive).

## 5. Common mistakes

- **Treating `S` and `R` as free parameters.** They are governed by
  the utility-cost curve: large `S` ⇒ cheaper unlearning but weaker
  aggregated model; large `R` ⇒ finer-grained unlearning but more
  checkpoint IO. The Bourtoule paper's §VI.C reports empirical
  guidance; the exercise should at least sweep `S ∈ {4, 10, 20}` and
  report both axes.
- **Non-deterministic dataloaders.** `DataLoader(shuffle=True)`
  without `torch.manual_seed` + `generator=` produces a different
  training trajectory each run. The exact-unlearning claim requires
  the trajectory to be reproducible.
- **Using the Python built-in `hash`.** Python hashes are randomised
  per interpreter instance (PEP 456). A record that lands in shard 3
  today lands in shard 7 tomorrow. Use SHA-256 or any
  non-randomised hash.
- **Forgetting the manifest.** The audit story in [[exercise-07]]
  depends on being able to prove the record is no longer in training
  data. The manifest is the primary evidence; omitting its update
  breaks the audit.
- **Overcounting savings.** The theoretical ratio assumes the
  earliest slice containing the record is `r*`. If the slicer is
  uniformly random, the *expected* cost is `(1/S) · ((R + 1) / (2R))`
  per unlearn, not `(1/S) · (1/R)`. Reporting the best case as the
  norm is misleading.
- **Claiming SISA is a privacy guarantee.** SISA is a *model-level*
  erasure mechanism. It says the final parameters are those of a
  clean retrain, not that the original data leakage from the first
  training run (logs, metrics, A/B telemetry) is erased. The runbook
  in [[exercise-01]] must still touch the other eight surfaces.
- **Combining SISA with full retraining heuristically.** For
  high-volume erasure (hundreds of records per day), SISA scales
  better than monolithic retrain; for very small volumes, a monthly
  full retrain can be simpler. The decision tree in [[exercise-06]]
  is where this trade-off belongs.
- **Ignoring the aggregation side-channel.** The ensemble output
  reveals which constituent voted; inference-time attacks (ATLAS
  `AML.T0024`) can probe per-shard behaviour. Label-vote minimises
  this; logit-average increases per-shard inferability.

## 6. References

- Bourtoule, L.; Chandrasekaran, V.; Choquette-Choo, C. A.; Jia, H.;
  Travers, A.; Zhang, B.; Lie, D.; Papernot, N. *Machine Unlearning.*
  IEEE S&P 2021. <https://arxiv.org/abs/1912.03817> — the SISA paper
  this exercise implements.
- Cao, Y.; Yang, J. *Towards Making Systems Forget with Machine
  Unlearning.* IEEE S&P 2015. <https://doi.org/10.1109/SP.2015.35> —
  the original "machine unlearning" formulation that motivates
  exact approaches.
- Nguyen, T. T.; Huynh, T. T.; Nguyen, P. L.; Liew, A. W.-C.; Yin, H.;
  Nguyen, Q. V. H. *A Survey of Machine Unlearning.*
  <https://arxiv.org/abs/2209.02299> — background on exact vs.
  approximate.
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
