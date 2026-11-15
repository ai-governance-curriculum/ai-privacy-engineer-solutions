# Exercise 05 — Canary Design and the Cross-Corpus False-Positive Drill — Reference Solution

## 1. Solution overview

This is a **design + implementation** exercise. Canaries are the core
instrument of the Carlini et al. "Secret Sharer" line of work
(Carlini, Liu, Erlingsson, Kos, Song, "The Secret Sharer: Evaluating and
Testing Unintended Memorization in Neural Networks," USENIX Security 2019,
<https://arxiv.org/abs/1802.08232>). The idea is simple and
widely misused: inject known, synthetic, distinctive strings into training
data, then measure the model's **exposure** to them. Measuring exposure
correctly requires engineering discipline and, critically, a **cross-corpus
false-positive drill** — running the same detector against models that
*never saw the canary* to establish the FP floor.

Deliverables:

1. A **canary design** — format, entropy, uniqueness, repetition schedule,
   categorical coverage.
2. A **training-time pipeline** that injects canaries and records their
   exact ground truth.
3. An **exposure metric** implementation following Carlini et al. 2019
   Def. 3 (log-perplexity rank).
4. A **cross-corpus FP drill**: the same detection machinery run against
   N "clean" models (known never to have seen the canary) to characterise
   the detector's false-positive rate at a given threshold.
5. A **review memo** that prevents stakeholders from reporting a raw
   canary-hit number as "the model memorises X% of our data."

## 2. Worked implementation

### 2.1 Canary design — what makes a good canary

A canary must be:

- **High-entropy.** Random tokens from a large alphabet; Carlini et al.
  2019 use e.g. 9-digit random suffixes inside a template. Entropy is what
  lets "exposure" be well-defined.
- **Distinctive.** The template must be unlikely to appear in any natural
  corpus. If the format can plausibly occur in training data unrelated to
  the canary, the FP drill will look bad.
- **Categorical.** Make several canary families so you can disaggregate
  risk by data class. Example family schema (synthetic, non-PII):

| Family | Template | Entropy source |
|---|---|---|
| CANARY-ID | `canary-id-{nonce20}` | 20 random alphanumerics |
| CANARY-SEQ | `the mystery code is {d9}` | 9 random digits |
| CANARY-PHRASE | `{adj}-{noun}-{d9}` | fixed vocab + digits |
| CANARY-STRUCTURED | `{"canary_id":"{uuid4}","payload":"{noun16}"}` | UUID + noun |

- **Un-PII.** Canaries are *synthetic*. Do not use real phone numbers,
  real names, real document IDs. Never inject real PII to create a demo.
- **Pre-registered.** Record every canary's ground-truth string, family,
  injection location(s), and repetition count in a sealed manifest *before*
  training. This manifest is the audit trail for the exposure score.

Repetition schedule: Carlini et al. 2019 vary insertion count explicitly;
typical ladder is `R ∈ {1, 4, 16, 64, 256}` per canary family to get a
clean exposure-vs-R curve.

### 2.2 Exposure metric — Carlini et al. 2019 Def. 3

Given a canary `s` drawn from a randomness space `R` of size `|R|`:

```
rank_θ(s) = | { s' ∈ R : log_perplexity_θ(s') ≤ log_perplexity_θ(s) } |
exposure_θ(s) = log2(|R|) - log2( rank_θ(s) )
```

Exposure is in bits. An exposure of 1 bit means the model considers `s`
better than half the random alternatives; `log2(|R|)` bits means the
model places `s` as the single most-likely string in `R`.

```python
# exposure.py
import math

def log_perplexity(model, tokens):
    # negative mean log-likelihood on the string tokens
    return -model.log_likelihood(tokens) / max(len(tokens), 1)

def exposure_by_sampling(model, canary_tokens, sample_fn, num_samples=10_000):
    """sample_fn() returns a random tokens list from R, same template as canary."""
    canary_pp = log_perplexity(model, canary_tokens)
    better_or_equal = 1  # include the canary itself
    for _ in range(num_samples):
        if log_perplexity(model, sample_fn()) <= canary_pp:
            better_or_equal += 1
    # Approx rank; see Carlini 2019 §5 for extrapolation beyond sample size.
    rank_estimate = (better_or_equal / (num_samples + 1)) * (len(sample_fn.space))
    return math.log2(len(sample_fn.space)) - math.log2(max(rank_estimate, 1))
```

For an entropy space too large to enumerate (e.g., 10^9 for 9-digit
suffixes), use Carlini et al. 2019 §5's extrapolation approach: fit a
skew-normal to the sampled log-perplexity distribution and compute the
canary's tail position analytically.

### 2.3 Training-time pipeline — required properties

- The injection must be idempotent and seeded. Re-running the pipeline
  with the same seed yields the same manifest.
- The manifest includes: canary string, family, repetition count,
  pre-training line indices or shard offsets, and a hash of the final
  training set that confirms the canaries landed.
- Deduplication *must* be applied *after* canary injection (otherwise the
  dedup step silently collapses repetitions to 1). Record whether dedup
  ran and at what step.

### 2.4 Cross-corpus false-positive drill — the heart of the exercise

This is the step most teams skip. Procedure:

1. Collect **N ≥ 3 "clean" reference models** — models trained on
   corpora that demonstrably do **not** contain any of the canary family
   members. This includes a public open-weights model trained before the
   canaries were generated (`cutoff_date < canary_generation_date`).
2. For each canary `s` and each clean model `f_clean`, compute
   `exposure_{f_clean}(s)`. This yields a distribution of
   *should-be-zero* exposures.
3. Report the **95th percentile** of the clean-model exposure
   distribution. That is your *false-positive floor*. A target-model
   exposure below this floor is not a leakage finding; it is within the
   noise band of the metric.
4. Report target exposure as `(exposure_target, FP_floor)` *pairs*,
   not as a single number.

```python
# fp_drill.py
def fp_floor(clean_models, canaries, sample_fn, num_samples=5_000, pct=95):
    import numpy as np
    exposures = []
    for f in clean_models:
        for c in canaries:
            exposures.append(exposure_by_sampling(f, c, sample_fn, num_samples))
    return float(np.percentile(exposures, pct))
```

### 2.5 Review memo — required structure

Two pages. The memo must answer:

- **What was injected?** Family × R × count. Attach the manifest hash.
- **What did the target model's exposure look like per family and per R?**
  A table, plus a plot of exposure vs R.
- **What is the FP floor?** Named, measured, with clean-model lineage.
- **Which exposures are above the FP floor by how many standard
  deviations?** Nothing within the floor is reported as leakage.
- **What is *not* measured?** Canary families you did not inject (e.g.,
  if you only tested numeric canaries, do not generalise to free-text
  PII memorisation).

### 2.6 Handover

- If target exposure > FP floor and `R` is small, route to mod-106 for
  DP-SGD and to mod-103 for deduplication policy.
- If exposure is high for `R=1` (one-shot memorisation), this is a
  "secret-sharer" signal — single-example memorisation. Carlini et al.
  2019 Table 2 shows this can happen even without pathological training.
  Treat as a high-priority finding regardless of model size.

## 3. Validation steps

1. **Injection confirmation.** Grep the finalised training manifest for
   each canary string and record the match count. It must equal the
   intended `R` per canary. If dedup silently collapsed duplicates, the
   pipeline is misordered.
2. **Clean-model FP floor is not zero.** With a finite entropy space and
   a finite sample size, the clean-model exposure distribution has mass
   above zero. If the FP floor comes out at exactly 0.0, the sampler is
   broken (probably returning the canary string itself).
3. **Exposure saturates at `log2(|R|)`.** Exposure cannot exceed
   `log2(|R|)`. If the reported number is larger, there is a numerical bug.
4. **Family vs family.** Compute exposure per family. If one family
   systematically shows higher exposure than others at the same `R`, the
   finding is categorical — name the family and its template in the memo.
5. **Decoder-free metric.** The exposure metric is computed on log
   likelihood, not on sampling. If the learner accidentally measures
   "fraction extracted by top-p sampling at temperature 1" and calls it
   exposure, correct this — that metric is in exercise-04.

## 4. Rubric or review checklist

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Canary families defined with explicit template, entropy, non-PII | 2 | At least three families |
| Injection pipeline is seeded + manifest + verified post-dedup | 1 | Manifest hash present |
| Exposure metric implemented per Carlini et al. 2019 Def. 3 | 2 | Not "fraction extracted" |
| Cross-corpus FP drill with ≥ 3 clean models | 2 | Clean lineage named |
| Report expresses target exposures relative to the FP floor | 1 | Not as naked numbers |
| Exposure-vs-R curve plotted per family | 1 | |
| "What is not measured" paragraph present | 1 | Categorical coverage gap |

Automatic failures:

- Using real PII as a canary. Automatic fail — a canary is a synthetic
  marker.
- Reporting a target-model exposure without a comparison against any
  clean model. Zero-FP-drill is zero-finding.
- Reporting a "zero exposure" result while never evaluating the model on
  the clean-model space (sample was the canary itself).

## 5. Common mistakes

- **Skipping the FP drill.** Exposure is a signed number with a
  distribution under the null hypothesis. Without a clean-model control,
  you can't tell signal from noise.
- **Injecting real PII as the canary.** The entire point of a canary is
  that it is *synthetic*; using real data turns the drill into a data
  breach.
- **Deduplicating after canary injection.** Dedup silently collapses `R`
  to 1; the exposure-vs-R curve becomes meaningless.
- **Reporting the canary-hit rate from greedy decoding.** That is the
  extraction metric from exercise-04, not exposure. They measure
  different things; a model can have high exposure but, under greedy
  decoding, not emit the canary (tail probability without mode).
- **Using a tiny `|R|`.** If `|R|` is small (say, 100), the maximum
  exposure is tiny (log2 100 ≈ 6.6 bits) and the metric saturates. Use
  high-entropy templates (|R| ≥ 10^9 is typical).
- **One canary per family.** Variance per family matters. Inject multiple
  canaries per family per R so you can report mean and spread.
- **Claiming "no memorisation" because exposures are below the FP
  floor.** Correct wording: "canary memorisation is *not distinguishable
  from the null distribution* at this budget and family coverage." That
  is honest and routes correctly.

## 6. References

- Nicholas Carlini, Chang Liu, Úlfar Erlingsson, Jernej Kos, Dawn Song.
  "The Secret Sharer: Evaluating and Testing Unintended Memorization in
  Neural Networks." USENIX Security, 2019.
  <https://arxiv.org/abs/1802.08232>
- Nicholas Carlini, Daphne Ippolito, Matthew Jagielski, Katherine Lee,
  Florian Tramèr, Chiyuan Zhang. "Quantifying Memorization Across Neural
  Language Models." ICLR 2023. <https://arxiv.org/abs/2202.07646>
- Katherine Lee, Daphne Ippolito, et al. "Deduplicating Training Data
  Makes Language Models Better." ACL 2022. <https://arxiv.org/abs/2107.06499>
- OWASP Machine Learning Security Top 10, ML03.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS, AML.T0024 Exfiltration via ML Inference API.
  <https://atlas.mitre.org/techniques/AML.T0024>
- NIST AI Risk Management Framework (AI RMF 1.0), MEASURE 2.7.
  <https://www.nist.gov/itl/ai-risk-management-framework>
