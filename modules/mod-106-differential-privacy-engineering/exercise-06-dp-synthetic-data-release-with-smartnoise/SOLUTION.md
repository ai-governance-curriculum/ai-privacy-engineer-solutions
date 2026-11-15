# Exercise 06 — DP Synthetic Data Release with SmartNoise — Reference Solution

## 1. Solution overview

This is an **implementation + release-engineering exercise**. The
learner must produce a **differentially private synthetic dataset** from
a tabular source using OpenDP's SmartNoise Synth library, package the
output as a reviewable release, and defend the fidelity-privacy trade-off.

Assessable outputs:

1. A **generation script** that invokes a SmartNoise Synth synthesiser
   (MWEM/PGM, AIM, PATE-CTGAN, or MST — each has trade-offs documented
   below) at an explicit (ε, δ) budget.
2. A **release package** consisting of: the synthetic data, the metadata
   schema, the privacy manifest, and a utility report comparing a short
   set of marginals and queries against the private source.
3. A **design note** justifying the synthesiser choice given the data
   shape (dimensionality, cardinality, correlation structure).

The code below targets SmartNoise Synth, which is the OpenDP-maintained
library for DP synthetic data; the design decisions are
synthesiser-agnostic.

## 2. Worked implementation

### 2.1 Generation script

```python
# synth_release.py
import pandas as pd
from snsynth import Synthesizer         # smartnoise-synth entry point

EPSILON = 3.0
DELTA = 1e-6                            # must be << 1/N
RELEASE_SIZE = None                     # None → same count as source

def build_metadata(df: pd.DataFrame) -> dict:
    """Column-level metadata required by MST/AIM/MWEM-PGM.

    For each column: type, categories (for discrete), and clipping bounds
    (for continuous). These bounds are PUBLIC prior knowledge — never
    derived from min/max of the private data without a DP release of
    those bounds first, since min/max are non-DP.
    """
    return {
        "age":        {"type": "int",         "lower": 18,  "upper": 95},
        "zip3":       {"type": "categorical", "categories": [f"{i:03d}" for i in range(1000)]},
        "sex":        {"type": "categorical", "categories": ["F", "M", "X"]},
        "diagnosis":  {"type": "categorical", "categories": ["A", "B", "C", "D", "other"]},
        "n_visits":   {"type": "int",         "lower": 0,   "upper": 50},
        "readmitted": {"type": "categorical", "categories": [0, 1]},
    }

def synthesize(private_df: pd.DataFrame, synth_name: str = "mst") -> pd.DataFrame:
    metadata = build_metadata(private_df)

    # SmartNoise Synth exposes a factory that picks a synthesiser from
    # a short name ("mwem_pgm", "aim", "mst", "patectgan", "dpctgan").
    synth = Synthesizer.create(synth_name, epsilon=EPSILON, delta=DELTA)

    synth.fit(
        private_df,
        preprocessor_eps=None,       # when None, the synth will spend a
                                     # small fixed share of epsilon on
                                     # preprocessing (bin edges, type
                                     # detection); documented in the
                                     # SmartNoise Synth tutorials.
        categorical_columns=[c for c, m in metadata.items() if m["type"] == "categorical"],
        continuous_columns=[c for c, m in metadata.items() if m["type"] == "int"],
    )

    n_release = RELEASE_SIZE or len(private_df)
    return synth.sample(n_release)
```

Why this script, and not a GAN trained on the raw data:

- **The synthesiser has to be DP end-to-end.** `Synthesizer.create(...)`
  attaches a DP budget to the fitting routine; the resulting sample is
  DP by post-processing. A regular `fit` on a non-DP CTGAN is **not** a
  DP release even if you call it "synthetic data."
- **Metadata bounds come from public knowledge or a prior DP release.**
  Auto-detecting bounds from the data (`df["age"].min()`,
  `df["age"].max()`) is a non-DP read; those reads leak and invalidate
  the privacy claim. Cite the source of each bound in the manifest.
- **Preprocessing eats budget.** Binning continuous columns, inferring
  which columns are categorical, and histogramming cardinalities are
  all DP computations. The SmartNoise Synth `preprocessor_eps` setting
  surfaces this; a learner who sets it to zero is almost certainly
  reading the data non-DP somewhere.
- **Release size is a design choice.** A larger sample does not reduce
  privacy loss (post-processing) but it also does not improve accuracy
  — it just resamples the learned distribution.

