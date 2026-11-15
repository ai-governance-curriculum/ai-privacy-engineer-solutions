# Exercise 05 — PATE Ensemble Engineering — Reference Solution

## 1. Solution overview

This is an **implementation + design exercise**. The learner must build
a **PATE** (Private Aggregation of Teacher Ensembles) pipeline that:

1. Partitions sensitive training data across *K* disjoint teachers.
2. Trains the teachers independently on their shards.
3. Aggregates their votes on an **unlabelled public dataset** through
   the Gaussian noisy-max (GNMAX) aggregator (Papernot et al. 2018).
4. Trains a **student** on the noisily-labelled public data.
5. Reports a defensible (ε, δ) using the **data-dependent** moments
   accountant published with the PATE paper.

The assessable outputs are:

- A pipeline (Python) with explicit roles for teacher training, voting,
  aggregation, and student training.
- A privacy report stating (ε, δ) with the accountant name, number of
  teachers *K*, GNMAX noise σ, and number of labelled public examples.
- A one-page design memo explaining when PATE is preferable to DP-SGD
  and what PATE does *not* protect against.

PATE is the right tool when you have (a) a lot of sensitive data you
can shard and (b) a smaller unlabelled public dataset in the same
distribution. If (b) is unavailable, use DP-SGD (Exercise 02 / 03)
instead.

## 2. Worked implementation

### 2.1 Pipeline skeleton (illustrative — framework-agnostic)

```python
# pate_pipeline.py
import math
import numpy as np

# ---------- 1. Shard the sensitive data across K teachers ----------
def shard(private_dataset, k: int, rng):
    # Disjoint shards. For user-level privacy, shard BY USER, not by row.
    indices = rng.permutation(len(private_dataset))
    return [private_dataset.subset(chunk) for chunk in np.array_split(indices, k)]

# ---------- 2. Train K teachers independently ----------
def train_teachers(shards, train_fn):
    # Any training routine works (non-DP). Teachers never touch each other's
    # data. Shard disjointness is what makes aggregation DP-analysable.
    return [train_fn(s) for s in shards]

# ---------- 3. Collect votes on public (unlabelled) data ----------
def collect_votes(teachers, public_x, num_classes: int) -> np.ndarray:
    # Returns an integer matrix V of shape (len(public_x), num_classes)
    # where V[i, c] = number of teachers that predicted class c for example i.
    votes = np.zeros((len(public_x), num_classes), dtype=np.int64)
    for t in teachers:
        preds = t.predict(public_x)   # argmax class labels
        for i, c in enumerate(preds):
            votes[i, c] += 1
    return votes

# ---------- 4. Noisy aggregation (GNMAX) + Confident-GNMAX gating ----------
def gnmax(vote_counts: np.ndarray, sigma: float, rng) -> int:
    # Add independent Gaussian noise to each class's vote count and
    # return the argmax. Sensitivity = 1 under "replace-one-teacher"
    # since a single teacher moves at most one count up and one down,
    # but with disjoint shards it affects exactly one count by 1.
    noisy = vote_counts + rng.normal(scale=sigma, size=vote_counts.shape)
    return int(np.argmax(noisy))

def confident_gnmax(vote_counts, sigma1, sigma2, threshold, rng):
    """Papernot et al. 2018 Confident-GNMAX.
    First tests whether the top count is confidently above `threshold`
    using sigma1-noise; if not, skips labelling (budget-cheap). If yes,
    labels using independent sigma2-noise.
    """
    top = vote_counts.max()
    if top + rng.normal(scale=sigma1) < threshold:
        return None   # abstain; no label published
    return gnmax(vote_counts, sigma2, rng)

# ---------- 5. Train the student on noisily-labelled public data ----------
def label_public(public_x, teachers, sigma1, sigma2, threshold, rng, num_classes):
    labels, kept = [], []
    votes = collect_votes(teachers, public_x, num_classes)
    for i, row in enumerate(votes):
        y = confident_gnmax(row, sigma1, sigma2, threshold, rng)
        if y is not None:
            labels.append(y)
            kept.append(i)
    return np.array(kept), np.array(labels), votes

def train_student(train_fn, public_x, kept, labels):
    return train_fn(public_x[kept], labels)
```

