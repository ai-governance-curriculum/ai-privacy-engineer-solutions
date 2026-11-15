# Exercise 03 — DP-SGD Training with TensorFlow Privacy — Reference Solution

## 1. Solution overview

This is an **implementation exercise** that mirrors Exercise 02, but
with TensorFlow / Keras and the `tensorflow_privacy` library. The
learner must convert a standard `tf.keras` training loop to DP-SGD,
use TF Privacy's DP optimizer (vectorised, per-sample-gradient-aware),
run it to completion, and produce a reviewable privacy statement using
the official `compute_dp_sgd_privacy_statement` function.

Assessable outputs:

1. A **training script** using `tensorflow_privacy.DPKerasSGDOptimizer`
   (or its Adam / Adagrad counterpart) with explicit `l2_norm_clip`,
   `noise_multiplier`, and `num_microbatches`.
2. A **privacy statement** produced by TF Privacy's official helper,
   with the input parameters pinned.
3. A **release manifest** matching the schema used in Exercise 02 so the
   two trainings (Opacus vs. TF Privacy) can be compared row-for-row.

The script below uses the API surface documented in
`tensorflow/privacy` (DP optimizers and `compute_dp_sgd_privacy_statement`).
Where the API has drifted historically (e.g., `compute_dp_sgd_privacy`
is deprecated in favour of the newer "statement" helper), the solution
uses the current recommendation.

## 2. Worked implementation

### 2.1 Minimal DP-SGD training script

```python
# train_dp_tf.py
import tensorflow as tf
from tensorflow_privacy.privacy.optimizers.dp_optimizer_keras import (
    DPKerasSGDOptimizer,
)
from tensorflow_privacy.privacy.analysis.compute_dp_sgd_privacy_lib import (
    compute_dp_sgd_privacy_statement,
)

# --- Hyperparameters tied to the privacy claim ---
NUM_EXAMPLES     = 60_000
BATCH_SIZE       = 250                 # also the number of microbatches below
MICROBATCHES     = 250                 # per-example gradients
L2_NORM_CLIP     = 1.0
NOISE_MULTIPLIER = 1.1
EPOCHS           = 15
LEARNING_RATE    = 0.15
TARGET_DELTA     = 1e-5

assert BATCH_SIZE % MICROBATCHES == 0, (
    "BATCH_SIZE must be an integer multiple of MICROBATCHES; one microbatch "
    "per example gives the tightest per-example clipping."
)

def build_model() -> tf.keras.Model:
    # DP-SGD needs per-example losses, so the output layer uses from_logits=True
    # and the loss uses reduction=NONE. BatchNorm is replaced by LayerNorm
    # because BN's batch statistics couple samples and invalidate per-example
    # gradient accounting.
    return tf.keras.Sequential([
        tf.keras.layers.Reshape((28, 28, 1), input_shape=(28, 28)),
        tf.keras.layers.Conv2D(16, 8, strides=2, padding="same", activation="relu"),
        tf.keras.layers.LayerNormalization(),
        tf.keras.layers.MaxPool2D(2, 1),
        tf.keras.layers.Conv2D(32, 4, strides=2, activation="relu"),
        tf.keras.layers.Flatten(),
        tf.keras.layers.Dense(32, activation="relu"),
        tf.keras.layers.Dense(10),
    ])

def main(train_ds, val_ds):
    model = build_model()

    # Per-example loss — critical. reduction=NONE preserves the shape
    # (BATCH,) so the DP optimizer can clip and noise each example's grad.
    loss_fn = tf.keras.losses.CategoricalCrossentropy(
        from_logits=True,
        reduction=tf.losses.Reduction.NONE,
    )

    optimizer = DPKerasSGDOptimizer(
        l2_norm_clip=L2_NORM_CLIP,
        noise_multiplier=NOISE_MULTIPLIER,
        num_microbatches=MICROBATCHES,
        learning_rate=LEARNING_RATE,
    )

    model.compile(optimizer=optimizer, loss=loss_fn, metrics=["accuracy"])

    model.fit(
        train_ds,
        epochs=EPOCHS,
        validation_data=val_ds,
        batch_size=BATCH_SIZE,
    )

    # Reviewable privacy statement — this is the canonical output.
    statement = compute_dp_sgd_privacy_statement(
        number_of_examples=NUM_EXAMPLES,
        batch_size=BATCH_SIZE,
        num_epochs=EPOCHS,
        noise_multiplier=NOISE_MULTIPLIER,
        delta=TARGET_DELTA,
    )
    print(statement)

    return {
        "privacy_statement": statement,
        "params": {
            "l2_norm_clip": L2_NORM_CLIP,
            "noise_multiplier": NOISE_MULTIPLIER,
            "num_microbatches": MICROBATCHES,
            "batch_size": BATCH_SIZE,
            "epochs": EPOCHS,
            "delta": TARGET_DELTA,
            "N": NUM_EXAMPLES,
        },
    }
```