### 2.2 Synthesiser decision table

| Shape of source data | Recommended synthesiser | Why |
|---|---|---|
| Mostly categorical, moderate columns (≤ 30), strong correlations | **MST** (marginal-based, Private PGM) | Winner of the NIST 2018 Synthetic Data Challenge contributed-track line of work; good marginal fidelity at low ε |
| Many columns, including continuous, strong higher-order correlations | **AIM** (Adaptive and Iterative Mechanism) | Iteratively selects informative marginals; typically beats MWEM/MST on mixed data at the same ε |
| Classic baseline, low-dimensional | **MWEM-PGM** | Transparent, easy to explain in audit; slow in high dimensions |
| Imagery / text-like tabular / very wide continuous | **PATE-CTGAN** | GAN-based; acceptable utility when marginal-based methods degrade |
| High-cardinality categorical | **DP-CTGAN** or **AIM** with cardinality capping | Marginal explosion needs capping |

Pick one, record the rejection reason for the others in the design note.

### 2.3 Release package contents

```
release-2026-11-15/
├── data/
│   └── synthetic.parquet
├── metadata/
│   ├── schema.json              # column types, bounds, categories (public knowledge)
│   └── column-provenance.md     # where each bound came from
├── privacy/
│   ├── manifest.json            # the DP claim; see §2.4 below
│   ├── transcript.txt           # SmartNoise Synth run log, including preprocessor_eps
│   └── accountant.json          # per-step DP cost breakdown if the synth logs it
├── utility/
│   ├── marginals-1way.csv       # per-column distributions, private vs. synth
│   ├── marginals-2way.csv       # pairwise contingency tables on a short list
│   ├── targeted-queries.md      # the business queries the release must support
│   └── fidelity.md              # KS / TV distance for each marginal
└── README.md                    # audit-oriented overview
```

### 2.4 Privacy manifest (`privacy/manifest.json`)

```json
{
  "release_id": "synthetic-admissions-2026-11-15",
  "mechanism": "SmartNoise Synth MST (OpenDP)",
  "library_version": "smartnoise-synth==<pin>",
  "epsilon": 3.0,
  "delta": 1e-6,
  "neighbouring_relation": "replace one record (bounded DP)",
  "unit_of_privacy": "row (one hospital admission)",
  "dataset_size_N": 182340,
  "delta_vs_N_check": "1e-6 << 1/182340 ≈ 5.49e-6 — within guidance",
  "preprocessor_epsilon_share": 0.05,
  "generator_epsilon_share": 2.95,
  "post_processing_notice": "The released data is a sample from a fitted DP model; any analysis of this sample is itself DP by post-processing.",
  "release_size": 182340,
  "seed": 0
}
```

The manifest's job is to answer, in one page, the questions an auditor
or data-use committee will ask about the release.

## 3. Validation steps

1. **Schema honesty.** Every bound in `metadata/schema.json` is sourced
   in `column-provenance.md`. Bounds derived from the private data (not
   from public knowledge or a prior DP release) are a validation
   failure.
2. **Fit is DP.** Confirm the synthesiser was constructed with
   `epsilon` and `delta` set; a run log that reports an ε spend
   different from the manifest is a validation failure.
3. **Preprocessor cost accounted.** The manifest's
   `preprocessor_epsilon_share + generator_epsilon_share` must equal
   the stated total ε. A missing preprocessor share usually means the
   preprocessor ran non-DP.
4. **Marginal fidelity.** For each column, the 1-way marginal of the
   synthetic sample is close to the private 1-way marginal in TV
   distance. Thresholds are task-specific; publish the numbers either
   way.
5. **Targeted-query fidelity.** The business queries the release is
   intended to support (listed in `utility/targeted-queries.md`) each
   return a plausible answer on the synthetic data; publish the delta
   against the (sealed) private answer for internal review.
6. **Downstream-model sanity.** Train the intended downstream model on
   the synthetic data; its held-out accuracy on *real* data should be
   within a declared tolerance of the model trained on the private data.
   If it collapses, the release is low-utility and either (a) ε is too
   small, (b) the synthesiser is wrong for the shape, or (c) the
   downstream model is latching onto DP noise.
7. **Membership inference baseline.** Run a membership-inference attack
   against the *private model* implied by the synthesiser (not against
   the synthetic data directly). The attacker's AUC should be close to
   0.5; a substantially higher AUC indicates implementation drift from
   the DP guarantee.