Points a reviewer verifies:

- **Disjoint sharding.** `np.array_split(permutation, k)` yields disjoint
  shards. If the unit of privacy is "user," the permutation must be over
  user IDs, not rows. Shared rows between teachers invalidates the DP
  proof.
- **Teachers are *not* DP individually.** Each teacher trains on its
  shard with ordinary (non-DP) SGD. The DP budget is *entirely* spent
  at aggregation time.
- **Confident-GNMAX is the recommended aggregator.** The vanilla GNMAX
  spends budget on every public example even when teachers disagree
  strongly; the "confident" variant abstains cheaply and keeps the
  per-label budget tight for high-agreement cases.
- **Student training is non-DP on the kept labels.** The student never
  touches the private data; its training is post-processing of the DP
  labels and therefore does not consume additional ε under the
  post-processing theorem.

### 2.2 Privacy accounting

PATE is **data-dependent**: the (ε, δ) depends on the actual vote
margins seen during labelling (teachers agree → tight ε; teachers
disagree → loose ε). The paper publishes a moments accountant that
tracks the data-dependent cost per labelled example.

Reference accountant (pseudocode of the accounting step — use a
library implementation in production, e.g., the
`tensorflow_privacy.privacy.analysis.rdp_accountant` or the
reference code released with Papernot et al. 2018):

```python
# Rough shape; use the published implementation.
def pate_rdp_cost(vote_counts, sigma, orders):
    # Returns per-order RDP epsilon contributions for THIS labelled example.
    # The data-dependent bound in Papernot et al. 2018 Theorem 6 uses the
    # second-largest count; near-unanimous votes contribute near zero.
    ...
```

A release reports:

- The **data-dependent** ε at the chosen δ, computed over the labelled
  examples that were actually released to the student.
- The **data-independent** ε as an upper bound (useful when the vote
  distribution itself is considered sensitive; see Q7 below).
- The number of labelled examples (`sum(kept)`), *K*, `sigma1`,
  `sigma2`, and `threshold`.

### 2.3 Design memo — when to pick PATE (one-pager outline)

1. **Problem shape.** Enough sensitive data to shard into ≥ 50 disjoint
   training sets *and* an unlabelled public dataset in-distribution.
2. **Why PATE over DP-SGD.** Teacher training is unconstrained (no
   clipping, no per-sample gradient); sometimes gives better utility
   when per-sample clipping cripples DP-SGD. Student is a plain model,
   easier to serve.
3. **Why DP-SGD over PATE.** No public dataset available; or training
   data cannot be partitioned cleanly (streaming, correlated users);
   or the task has few labelled examples per class and shardwise
   teacher accuracy collapses.
4. **Data-dependent ε honesty.** The *released* ε depends on the data
   the teachers saw; publishing it without the data-independent upper
   bound can itself leak information. Papernot et al. 2018 recommends
   privatising the ε value for release in sensitive settings.
5. **Shard vs. user partitioning.** Always partition by the unit of
   privacy, not by row.

## 3. Validation steps

1. **Shard disjointness.** `assert sum(len(s) for s in shards) == len(private)`
   and all shard index sets intersect to the empty set.
