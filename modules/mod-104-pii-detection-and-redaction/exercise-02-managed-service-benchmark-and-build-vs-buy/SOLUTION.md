# Exercise 02 — Managed Service Benchmark and Build vs. Buy — Reference Solution

## 1. Solution overview

This is a **design exercise** that produces two deliverables:

1. A benchmark report comparing at least three PII detection options —
   typically one self-hosted (Microsoft Presidio) and two managed services
   (chosen from AWS Macie, Amazon Comprehend PII, Google Cloud Data Loss
   Prevention / Sensitive Data Protection, and Azure AI Language PII
   detection) — on a gold-labelled fixture the learner provides.
2. A build-vs-buy recommendation that is defensible to a product lead, a
   data-protection officer, and a VP Engineering, using the benchmark numbers
   plus a cost model and a constraint matrix.

The assessable artifact is a repository with:

- A `benchmarks/` directory containing (a) the gold fixture (synthetic or
  de-identified), (b) adapter scripts per provider that call each service
  through its official SDK, (c) a metrics harness that computes
  per-entity-type precision, recall, F1, and median latency, and (d) a
  `results.md` with the numbers and the methodology.
- A `decision.md` that writes down the build-vs-buy recommendation, the
  constraints that drove it, the one or two decision factors most likely to
  flip, and the re-evaluation trigger.

A learner who returns only `results.md` without `decision.md` has produced
half the exercise; a learner who returns only `decision.md` has produced
unsupported opinion. Both are required.

## 2. Worked implementation

### 2.1 Benchmark methodology

A credible benchmark requires five commitments, written down before the
first API call:

1. **Fixed gold fixture.** A single JSONL file with `(text, spans)` rows,
   locked by hash before any provider is scored. Target: ≥ 500 rows,
   covering the entity types the product actually needs. Drawn from the
   real corpus (de-identified or synthetic) — never from a provider's own
   demo text.
2. **Entity-type mapping table.** Every provider uses a different taxonomy.
   AWS Comprehend's `EMAIL` is Presidio's `EMAIL_ADDRESS` is GCP DLP's
   `EMAIL_ADDRESS` is Azure Language's `Email`. The learner commits a
   mapping table before running the harness; otherwise "recall was lower on
   Azure" is a taxonomy bug, not a model finding.
3. **Span-overlap scoring.** Score a prediction as true-positive when it
   overlaps the gold span with the correct mapped type. Exact-match scoring
   penalises providers that return different boundaries (e.g., including or
   excluding the `@` in an email address); this is an accounting issue, not
   a quality issue.
4. **Latency measured warm.** Cold-start p99 is a deployment property, not a
   model property. Discard the first 50 requests per adapter.
5. **One region per provider.** Cross-region comparison conflates network
   with model. If the deployment target is `us-east-1`, score every
   provider in or near that region.

### 2.2 Benchmark harness skeleton

```python
# benchmarks/run.py
"""Run one provider adapter against the gold fixture and dump metrics.

Each adapter module exposes `detect(text) -> List[Dict[type,start,end,score]]`.
Entity-type mapping is a dict from provider-native type -> canonical type
defined in `taxonomy.yaml`. The scorer is the span-overlap scorer shared
with exercise 01.
"""
import json, importlib, time, statistics
from collections import defaultdict


def run(adapter_name: str, fixture_path: str, taxonomy: dict) -> dict:
    adapter = importlib.import_module(f"adapters.{adapter_name}")
    tp = defaultdict(int); fp = defaultdict(int); fn = defaultdict(int)
    latencies = []
    with open(fixture_path) as f:
        for i, line in enumerate(f):
            row = json.loads(line)
            t0 = time.perf_counter()
            preds = adapter.detect(row["text"])
            latencies.append(time.perf_counter() - t0)
            if i < 50:              # discard warm-up
                continue
            canon = [{"type": taxonomy.get(p["type"], p["type"]),
                       "start": p["start"], "end": p["end"]} for p in preds]
            score(canon, row["spans"], tp, fp, fn)
    return build_report(tp, fp, fn, latencies)
```

Each adapter is thin: it authenticates, calls the service's documented
detection endpoint, and normalises the response. The point of the exercise
is the methodology, not the SDK glue.

### 2.3 Build-vs-buy comparison matrix

The following is the **structure** the learner must fill in. Numeric values
in the learner's submission must come from their benchmark, not from this
reference. Qualitative cells cite the provider's official documentation and
the provider's SOC 2 / ISO 27001 / HIPAA BAA status as of the submission
date.

