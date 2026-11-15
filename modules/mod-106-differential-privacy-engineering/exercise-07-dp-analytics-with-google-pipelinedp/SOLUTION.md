# Exercise 07 — DP Analytics with Google PipelineDP — Reference Solution

## 1. Solution overview

This is an **implementation exercise** on *DP analytics*: producing
differentially private aggregate statistics (counts, sums, means,
privacy-id-counts) from a large event dataset using **PipelineDP**
(co-developed by Google and OpenMined, backed by the Google Differential
Privacy C++ library).

The learner must:

1. Model the data in PipelineDP's `(privacy_id, partition_key, value)`
   abstraction.
2. Choose a per-partition DP aggregation (`count`, `sum`, `mean`,
   `privacy_id_count`, `variance`, `quantiles`) and supply the
   contribution bounds that PipelineDP's DP sensitivity analysis
   requires.
3. Run the pipeline on a local collection (and show the same code
   working against Spark / Beam under a flag).
4. Publish a release manifest that an auditor can read without
   knowing PipelineDP internals.

Assessable outputs:

- A **script** that computes at least one DP aggregation per supported
  type on a worked dataset.
- A **release manifest** documenting ε, δ, neighbouring relation,
  unit of privacy, per-metric contribution bounds, and the partition
  strategy (public vs. private partitions).
- A **design note** choosing between PipelineDP, PyDP, and SmartNoise
  SQL for this workload.

PipelineDP is maintained as the pipeline-oriented front end to the
Google DP library; its native differentiators are (a) first-class
partition-selection DP (for "which partitions exist" when the key set
is private), and (b) runners for local / Beam / Spark without changing
the pipeline code.

## 2. Worked implementation

### 2.1 Sample script — DP counts and means over a user-event log

```python
# dp_analytics.py
import pipeline_dp
from pipeline_dp import NaiveBudgetAccountant

# Running example: a log of user interactions.
# Each record: Event(user_id="u123", item_id="A", session_s=42.0, date="2026-11-15")

def run(events, backend=pipeline_dp.LocalBackend()):
    # ---- 1. Total budget. Split across the two metrics we publish. ----
    budget_accountant = NaiveBudgetAccountant(total_epsilon=2.0, total_delta=1e-6)

    dp_engine = pipeline_dp.DPEngine(budget_accountant, backend)

    # ---- 2. Data extractors. The three roles PipelineDP needs. ----
    extractors = pipeline_dp.DataExtractors(
        privacy_id_extractor=lambda e: e.user_id,
        partition_extractor=lambda e: e.item_id,
        value_extractor=lambda e: e.session_s,
    )

    # ---- 3. DP COUNT of sessions per item (private partitions). ----
    count_params = pipeline_dp.AggregateParams(
        noise_kind=pipeline_dp.NoiseKind.LAPLACE,
        metrics=[pipeline_dp.Metrics.COUNT, pipeline_dp.Metrics.PRIVACY_ID_COUNT],
        max_partitions_contributed=5,           # per-user cap on item_ids
        max_contributions_per_partition=50,     # per-user cap of sessions per item
        # No public_partitions → PipelineDP uses DP partition selection
        # (Thresholding / Gumbel) so the released key set is itself DP.
    )
    count_result = dp_engine.aggregate(events, count_params, extractors)

    # ---- 4. DP MEAN session length per item (public partitions). ----
    item_catalog = list_of_known_public_items()  # must be independent of the private data
    mean_params = pipeline_dp.AggregateParams(
        noise_kind=pipeline_dp.NoiseKind.GAUSSIAN,
        metrics=[pipeline_dp.Metrics.MEAN],
        max_partitions_contributed=5,
        max_contributions_per_partition=50,
        min_value=0.0,                           # value clip; public knowledge
        max_value=1800.0,                        # 30 minutes; public policy max
    )
    mean_result = dp_engine.aggregate(
        events, mean_params, extractors,
        public_partitions=item_catalog,
    )

    # ---- 5. Compute budgets and materialise the output. ----
    budget_accountant.compute_budgets()

    return list(count_result), list(mean_result)
```

Design notes a reviewer verifies:

- **`privacy_id` ≠ `partition`.** `privacy_id` is the unit of privacy
  (user, device, session); `partition` is the group-by key. A record's
  presence/absence is removed at the `privacy_id` level, not at the
  partition level.
- **`NaiveBudgetAccountant(total_epsilon, total_delta)` is global.**
  The budget is split across the aggregations by the number and weight
  of metric declarations; the accountant is called with
  `compute_budgets()` before the pipeline materialises. A pipeline
  that runs twice doubles the ε spend against this corpus.
