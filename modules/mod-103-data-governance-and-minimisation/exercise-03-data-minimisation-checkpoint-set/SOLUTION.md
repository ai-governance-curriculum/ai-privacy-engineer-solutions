# Exercise 03 — Data Minimisation Checkpoint Set — Reference Solution

## 1. Solution overview

This is a **design + implementation exercise**. "Data minimisation" in
GDPR Art. 5(1)(c) says personal data must be "adequate, relevant and
limited to what is necessary in relation to the purposes for which they
are processed." That is a conclusion, not a control. An engineering
contract makes it a conclusion you can defend: a fixed set of **named
checkpoints** across the ML lifecycle where a reviewer or an automated
gate asks "what is the minimum here, and are we meeting it?"

The assessable output is a **checkpoint set** — a numbered list of six
gates that cover the lifecycle from collection to retirement. For each
gate the learner specifies:

- what the gate asks (the question the gate exists to force),
- the artifact it is evaluated against (schema, manifest, dashboard),
- the owner (who signs the gate),
- the enforcement (CI check, review, policy-as-code rule),
- the pass/fail criteria, and
- what the gate must *not* try to do (so gates stay narrow and
  composable).

A checkpoint set without enforcement is a policy; a checkpoint set
without named owners is a wish. Both fail this exercise.

## 2. Worked answer

### 2.1 The six-gate set

The minimum lifecycle coverage is six gates. Numbering is deliberate:
earlier gates catch cheaper mistakes, later gates catch the drift the
earlier ones let through.

#### G1 — Collection-schema gate

- **Question:** For each field being collected, what is the specific
  purpose that *requires* this field, and is a less-identifying form
  sufficient (aggregate, pseudonymous, truncated)?
- **Artifact:** `collection_schemas/<activity_id>.yaml` listing fields,
  types, purpose_id references, less-identifying alternatives
  considered, and the reviewer who approved.
- **Owner:** Privacy engineering (sign); Product (co-sign for user-
  facing fields); Data engineering (co-sign for warehouse schema).
- **Enforcement:** CI rule rejects a schema PR without a `rationale`
  for every field and a `less_identifying_considered` note where the
  field is personal data; rejects fields with `retention_days` blank.
- **Pass/fail:** PASS if every personal-data field has (a) a cited
  purpose from the active RoPA, (b) an explicit alternative-considered
  note, (c) a numeric retention, (d) a sensitive/non-sensitive tag.
  FAIL otherwise.
- **Not for:** training-time filtering (that is G4), consent-state
  checking (that is G0 in the propagation contract — exercise 02).

#### G2 — Storage-shape gate

- **Question:** Once in storage, is the record kept in the least
  identifying form adequate for the purpose? Are the join keys to
  identity isolated from the processing surface?
- **Artifact:** Data-catalog entry with per-column classification,
  encryption key id, and the pseudonymisation boundary (which service
  holds the mapping, which roles can call it).
- **Owner:** Data platform; co-signed by privacy engineering for any
  `sensitive=true` column.
- **Enforcement:** policy-as-code rule in the catalog: `sensitive=true`
  columns MUST use a sensitive-class KMS key and MUST NOT be colocated
  with the pseudonym mapping in the same role scope. CI blocks a
  catalog apply that violates either.
- **Pass/fail:** PASS if (a) classification is populated, (b) KMS key
  class matches classification, (c) pseudonymisation boundary is named
  for every table that stores personal data.
- **Not for:** access review cadence (that is a security/G2 adjacent
  concern governed elsewhere).

#### G3 — Feature-store gate

- **Question:** For every feature materialised from personal data, is
  the purpose narrower or equal to the purpose authorised at
  collection? Is the feature definition free of columns that were not
  authorised for this purpose?
- **Artifact:** Feature definition file (`features/*.yaml`) with
  `upstream_tables`, `upstream_columns`, `purpose_id`, and `owner`.
- **Owner:** ML platform (feature author); co-sign by privacy
  engineering when the feature joins any `sensitive=true` column.
- **Enforcement:** build-time check walks `upstream_columns` against
  the collection-schema `purpose_id` set; feature `purpose_id` must be
  a subset (not a superset) of the authorised purposes of each source
  column.
- **Pass/fail:** PASS when the purpose subset relation holds and every
  column has an explicit entry. FAIL on any column not covered by the
  union of authorised purposes.
- **Not for:** inference-time use. Serving the feature is G6.

#### G4 — Training-corpus build gate

- **Question:** Does the materialised corpus include only subjects with
  active consent / basis for *this* training activity, only columns
  authorised for *this* activity, and only time ranges inside the
  retention window?
- **Artifact:** Corpus manifest (`datasets/<id>/manifest.yaml`) with
  the columns, the `activity_id`, the `basis_filter_commit` (which
  snapshot of the consent state was used), the time window, and the
  deduplication / redaction configuration.