Design decisions a reviewer verifies:

- **`DPKerasSGDOptimizer`** (not plain `SGD`). Adam / Adagrad counterparts
  exist for other optimizers; the DP property is the clip-and-noise, not
  the base optimiser's update rule.
- **`num_microbatches` = `BATCH_SIZE`.** One example per microbatch
  yields the tightest per-example clipping. Using fewer microbatches
  averages gradients before clipping and *degrades* the privacy/utility
  trade-off (it does not break the DP proof, but it does loosen it).
- **Loss uses `reduction=NONE`.** The optimizer needs a vector loss to
  compute per-example gradients. A scalar `reduction=SUM_OVER_BATCH_SIZE`
  loss raises in `DPKeras*Optimizer`.
- **LayerNorm, not BatchNorm.** Same reasoning as Opacus — BN couples
  samples across the batch dimension.
- **`compute_dp_sgd_privacy_statement`** is the *reportable* ε. The
  older `compute_dp_sgd_privacy` function uses a cruder accountant and
  is deprecated.

### 2.2 Example privacy statement (format)

The helper returns a multi-line string. An example shape (values will
differ per run):

```
DP-SGD performed over 60000 examples with 250 examples per iteration, noise
multiplier 1.1 for 15 epochs with microbatching, and no bound on number of
users.

This privacy guarantee protects the release of all model checkpoints in
addition to the final model.

Example-level DP with add-or-remove-one adjacency at delta = 1e-05 computed
with PLD accounting:
    Epsilon with each example occurring once per epoch:     <epsilon_1>
    Epsilon assuming Poisson sampling (*):                  <epsilon_2>
...
```

Record the *entire* statement verbatim in the release manifest. Do not
summarise it into a single ε — the statement explicitly distinguishes
"each example occurs once per epoch" (shuffle semantics) from "Poisson
subsampling," and auditors read those two lines differently.

### 2.3 Release manifest

| Field | Value |
|---|---|
| Mechanism | DP-SGD (Abadi et al. 2016), implemented by TF Privacy DPKeras*Optimizer |
| Unit of privacy | Example (default) or User (if the data loader groups by user) |
| Neighbouring relation | Add/remove one example (or user) |
| Accountant | PLD / RDP, per the version of `compute_dp_sgd_privacy_statement` |
| `l2_norm_clip` | `L2_NORM_CLIP` |
| `noise_multiplier` | `NOISE_MULTIPLIER` |
| `batch_size`, `num_microbatches` | Pinned numbers from the script |
| `num_epochs`, `number_of_examples` | Pinned numbers from the script |
| `delta` | `TARGET_DELTA` (δ ≪ 1/N) |
| Library version | `tensorflow_privacy==<pin>` + `tensorflow==<pin>` |
| Sampling convention | "once per epoch" vs. "Poisson" — the helper reports both; mark which one the loader actually implements |

## 3. Validation steps

1. **Per-example loss check.** `loss_fn(y_true, y_pred).shape == (BATCH,)`.
   If it's scalar, the optimizer will compute wrong gradients.
2. **Microbatch sanity.** `BATCH_SIZE % num_microbatches == 0`; and
   preferably `num_microbatches == BATCH_SIZE` for per-example clipping.
3. **Model layer compatibility.** Grep the model for `BatchNormalization`
   — must be zero occurrences.
4. **Privacy statement reproducibility.** Running
   `compute_dp_sgd_privacy_statement` with the same arguments twice
   gives the same output; store it and diff.
5. **δ sanity.** `delta < 1 / number_of_examples`. The helper does not
   enforce this; the release manifest must.
6. **ε regression.** Keep the statement from the previous release; a
   training change that increases ε at the same accuracy is a regression
   that needs justification.
7. **Sampling convention match.** If the data loader shuffles and iterates
   (no Poisson), report the "each example once per epoch" ε. Only report
   the "Poisson" ε if the loader actually uses Poisson sampling.