| Dimension | Self-host (Presidio) | AWS Comprehend PII / Macie | GCP Sensitive Data Protection (DLP) | Azure AI Language PII |
|---|---|---|---|---|
| Deployment model | You run it | Managed API | Managed API | Managed API |
| Per-entity precision / recall on *your* fixture | from benchmark | from benchmark | from benchmark | from benchmark |
| Entity-type coverage (count, overlap with product need) | Built-in list + custom recognisers | Service-defined list | Service-defined list ("infoTypes") | Service-defined list |
| Custom entity support | First-class (regex + ML) | Comprehend Custom Entity Recognition | Custom infoTypes (regex, dictionary, hotword) | Custom NER (Language Studio) |
| Sends data off-prem | No | Yes | Yes | Yes |
| Data-residency controls | You choose | Region pin, see provider's documented defaults | Region pin, see provider's documented defaults | Region pin, see provider's documented defaults |
| HIPAA eligibility | N/A — you control the environment | Requires BAA and eligible-services list | Requires BAA and eligible-services list | Requires BAA and eligible-services list |
| Pricing model | Infra + staff | Per character / per request | Per unit (bytes / records) | Per character |
| Steady-state latency (p50 / p95) | from benchmark | from benchmark | from benchmark | from benchmark |
| Operational burden | spaCy/transformer updates, scaling, on-call | Provider | Provider | Provider |
| Lock-in risk | Low (OSS) | Moderate | Moderate | Moderate |