- **Contribution bounds are mandatory.**
  `max_partitions_contributed` (how many distinct keys one user shows
  up in) and `max_contributions_per_partition` (how many rows one user
  contributes to one key) together bound the per-user L-infinity
  sensitivity. Values set too low discard real data; values set too
  high inflate DP noise. These two numbers are *the* operational
  controls.
- **Public vs. private partitions.** If the partition keys are public
  knowledge (a known catalog of SKUs, published zip codes), set
  `public_partitions=...` to avoid spending ε on partition selection.
  If the keys themselves are sensitive, omit it and let PipelineDP
  run its DP partition-selection mechanism.
- **`NoiseKind`.** Laplace for pure ε-DP; Gaussian for (ε, δ)-DP with
  sometimes-tighter composition over many metrics. MEAN uses Gaussian
  because it composes two internal queries (sum and count) and
  benefits from the Gaussian mechanism's composition story.
- **Value clipping (`min_value`, `max_value`) is public knowledge.**
  Derive these from a published policy or a prior DP release; never
  from the dataset's own min/max.

### 2.2 Release manifest

```json
{
  "release_id": "sessions-per-item-2026-11-15",
  "mechanism": "PipelineDP (Google DP library); NaiveBudgetAccountant",
  "library_versions": {
    "pipeline_dp": "<pin>",
    "google_dp": "<pin>"
  },
  "total_epsilon": 2.0,
  "total_delta": 1e-6,
  "neighbouring_relation": "add/remove one user",
  "unit_of_privacy": "user (user_id)",
  "aggregations": [
    {
      "name": "sessions_per_item_count",
      "metric": ["COUNT", "PRIVACY_ID_COUNT"],
      "noise_kind": "LAPLACE",
      "max_partitions_contributed": 5,
      "max_contributions_per_partition": 50,
      "partition_strategy": "private (DP partition selection)",
      "epsilon_share": 1.0,
      "delta_share": 5e-7
    },
    {
      "name": "mean_session_length_per_item",
      "metric": ["MEAN"],
      "noise_kind": "GAUSSIAN",
      "value_clip": {"min": 0.0, "max": 1800.0},
      "clip_source": "policy: SESSION_CAP = 30 minutes (public)",
      "max_partitions_contributed": 5,
      "max_contributions_per_partition": 50,
      "partition_strategy": "public (catalog of known items)",
      "epsilon_share": 1.0,
      "delta_share": 5e-7
    }
  ],
  "dataset_size_N": 4800000,
  "delta_vs_N_check": "1e-6 << 1/4800000 ≈ 2.08e-7 — fails: tighten delta to 1e-8",
  "notes": "Clip source = public policy; partition catalog loaded from public SKU list hash <sha>."
}
```

The *fails: tighten delta* annotation is intentional — the exercise
should force the learner to notice that the published δ ≪ 1/N rule
sometimes requires revisiting δ after seeing N. A real submission
either tightens δ or justifies why N here is not the right denominator
(e.g., user count is smaller than row count).

### 2.3 Choosing PipelineDP vs. the alternatives (design note outline)

| Tool | Best at | Reason not to pick |
|---|---|---|
| **PipelineDP** | Distributed analytics (Beam/Spark), DP partition selection, user-level contribution bounds | Not an SQL surface; requires Python glue code |
| **PyDP** (Python bindings for Google DP) | Single-process DP aggregations in notebooks | No pipeline runner, no partition selection abstraction |
| **SmartNoise SQL** (OpenDP) | Analysts writing SQL against a DP-aware engine | Less flexible for per-user contribution bounds over complex event logs |
| **Chorus / dp-sqlite** | Research / historical | Not production-maintained |

Pick PipelineDP when the workload is a data-pipeline aggregation on a
batch event log and you need user-level DP without rewriting SQL.

## 3. Validation steps

1. **`privacy_id` extraction returns the correct unit.** A grep of the
   extractor shows it returns `user_id`, not `event_id` or
   `request_id`. The DP guarantee is only as strong as this extractor
   is honest.
2. **Contribution bounds are set and visible.** `max_partitions_contributed`
   and `max_contributions_per_partition` are not left at library defaults;
   both are in the manifest with a sourced rationale.
3. **Value clips sourced.** For SUM / MEAN / VARIANCE / QUANTILES,
   `min_value` and `max_value` are present and traced to a published
   source.