8. **Determinism harness.** Set `tf.keras.utils.set_random_seed`, run
   twice; the privacy statement is deterministic independent of the
   training seed — training loss curves should match within tolerance.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Uses a `DPKeras*Optimizer` with all three DP hyperparameters set explicitly | 2 | `l2_norm_clip`, `noise_multiplier`, `num_microbatches` |
| Loss function has `reduction=NONE` and `from_logits=True` (or equivalent) | 1 | Per-example losses are mandatory |
| Model is DP-compatible (no BatchNorm; LayerNorm/GroupNorm used) | 1 | — |
| `num_microbatches == batch_size` for tightest clipping (or justified deviation) | 1 | — |
| Reports ε via `compute_dp_sgd_privacy_statement`, not the deprecated helper | 1 | — |
| Chosen sampling convention (shuffle vs. Poisson) matches what the loader does | 1 | — |
| Manifest records δ, N, and performs δ ≪ 1/N check | 1 | — |
| Library versions pinned for TF, TF Privacy, and the privacy statement is stored verbatim | 2 | Reviewer can reproduce |

Automatic failures:

- Using `tf.keras.losses.X(reduction=AUTO)` (scalar) with a DP optimizer.
- Reporting ε from `compute_dp_sgd_privacy` (deprecated) without justification.
- Averaging gradients before clipping (symptom: `num_microbatches = 1`).
- Any BatchNorm layer in the trainable model.
- Reporting "Poisson subsampling ε" when the loader shuffles deterministically.

## 5. Common mistakes

- **Scalar loss.** Default Keras losses reduce over the batch dimension.
  A DP optimizer needs the per-example vector loss to compute per-example
  gradients.
- **Confusing microbatches with mini-batches.** Microbatches in TF
  Privacy are the unit of clipping, not the unit of SGD step. Setting
  `num_microbatches = 1` recovers plain SGD with noise added once;
  setting `num_microbatches = batch_size` is the per-example-clip version
  the DP proof assumes.
- **Keeping BatchNorm.** Same failure mode as Opacus. Replace with
  LayerNorm / GroupNorm.
- **Reporting ε from `compute_dp_sgd_privacy`.** The older function
  uses an older accountant and is deprecated. The current recommendation
  is `compute_dp_sgd_privacy_statement`, which uses PLD accounting.
- **Reporting the "Poisson subsampling" ε when using plain batches.**
  The statement helper reports two numbers for a reason — choose the
  correct one for how the loader actually samples.
- **Setting δ by taste.** δ must be smaller than 1/N; 1e-5 with N = 60 000
  is fine, 1e-3 with N = 100 is not.
- **Changing model while reusing the ε.** The ε depends on the
  mechanism's sensitivity (set by `l2_norm_clip`) and on the schedule
  (noise, batch, epochs). Changing the architecture at fixed hyperparameters
  does not invalidate ε, but changing *clip norm, noise, batch size,
  epochs, or N* does. Recompute.
- **Not pinning TF Privacy.** The accountant implementation evolves; the
  statement is tied to the version.

## 6. References

- Abadi, M. et al. *Deep Learning with Differential Privacy.* ACM CCS 2016.
  <https://arxiv.org/abs/1607.00133>
- TensorFlow Privacy project.
  <https://github.com/tensorflow/privacy>
- TensorFlow Privacy classification tutorial.
  <https://www.tensorflow.org/responsible_ai/privacy/tutorials/classification_privacy>
- `compute_dp_sgd_privacy_statement` source.
  <https://github.com/tensorflow/privacy/blob/master/tensorflow_privacy/privacy/analysis/compute_dp_sgd_privacy_lib.py>
- Mironov, I. *Rényi Differential Privacy.* CSF 2017.
  <https://arxiv.org/abs/1702.07476>
- Doroshenko, V.; Ghazi, B.; Kamath, P.; Kumar, R.; Manurangsi, P.
  *Connect the Dots: Tighter Discrete Approximations of Privacy Loss
  Distributions.* PoPETs 2022 (PLD accountant used in the current
  statement helper).
  <https://arxiv.org/abs/2207.04380>
- NIST Special Publication 800-226, *Guidelines for Evaluating
  Differential Privacy Guarantees* (final, 2025).
  <https://csrc.nist.gov/pubs/sp/800/226/final>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