The cells marked "from benchmark" and the BAA / eligibility cells must have
a timestamp and a link to the provider's canonical documentation page
(e.g., AWS's "HIPAA Eligible Services" page, GCP's "HIPAA Compliance",
Azure's "Compliance offerings") because eligibility is a moving target.

### 2.4 Decision framework

The build-vs-buy decision is dominated by four factors, in roughly this
order. The learner must argue each one explicitly.

1. **Can you legally send the data off-prem to this region?** If the data is
   subject to a sector rule that forbids external processing or if the
   service is not in the provider's HIPAA / FedRAMP / UK G-Cloud eligible
   list for the needed region, the managed option is removed from the
   shortlist. This is a gate, not a trade-off.
2. **Is accuracy on *your* data within an acceptable band?** Managed
   services optimise for the mean customer. If the fixture shows recall on
   the one entity type that matters most is 20 points lower than Presidio
   with a custom recogniser, no amount of operational savings closes that
   gap — you will add a second detector on top anyway.
3. **What is the three-year TCO at expected volume?** Build the two-sided
   cost model: managed cost at volume (per-request × requests, plus egress)
   vs. self-host cost (compute, model licensing, engineering time to own
   the recogniser registry and pager rotation). TCO under 3× staff cost
   almost always favours buy; TCO 10× staff cost often flips to build at
   scale. A learner who claims "it's cheaper to run our own" without a
   written cost model fails this criterion.
4. **What is the fastest path to a working control?** A privacy incident is
   a date, not an accuracy number. If the DPIA says "redaction in place by
   Q-end," a managed service is in service in days; a self-host rollout is
   in service in weeks. Weigh this against the opposite direction: a bad
   managed decision can take months to migrate off of.

### 2.5 Worked recommendation (template)

> **Context.** Our product processes ~50 M characters/day of free-text user
> messages in `us-east-1`. The regulated subset is HIPAA PHI (~5%). The
> rest is general PII. The need is to redact both before training and
> before storing in long-term logs.
>
> **Decision.** Use a **hybrid**. Call AWS Comprehend PII (operating under
> our BAA, in the HIPAA-eligible region) for the PHI-tagged stream in the
> ingestion path. Use self-hosted Presidio with custom recognisers for
> product-internal identifier types that no managed service covers. Apply
> self-hosted Presidio as a second-pass redactor on top of the managed
> output to catch types absent from Comprehend's taxonomy.
>
> **Why not pure build?** Our benchmark shows Comprehend's recall on
> `PERSON`, `ADDRESS`, `PHONE`, `EMAIL`, `SSN` matches or exceeds our
> Presidio configuration, and the engineering team needed to maintain
> parity on these generic entities is larger than the reduction in cloud
> cost.
>
> **Why not pure buy?** Our product-internal identifiers (`CASE-####-XX`,
> `EID-#######`) are not supported by any managed service's built-in
> taxonomy. Custom entity training in the managed products is possible but
> introduces an annotation + retraining cycle we would rather own.
>
> **Flip triggers.** We revisit the decision when: (a) our daily character
> volume crosses a threshold at which the managed per-char cost exceeds
> the amortised cost of self-hosting plus staff, (b) Comprehend recall on
> our gold fixture drops below our stated threshold on a quarterly
> re-benchmark, or (c) a data-residency change moves our workload to a
> region where the managed service is not in the HIPAA-eligible list.

## 3. Validation steps

1. **Fixture lock.** Produce a SHA-256 of `benchmarks/fixtures/gold.jsonl`
   and record it in `results.md`. All adapters must have been scored
   against the same hash.
2. **Taxonomy mapping committed first.** `taxonomy.yaml` must have the same
   commit timestamp as or earlier than the first adapter run (check
   `git log --follow`). Mapping tables invented after seeing results are
   motivated reasoning.
3. **Per-entity-type report.** Every provider report prints rows for each
   entity type the product needs, including rows where the provider
   returned zero predictions (recall = 0 is a result, not a gap).
4. **Cost model reproducibility.** The TCO spreadsheet / script accepts
   volume and region as inputs and recomputes managed cost from the
   provider's published rate as of a dated snapshot. Staff cost is derived
   from a stated FTE assumption.
5. **Legal gate check.** `decision.md` contains a row per candidate
   provider recording whether the specific service is in the provider's
   HIPAA / FedRAMP / PCI eligibility list for the target region, with a
   dated link to the provider's canonical eligibility page.
6. **Flip triggers.** The recommendation lists at least three conditions
   that would reopen the decision; each is phrased as a measurable event,
   not a feeling.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Gold fixture is real / synthetic customer data, not provider demos, with a locked hash | 2 | Fail if `fixture_url` is a provider sample |
| At least one self-host and at least two managed services benchmarked | 1 | Comparing two managed services only is a tie, not a decision |
| Per-entity precision / recall / F1 reported with latency | 2 | Aggregate-only loses the point; latency must be warm |
| Taxonomy mapping committed before results | 1 | Verifiable via `git log` |
| Written cost model with inputs, rates, and an FTE assumption | 1 | Opinion without math fails |
| Legal / residency gate applied explicitly per candidate | 1 | Eligibility links dated |
| Recommendation names a specific vendor (or hybrid) with named flip triggers | 1 | "It depends" without triggers fails |
| Discussion of lock-in and the migration-off cost | 1 | Even for the recommended option |

Automatic failures:
- Benchmark results reproduce the provider's published marketing numbers
  with no local fixture.
- Build-vs-buy conclusion contradicts the numbers without an argued reason.
- No mention of data-residency / sector-eligibility; the recommendation is
  not legally executable.
- Recommendation is "always build" or "always buy" independent of volume
  and data class.

## 5. Common mistakes

- **Scoring providers on different fixtures.** "AWS was 92% on a 2 k
  fixture and GCP was 89% on a 500-row fixture" is not a comparison.
- **Entity-type taxonomy drift.** Counting GCP `EMAIL_ADDRESS` as the same
  entity as Azure `Email` *implicitly*, without a mapping table, inflates
  or deflates recall for one provider for free.
- **Assuming sector eligibility is a property of a provider.** Eligibility
  is per-service, per-region, and sometimes per-feature. AWS's HIPAA list,
  GCP's HIPAA-covered products, and Azure's compliance offerings each
  publish a dated canonical list; cite the specific service.
- **Omitting the custom-entity dimension.** Managed services can learn
  custom entities, but each has a different data, label, and iteration
  model. A build-vs-buy that omits custom types is out of date the moment
  Product adds a new identifier.
- **Underpricing self-host.** The "free Presidio" column rarely includes
  the on-call rota, model-update cycle, custom-recogniser governance, or
  the cost of reproducing managed-service compliance attestations.
- **Overpricing managed.** Teams quote list-price per character without
  acknowledging batch mode, committed-use discounts, or the egress savings
  from co-locating with other workloads already in the provider.
- **Benchmarking once.** Managed models improve. A build-vs-buy decision
  made against a model from 18 months ago may already be wrong; the
  recommendation must schedule a re-benchmark.

## 6. References

- Microsoft Presidio. <https://microsoft.github.io/presidio/>
- Amazon Comprehend — PII detection documentation.
  <https://docs.aws.amazon.com/comprehend/latest/dg/pii.html>
- Amazon Macie documentation. <https://docs.aws.amazon.com/macie/>
- AWS HIPAA Eligible Services Reference.
  <https://aws.amazon.com/compliance/hipaa-eligible-services-reference/>
- Google Cloud Sensitive Data Protection (Cloud DLP).
  <https://cloud.google.com/sensitive-data-protection/docs>
- Google Cloud compliance (HIPAA and other frameworks).
  <https://cloud.google.com/security/compliance>
- Azure AI Language — PII detection.
  <https://learn.microsoft.com/azure/ai-services/language-service/personally-identifiable-information/overview>
- Microsoft compliance offerings. <https://learn.microsoft.com/compliance/>
- NIST AI Risk Management Framework (AI RMF 1.0) — used to frame "manage"
  and "measure" responsibilities against which the vendor split is
  evaluated. <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — risk context for depending on a
  third-party model whose training data and update cadence are opaque.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