2. **User-level sharding (if that's the unit).** The shard split is on
   user IDs; a user's rows never appear in two teachers.
3. **Teachers trained independently.** No shared state (optimizer, seed,
   data) across teachers. In practice, that means spawning teacher
   trainings as independent jobs.
4. **Label-side DP check.** The only DP noise in the pipeline is inside
   `gnmax`. The student training routine is non-DP (and MUST be — adding
   DP noise to a DP-labelled dataset would inflate ε).
5. **Accountant transcript.** The reported ε equals what the accountant
   returns for the kept-examples list; replay the accountant on the
   logged vote matrix and the two σ values.
6. **Abstain rate check.** Confident-GNMAX typically abstains on
   10–60 % of examples in realistic runs. An abstain rate near 0 % or
   100 % usually means `sigma1` or `threshold` is miscalibrated.
7. **Post-processing honesty.** The student has no backchannel to the
   private data: no features derived from private rows, no hyperparameter
   search informed by private metrics.
8. **Second-opinion recomputation.** For at least one labelled example,
   recompute GNMAX by hand: take the vote row, add independent Gaussian
   noise with σ = `sigma2`, argmax, and verify the code agrees.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Disjoint shards at the stated unit of privacy | 2 | User-level sharding if that's the unit |
| Teachers trained independently and non-DP | 1 | Shared state fails |
| Aggregator is GNMAX or Confident-GNMAX with explicit σ (and threshold) | 2 | Vanilla GNMAX acceptable with justification |
| Student trained on *kept* labelled examples only, as post-processing | 1 | — |
| Privacy report uses PATE's data-dependent moments accountant | 2 | RDP conversion to (ε, δ); both data-dependent and data-independent values |
| Memo identifies when PATE is / is not preferable to DP-SGD | 1 | Honest trade-offs |
| δ ≪ 1/N recorded and checked | 1 | N = number of distinct units in the private set |

Automatic failures:

- Overlapping shards.
- DP noise added anywhere other than the aggregator.
- Student training using features or hyperparameters derived from the
  private data (post-processing violation).
- Reporting ε without saying which accountant produced it.
- Treating the student's own validation on private data as "just
  evaluation" — any read of the private data by the student pipeline
  consumes budget.

## 5. Common mistakes

- **Sharding by row instead of by user.** If one user appears in two
  shards, deleting that user moves counts in two teacher votes, not
  one, and the sensitivity assumption in the GNMAX proof breaks.
- **Letting teachers share hyperparameter sweeps on private data.**
  Hyperparameter tuning that reads private validation metrics is a
  private computation and either (a) must itself be DP or (b) must be
  done on public holdout only.
- **Reporting only the data-dependent ε.** The data-dependent ε is tight
  but itself depends on private information (vote margins). Papernot
  et al. 2018 §5.3 notes that disclosing the data-dependent ε can leak;
  pair it with the data-independent upper bound.
- **Choosing `sigma1`, `sigma2`, `threshold` by taste.** The three
  parameters trade abstain rate, per-label cost, and attack margin.
  Grid-search them *on a public simulation*, not on the real teachers'
  votes, before training the student.
- **Under-provisioning teachers.** PATE needs enough teachers for the
  vote distribution to have signal. K = 5 is almost never enough; the
  original paper uses K ≥ 100 for image tasks.
- **Letting the student overfit the released labels.** The student
  training is post-processing — its overfitting is not a DP violation,
  but a student that memorises the noisily-labelled public set defeats
  the purpose. Early stopping on a public holdout is required.
- **Forgetting the student model is DP via post-processing *only if the
  inputs are disjoint from the private data*.** If the public data is
  actually a re-release of private data, PATE is leaking.

## 6. References

- Papernot, N.; Abadi, M.; Erlingsson, Ú.; Goodfellow, I.; Talwar, K.
  *Semi-supervised Knowledge Transfer for Deep Learning from Private
  Training Data.* ICLR 2017 (PATE).
  <https://arxiv.org/abs/1610.05755>
- Papernot, N.; Song, S.; Mironov, I.; Raghunathan, A.; Talwar, K.;
  Erlingsson, Ú. *Scalable Private Learning with PATE.* ICLR 2018
  (Confident-GNMAX and data-dependent RDP accountant).
  <https://arxiv.org/abs/1802.08908>
- Mironov, I. *Rényi Differential Privacy.* CSF 2017.
  <https://arxiv.org/abs/1702.07476>
- NIST Special Publication 800-226, *Guidelines for Evaluating
  Differential Privacy Guarantees* (final, 2025).
  <https://csrc.nist.gov/pubs/sp/800/226/final>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- Reference PATE implementation and accountant.
  <https://github.com/tensorflow/privacy/tree/master/research/pate_2018>