8. **δ sanity.** `delta < 1/N`; the explicit check is in the manifest.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Synthesiser chosen deliberately with rationale against alternatives | 2 | MST/AIM/MWEM-PGM/PATE-CTGAN each have trade-offs |
| Metadata schema with provenance (no non-DP reads of the private data for bounds) | 2 | — |
| Preprocessor ε and generator ε accounted separately and summed to total | 1 | — |
| Release package layout matches §2.3 (data, metadata, privacy, utility) | 1 | — |
| Fidelity report includes 1-way and some 2-way marginals, plus the targeted queries | 1 | — |
| Downstream-model sanity check present (synthetic-trained, real-tested) | 1 | — |
| δ ≪ 1/N recorded and checked numerically | 1 | — |
| Library versions pinned | 1 | — |

Automatic failures:

- Any bound in the schema sourced from `df.min()` / `df.max()` of the
  private data without a DP release of those bounds.
- "DP-GAN" used in the common sense — a GAN trained on raw data and
  labelled "synthetic." Only DP-fitted generators qualify.
- Omitting the preprocessor ε from the manifest.
- Reporting a single "it looks fine" fidelity statement instead of
  numerical marginals.
- Shipping the fit's training checkpoints alongside the sample —
  checkpoints are DP but the *release surface* includes only the
  sample.

## 5. Common mistakes

- **"Synthetic data is automatically private."** It is not. A GAN or
  copula fit on raw data is not DP; only generators trained with a DP
  mechanism (DP-SGD, DP marginal selection, etc.) produce DP releases.
- **Letting the preprocessor read the private data for free.** Binning
  edges, detected categorical sets, and cardinality counts are all
  computations on the private data. They must either consume budget or
  come from public prior knowledge.
- **Using min/max for clipping bounds.** `df["age"].min()`,
  `df["age"].max()` are non-DP summaries; using them for clipping leaks
  the extremes of the data.
- **Picking the synthesiser by familiarity.** CTGAN is well-known but
  often not the best DP synth; MST/AIM frequently outperform it on
  tabular data at moderate ε.
- **Measuring utility with "it looks similar."** Publish TV / KS
  distances for marginals and a small number of targeted queries; "it
  looks similar" fails under audit.
- **Resampling synthetic data to boost ε.** Drawing a larger sample does
  not reduce ε (post-processing); drawing a smaller sample does not
  increase ε either. The budget is spent on the fit, not on sampling.
- **Confusing bounded-DP with unbounded-DP sensitivity.** SmartNoise's
  synthesisers default to a particular convention; the manifest must
  name it and the mechanism must match.
- **Keeping the private source and the synthetic release in the same
  access boundary.** The *point* of the synth release is that the
  synthetic sample can cross the boundary the private data cannot.
  Store them in different access scopes and name the asymmetry.

## 6. References

- SmartNoise Synth (OpenDP) — library for DP synthetic data.
  <https://github.com/opendp/smartnoise-sdk>
- SmartNoise Synth documentation (tutorials and synthesiser list).
  <https://docs.smartnoise.org/synth/>
- McKenna, R.; Miklau, G.; Sheldon, D. *Winning the NIST Contest: A
  Scalable and General Approach to Differentially Private Synthetic
  Data.* arXiv:2108.04978 (MST / Private PGM).
  <https://arxiv.org/abs/2108.04978>
- McKenna, R.; Mullins, B.; Sheldon, D.; Miklau, G. *AIM: An Adaptive
  and Iterative Mechanism for Differentially Private Synthetic Data.*
  VLDB 2022.
  <https://arxiv.org/abs/2201.12677>
- Hardt, M.; Ligett, K.; McSherry, F. *A Simple and Practical Algorithm
  for Differentially Private Data Release.* NeurIPS 2012 (MWEM
  baseline).
  <https://arxiv.org/abs/1012.4763>
- Xie, L. et al. *Differentially Private Generative Adversarial Networks.*
  arXiv:1802.06739 (DP-CTGAN line of work; context only).
  <https://arxiv.org/abs/1802.06739>
- NIST Special Publication 800-226, *Guidelines for Evaluating
  Differential Privacy Guarantees* (final, 2025).
  <https://csrc.nist.gov/pubs/sp/800/226/final>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — used in the design note for
  framing non-DP residual risks around generative releases.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