- **Owner:** ML platform; co-sign by privacy engineering.
- **Enforcement:** CI job replays the manifest against the current
  consent/basis store; the corpus must pass (a) consent-delta ≤ the
  activity's drift threshold, (b) column set ⊆ features authorised at
  G3, (c) time window within retention, (d) redaction report attached
  and residual-leakage sample ≤ threshold.
- **Pass/fail:** PASS on all four. FAIL on any.
- **Not for:** fairness or bias analysis (handled by the EU AI Act
  Article 10 artifacts in mod-101 exercise 05 and here referenced only
  for cross-check).

#### G5 — Inference-input gate

- **Question:** For every inference call, is the input free of fields
  the model did not need at training time? Are user-identifying
  elements removed from the prompt before it reaches the model
  provider (especially where the provider is a third-party processor)?
- **Artifact:** Serving manifest (`services/<name>/serving.yaml`) with
  the `allowed_input_fields`, the pre-serve redaction config, and the
  processor / provider register row.
- **Owner:** ML platform; co-sign by privacy engineering for any
  third-party processor hop.
- **Enforcement:** request-schema validation at the service boundary;
  a redaction layer that strips fields not in `allowed_input_fields`
  and applies the inference-time scrub pipeline; audit event on every
  stripped field.
- **Pass/fail:** PASS if 100% of requests pass the schema, 0% include
  a non-allowed field at the provider boundary, and the audit has a
  bounded fallout rate (e.g., `<0.1%` requests with scrub hits over
  7 days — threshold set per activity).
- **Not for:** consent revocation at inference — that is handled by
  the propagation contract (G0 in exercise 02).

#### G6 — Log + retention gate

- **Question:** For every log line, event, and retained artifact (model
  prompts, completions, traces), is the retention period the minimum
  that the stated purpose requires? Is the log pipeline itself a
  distinct `activity_id` when used for anything beyond operations?
- **Artifact:** Log-retention policy per log class; TTL config on the
  underlying store; separate `activity_id` row in the RoPA for any
  log class fed into training or analytics.
- **Owner:** Observability + privacy engineering jointly; DPO sign-off
  on retention changes.
- **Enforcement:** TTL job runs and reports monthly per class; alert
  on classes with no TTL or with a mismatch between configured and
  observed; alert on logs ingested by analytics or training without a
  matching RoPA row.
- **Pass/fail:** PASS if every log class has a TTL, the observed TTL
  matches, and no training/analytics pipeline reads a log class that
  lacks its own RoPA row.
- **Not for:** security incident retention — security logs often have
  longer retention under a different lawful basis and belong in a
  separate row of the RoPA, not under this gate's SLA.

### 2.2 Worked implementation — the gates as a CI pipeline

The six gates become six CI jobs under a single `data-minimisation`
workflow. A toy pipeline (illustrative):

```yaml
# .github/workflows/data-minimisation.yml
name: data-minimisation
on:
  pull_request:
    paths:
      - "collection_schemas/**"
      - "catalog/**"
      - "features/**"
      - "datasets/**/manifest.yaml"
      - "services/**/serving.yaml"
      - "log_classes/**"

jobs:
  g1-collection-schema:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: tools/minimisation/g1-schema-check

  g2-storage-shape:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: tools/minimisation/g2-catalog-check

  g3-feature-store:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: tools/minimisation/g3-feature-purpose-subset

  g4-training-corpus:
    runs-on: ubuntu-latest
    needs: [g3-feature-store]
    steps:
      - uses: actions/checkout@v4
      - run: tools/minimisation/g4-corpus-replay

  g5-inference-input:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: tools/minimisation/g5-serving-manifest-check

  g6-log-retention:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: tools/minimisation/g6-log-ttl-and-activity-check
```

Each `tools/minimisation/gN-*` script fails the CI run with a specific
error message that identifies the file and field. The error message is
the artifact a reviewer uses to argue a fix, not a "privacy failed"
banner.

### 2.3 Rationale: why six gates, not three and not twenty

- **Why not three?** Because the drift between collection and training
  is where most minimisation failures occur, and three gates cannot
  separate feature-store joins from corpus builds from serving. A
  corpus can be minimised while a feature store silently exposes
  columns that leak back to it; three gates will not see that.
- **Why not twenty?** Because each gate must have an owner, an
  artifact, and a CI job. Twenty gates mean twenty owners, twenty
  files, and twenty review loops. Anything beyond six becomes a
  policy document rather than a set of enforceable gates.
- **Why these six, in this order?** The order matches the lifecycle.
  G1 refuses garbage at the door; G2 defines the shape in storage; G3
  prevents feature-store scope creep; G4 catches drift between
  collection and training; G5 bounds the inference request; G6 bounds
  the retention and prevents log-to-training silent feeds.

### 2.4 Worked example — a failing PR and its fix

A product PR adds a new field `user_locale_city` to the collection
schema for the support chatbot. G1 fails:

```
G1 FAIL collection_schemas/assistant-training-eu.yaml
field: user_locale_city
  - missing rationale citing a purpose from the RoPA
  - less_identifying_considered is blank
  - retention_days is blank
```

The fix is one of:

- Remove the field. The team's purpose statement does not require
  city-level location; postal-region or country already covers the
  need.
- Downgrade to `user_locale_country` with a documented rationale.
- Keep `user_locale_city`, add a rationale citing a specific
  prediction task that needs it, add `less_identifying_considered:
  "country tried first, insufficient for X"`, and add
  `retention_days: 90`.

In each case the PR is now auditable. The CI gate did not make the
decision — it forced the decision to be visible.

## 3. Validation steps

1. **Completeness check.** All six gates exist as code (not prose).
2. **Named owner per gate.** Each gate has exactly one primary owner
   and at most one co-signer.
3. **Artifact-per-gate check.** Open each gate's artifact — a reviewer
   should be able to read it without asking anyone.
4. **Failure-is-specific check.** Pick a gate, deliberately fail it,
   and confirm the error message names the file and the field. A gate
   whose failure is "violates minimisation" is not actionable.
5. **Purpose-subset check.** Walk a feature from G3 back to G1 and
   confirm the feature's `purpose_id` is a subset (never a superset)
   of each upstream column's authorised purposes.
6. **Log → training trap test.** Create a feature or corpus that reads
   a log class without a matching RoPA row. G4 and G6 together must
   block it.
7. **Gate-independence check.** Reorder the gates in the pipeline; the
   outcomes do not change. If they do, the gates are entangled.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Six gates covering collection → storage → feature → corpus → inference → log | 2 | Fewer than four unique stages fails |
| Each gate has a named owner and co-signer (where applicable) | 1 | "Everyone" fails |
| Each gate has an artifact a reviewer can open | 1 | Policy page does not count |
| Each gate has a concrete enforcement (CI / policy-as-code) | 2 | Prose fails |
| Each gate has a pass/fail criterion a build can decide automatically | 1 | "Looks fine" fails |
| Decision rationale explains ordering and scope | 1 | Un-ordered sets fail |
| Negative-test walk-through included (a failing PR) | 1 | Required to prove the gates bite |
| Log → training trap is explicitly addressed | 1 | Missing this is an automatic fail of this item |

Automatic failures:

- No enforcement: a gate that is only written in English.
- A single gate that covers everything from collection to retirement.
- A gate that depends on human reviewer sign-off only, with no
  machine-checkable artifact.
- Treating retention as optional.

## 5. Common mistakes

- **Gates as policy pages.** If a reviewer cannot run the gate as
  code, it is not a gate — it is an aspiration.
- **Catch-all gate.** "Privacy review" as a single stage lets drift
  accumulate; split it by lifecycle point.
- **No pseudonymisation boundary.** G2 must name where the identity
  mapping lives, because without that boundary the catalog taxonomy
  is theatre.
- **Treating features as pass-through.** G3 is the place where a team
  most often silently grows the purpose. Feature authors routinely
  justify "we might need it later." That is the failure mode; the
  subset check forces it into the open.
- **Confusing minimisation with anonymisation.** Minimisation is
  "collect only what is needed." Anonymisation is "process out
  identifiability." Pseudonymisation is weaker than anonymisation.
  The three are not interchangeable and the gate should not conflate
  them.
- **Ignoring logs.** Inference prompts and completions, debug logs,
  and traces are personal data when they contain personal data. G6
  exists because teams ship these as "operational" and then quietly
  use them for training.
- **Retention everywhere = retention nowhere.** A single global
  retention policy does not survive contact with real log classes.
  Per-class TTL is the minimum.
- **Writing the gates for the auditor and not the engineer.** A gate
  that passes with a policy citation but no technical test is a gate
  the engineering team will route around.

## 6. References

- Regulation (EU) 2016/679 (General Data Protection Regulation),
  Arts. 5(1)(c) — minimisation, 5(1)(e) — storage limitation,
  5(1)(b) — purpose limitation, 25 — data protection by design and
  by default, 30 — records of processing activities.
  <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- EDPB Guidelines 4/2019 on Article 25 Data Protection by Design and
  by Default, version 2.0. <https://edpb.europa.eu/>
- Regulation (EU) 2024/1689 — Artificial Intelligence Act, Article
  10 (data governance). <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- NIST Privacy Framework v1.0 — Control (CT) and Protect (PR)
  functions. <https://www.nist.gov/privacy-framework>
- NIST AI Risk Management Framework (AI RMF 1.0) — Govern, Map,
  Measure functions. <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST AI 600-1 — Generative AI Profile, data-provenance and
  data-minimisation guidance.
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
- OWASP Machine Learning Security Top 10, in particular ML02 (Data
  Poisoning) and ML06 (AI Supply Chain Attacks) — minimisation
  reduces blast radius of these. <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — ML Supply Chain Compromise and Data from Information
  Repositories techniques. <https://atlas.mitre.org/>