4. **Partition strategy matches reality.** If `public_partitions` is
   provided, confirm the catalog is in fact public (independent of the
   private data). If omitted, confirm the pipeline logs DP partition
   selection activity.
5. **Budget consumed matches the budget granted.** Call
   `budget_accountant.compute_budgets()` and confirm the sum of
   per-metric ε equals `total_epsilon`.
6. **δ ≪ 1/N check.** Run it, print it. If it fails, either tighten δ
   or state why 1/N is the wrong denominator (e.g., N is the user count,
   and user count < row count).
7. **Determinism of the ε accounting.** Running the pipeline twice at
   the same configuration produces the same ε spend per metric. The
   data-dependent *values* change (they contain noise); the ε
   accounting does not.
8. **Negative test.** Flip `max_contributions_per_partition` from 50 to
   1 and re-run; the noise relative to a count should visibly drop
   (sensitivity went down), at the cost of truncating real data.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Correct `privacy_id`, `partition_key`, `value` extractors | 1 | — |
| Contribution bounds (partition × per-partition) set deliberately | 2 | Both numbers in manifest; both justified |
| Partition strategy chosen (public vs. private) matches reality | 1 | — |
| Value clips sourced from public knowledge, not the private data | 1 | SUM/MEAN/VARIANCE require this |
| `NaiveBudgetAccountant` instantiated with explicit total ε and δ | 1 | — |
| `compute_budgets()` called and per-metric spend logged | 1 | — |
| δ ≪ 1/N checked and resolved (tightened or justified) | 1 | — |
| Release manifest lists library versions, neighbouring relation, unit of privacy | 2 | Reviewer can reproduce |

Automatic failures:

- Using row IDs as `privacy_id` when a user contributes multiple rows.
- Omitting contribution bounds for any aggregation.
- Deriving value clips from the data (`df.min()` / `df.max()`).
- Running the pipeline without calling `compute_budgets()`.
- Claiming public partitions without publishing the key set.

## 5. Common mistakes

- **Mistaking row-level DP for user-level DP.** If a user appears in
  100 rows and `privacy_id = event_id`, the release is row-level DP,
  not user-level. The guarantee is dramatically weaker.
- **Setting `max_partitions_contributed = ∞` implicitly.** Omitting
  the parameter in older PipelineDP versions used to default to a
  large number; the current versions require it. Either way, the
  value appears in the manifest.
- **Deriving value clips from the data.** Sensitivity analysis needs
  public bounds. Always publish the source of `min_value` and
  `max_value`.
- **Choosing Laplace for MEAN.** PipelineDP's MEAN decomposes into
  SUM / COUNT and composes better with Gaussian noise under (ε, δ).
  Laplace works but typically needs a larger ε for the same accuracy.
- **Treating public_partitions as a performance knob.** The choice
  changes the *mechanism* (partition selection vs. not). It is a DP
  decision, not a tuning knob.
- **Running the same pipeline twice against the same corpus.** The
  budgets compose; a second run doubles the ε spend. If the pipeline
  is idempotent, cache the result and do not re-execute.
- **Shipping the pipeline's checkpointing artifacts.** Intermediate
  state (pre-noise sums, partition-selection internals) is inside the
  DP boundary; only the final DP outputs are safe to release.
- **Using PipelineDP for per-record *predictions*.** PipelineDP is for
  aggregates. For per-record DP labelling, use PATE (Exercise 05).

## 6. References

- Google / OpenMined PipelineDP.
  <https://pipelinedp.io/>
  Source: <https://github.com/OpenMined/PipelineDP>
- Google Differential Privacy library (C++ core, used by PipelineDP).
  <https://github.com/google/differential-privacy>
- Wilson, R.; Zhang, C. Y.; Lam, W.; Desfontaines, D.; Simmons-Marengo,
  D.; Gipson, B. *Differentially Private SQL with Bounded User
  Contribution.* PoPETs 2020 (contribution-bounding theory behind
  PipelineDP).
  <https://arxiv.org/abs/1909.01917>
- Desfontaines, D.; Pejó, B. *SoK: Differential Privacies.* PoPETs 2020
  (comparison of DP variants useful when choosing between tools).
  <https://arxiv.org/abs/1906.01337>
- OpenDP SmartNoise SQL (alternative tool compared in the design note).
  <https://github.com/opendp/smartnoise-sdk>
- PyDP (alternative tool compared in the design note).
  <https://github.com/OpenMined/PyDP>
- NIST Special Publication 800-226, *Guidelines for Evaluating
  Differential Privacy Guarantees* (final, 2025).
  <https://csrc.nist.gov/pubs/sp/800/226/final>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
