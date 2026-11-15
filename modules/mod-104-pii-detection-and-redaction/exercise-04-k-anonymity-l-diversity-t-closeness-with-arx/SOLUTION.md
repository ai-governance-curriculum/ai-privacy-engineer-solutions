# Exercise 04 — k-Anonymity, l-Diversity, t-Closeness with ARX — Reference Solution

## 1. Solution overview

This is an **implementation + design exercise**. The learner is asked to
take a dataset with direct and quasi-identifiers, use the ARX Data
Anonymization Tool to apply three progressively stronger privacy models
(k-anonymity, l-diversity, t-closeness), measure the privacy / utility
trade-off, and write down which configuration they would ship and why.

The assessable output is a repository with:

1. The input CSV, a data dictionary identifying each column as a direct
   identifier, quasi-identifier, sensitive attribute, or insensitive.
2. Generalisation hierarchies (one per quasi-identifier) in ARX's expected
   format.
3. An ARX project file (or an equivalent reproducible script using the ARX
   API) that (a) suppresses direct identifiers, (b) enforces k-anonymity,
   l-diversity, and t-closeness in three runs with explicit parameters,
   (c) exports the anonymised dataset per run, and (d) emits a utility
   report (non-uniform entropy, granularity, discernibility).
4. A `report.md` that explains the chosen parameters, shows the three
   outputs, compares utility metrics, and recommends one configuration
   for release.

The reference below walks through the privacy-model definitions, the
parameter choices a learner must defend, worked example configurations, a
utility-measurement plan, and the rubric. It uses the ARX project terms
as documented at <https://arx.deidentifier.org/>.

## 2. Worked implementation

### 2.1 Column classification (the step that most learners get wrong)

Before any anonymisation runs, every column is placed into exactly one of
four buckets. The default classifications in the reference below assume a
typical healthcare analytics-style dataset; the learner's classification
must match their data dictionary and be defended in `report.md`.

| Bucket | Treatment | Example columns |
|---|---|---|
| Direct identifier | Suppressed (removed or pseudonymised) | `patient_id`, `full_name`, `email`, `mrn` |
| Quasi-identifier (QI) | Generalised to achieve k / l / t | `zip_code`, `date_of_birth` (→ year), `gender`, `admission_date` (→ month) |
| Sensitive attribute | Kept as-is; distribution constrained by l / t | `diagnosis_icd10`, `salary_band`, `religion` |
| Insensitive | Kept as-is, no constraint | `encounter_duration_minutes`, `clinical_notes_hash` |

Design rationale:

- **QI selection decides everything.** Attackers link records by
  combinations of QIs (famously, five-digit ZIP + DOB + gender identifies
  a large fraction of US adults, per Sweeney's work and the Netflix /
  AOL cases). If you misclassify a QI as insensitive, k-anonymity is
  cosmetic.
- **Sensitive attribute is the *target* of inference.** It is the thing an
  attacker wants to learn. l-Diversity and t-Closeness protect the
  sensitive attribute; they are irrelevant without one.
- **Pseudonymising a direct identifier is not anonymisation.** Pseudonyms
  remove the obvious name but leave linkage-by-QI intact. Suppress direct
  identifiers unless a specific downstream use *requires* a stable
  pseudonym (and then record why).

### 2.2 Generalisation hierarchies

Each QI needs a hierarchy that moves from most specific to most general
in discrete levels. ARX accepts these as CSV files (level-0, level-1, …).

```
# zip_hierarchy.csv (interval-based, 5 → 4 → 3 → * digits)
10001,1000*,100**,*
10002,1000*,100**,*
10010,1001*,100**,*
…
```

```
# dob_hierarchy.csv (date → year → 5-year band → *)
1981-07-02,1981,1980-1984,*
1982-09-15,1982,1980-1984,*
1986-03-22,1986,1985-1989,*
…
```

```
# gender_hierarchy.csv (2 → *)
F,*
M,*
X,*
```

The hierarchies must use **suppression at the top**, never a secret
constant. Replacing a ZIP with `"REDACTED"` instead of `*` complicates
downstream analytics and does not change the privacy claim.

### 2.3 Running ARX three ways

The learner must configure three runs with explicit parameters. The
reference values below are *defensible defaults* for a dataset in the
range 10 k – 100 k rows with the schema above; the learner must defend
their own numbers.

#### Run 1 — k-anonymity only

- Model: `k-anonymity`
- `k = 5`
- Suppression limit: `3%` (ARX will suppress up to 3 % of rows that cannot
  be placed into a group of size k)
- Transformation metric: `Loss` or `Non-Uniform Entropy`
- Monotonic `true`

Interpretation: every record is indistinguishable from at least four
others on the QI combination. Rationale for `k=5`: historical HHS Safe
Harbor plus Expert Determination practice uses `k ≥ 5` as a widely-cited
starting point for low-risk release; higher `k` is used for public
releases.

#### Run 2 — k-anonymity + l-diversity

- Everything from Run 1, plus
- Model: `distinct-l-diversity` (or `entropy-l-diversity` for stricter
  distribution control) on the sensitive attribute
- `l = 3`
- Suppression limit: `5%`

Interpretation: within every equivalence class of k records, the
sensitive attribute takes on at least l distinct values. Addresses
Machanavajjhala et al.'s observation that k-anonymity alone is
vulnerable to **homogeneity** and **background-knowledge** attacks when
the sensitive attribute is uniform or near-uniform in an equivalence
class.

#### Run 3 — k-anonymity + t-closeness

- Everything from Run 1, plus
- Model: `t-closeness` with `t = 0.2`, using **earth-mover's distance** for
  categorical sensitive attributes and the equal-distance variant for
  numerical ones
- Suppression limit: `10%`

Interpretation: the distribution of the sensitive attribute within any
equivalence class is within distance `t` of the distribution in the full
table. Addresses Li, Li, and Venkatasubramanian's observation that
l-diversity does not protect against **skewness** and **similarity**
attacks when the sensitive values in a class, though diverse, are all
"close" in their semantic meaning or all far from the population mean.

### 2.4 Example ARX API script (Java)

Programmatic reproduction matters more than a GUI session. The script
below configures and runs the first of the three variants; the other two
swap in the corresponding `PrivacyCriterion`.

```java
// Anonymise.java (ARX API 3.x; see https://arx.deidentifier.org/anonymization-tool/api/)
import org.deidentifier.arx.*;
import org.deidentifier.arx.criteria.*;
import org.deidentifier.arx.aggregates.*;
import org.deidentifier.arx.metric.*;
import java.io.*;
import java.nio.charset.StandardCharsets;

public class Anonymise {
  public static void main(String[] args) throws Exception {
    Data data = Data.create("input.csv", StandardCharsets.UTF_8, ',');

    // Attribute types
    data.getDefinition().setAttributeType("patient_id",
        AttributeType.IDENTIFYING_ATTRIBUTE);
    data.getDefinition().setAttributeType("email",
        AttributeType.IDENTIFYING_ATTRIBUTE);
    data.getDefinition().setAttributeType("zip_code",
        AttributeType.QUASI_IDENTIFYING_ATTRIBUTE);
    data.getDefinition().setAttributeType("date_of_birth",
        AttributeType.QUASI_IDENTIFYING_ATTRIBUTE);
    data.getDefinition().setAttributeType("gender",
        AttributeType.QUASI_IDENTIFYING_ATTRIBUTE);
    data.getDefinition().setAttributeType("diagnosis_icd10",
        AttributeType.SENSITIVE_ATTRIBUTE);
    data.getDefinition().setAttributeType("encounter_duration_minutes",
        AttributeType.INSENSITIVE_ATTRIBUTE);

    data.getDefinition().setHierarchy("zip_code",
        Hierarchy.create("zip_hierarchy.csv", StandardCharsets.UTF_8, ','));
    data.getDefinition().setHierarchy("date_of_birth",
        Hierarchy.create("dob_hierarchy.csv", StandardCharsets.UTF_8, ','));
    data.getDefinition().setHierarchy("gender",
        Hierarchy.create("gender_hierarchy.csv", StandardCharsets.UTF_8, ','));

    ARXConfiguration config = ARXConfiguration.create();
    config.addPrivacyModel(new KAnonymity(5));
    config.setSuppressionLimit(0.03d);
    config.setQualityModel(Metric.createLossMetric());

    ARXAnonymizer anonymizer = new ARXAnonymizer();
    ARXResult result = anonymizer.anonymize(data, config);

    result.getOutput(false).save("output_k5.csv", ',');
    StatisticsQuality quality = result.getOutput(false)
        .getStatistics().getQualityStatistics();
    System.out.println("Granularity: " + quality.getGranularity().getValue());
    System.out.println("Discernibility: " + quality.getDiscernibility().getValue());
    System.out.println("Non-uniform entropy: " + quality.getNonUniformEntropy().getValue());
  }
}
```

For Run 2 (l-diversity) swap the privacy-model line:

```java
config.addPrivacyModel(new KAnonymity(5));
config.addPrivacyModel(new DistinctLDiversity("diagnosis_icd10", 3));
config.setSuppressionLimit(0.05d);
```

For Run 3 (t-closeness):

```java
config.addPrivacyModel(new KAnonymity(5));
config.addPrivacyModel(new HierarchicalDistanceTCloseness(
    "diagnosis_icd10", 0.2d,
    Hierarchy.create("diagnosis_hierarchy.csv", StandardCharsets.UTF_8, ',')));
config.setSuppressionLimit(0.10d);
```

### 2.5 Utility measurement

ARX reports three utility metrics out of the box; the learner should
report all three per run. In addition, the learner must demonstrate
*task utility* on at least one downstream query the dataset was built to
serve.

| Metric | What it measures | How to read it |
|---|---|---|
| Granularity | How coarse the generalisation is per cell, averaged | Lower is better (less generalisation) |
| Discernibility | Quadratic cost of equivalence-class sizes above k | Lower is better |
| Non-uniform entropy | Residual information in the dataset | Higher is better |
| Task-specific utility | Downstream query answered on anonymised vs. raw | Business-defined threshold |

A typical task-utility test on a health dataset is "age-group × diagnosis
counts within ±5 % of raw." If no run meets the task threshold, go back
to the hierarchies before weakening the privacy parameters.

### 2.6 Recommendation template

> **Recommend: Run 3 (k = 5, t = 0.2).**
>
> **Why.** Run 1 (k only) is vulnerable to the homogeneity attack on
> `diagnosis_icd10` — equivalence classes with 5 patients who all share
> the same diagnosis leak the diagnosis for any attacker who can place a
> target in the class. Run 2 (l = 3) fixes that but, on our data, several
> l-diverse classes consist of three ICD-10 codes that are all variants
> of the same condition family (`E11.x` diabetes subtypes); diversity is
> satisfied but *similarity* is not. Run 3 closes the gap: diagnosis
> distribution within each class is within EMD = 0.2 of the full-table
> distribution.
>
> **Utility cost.** Granularity rises from 0.21 (k only) to 0.34 (k+t).
> The age × diagnosis task query is within 4 % of raw on Run 3; we
> accept this.
>
> **Flip triggers.** Re-anonymise if (a) a QI is added to the schema,
> (b) a sensitive attribute changes (e.g., a new condition is added to
> `diagnosis_icd10`) and shifts the full-table distribution by > 5 %,
> or (c) the suppression limit on Run 3 (10 %) proves too lossy for a
> downstream use.

## 3. Validation steps

1. **Direct identifier check.** `head output_*.csv` shows no column
   containing a direct identifier. Pseudonyms are allowed only with a
   documented reason.
2. **k check.** Group the output by the QI combination and assert the
   smallest group has size ≥ k.
3. **l check (Run 2).** For each equivalence class in Run 2, count
   distinct values of the sensitive attribute; the minimum must be ≥ l.
4. **t check (Run 3).** Compute the EMD between each equivalence class's
   sensitive-attribute distribution and the full-table distribution. The
   maximum must be ≤ t.
5. **Suppression accounting.** The number of suppressed rows per run must
   not exceed the configured suppression limit times input rows.
6. **Reproducibility.** Running the ARX script on the same input must
   produce identical output. If results vary, the configuration is
   underspecified (missing hierarchy, non-deterministic quality model).
7. **Task-utility test.** Run the agreed downstream query on both raw and
   anonymised; the delta must be within the documented threshold.
8. **Privacy regression.** Re-run all three validation checks after any
   change to the hierarchies or parameters.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Columns correctly classified into identifier / QI / sensitive / insensitive, with a written rationale | 2 | Misclassifying a QI is an automatic fail |
| Generalisation hierarchies exist for every QI and reach `*` at the top | 1 | No `REDACTED` placeholders |
| Three runs executed: k-only, k+l, k+t, with reproducible scripts | 2 | GUI-only runs lose a point |
| Parameters (k, l, t, suppression limit) are explicitly defended | 1 | Not "defaults from the docs" |
| Utility metrics reported per run (granularity, discernibility, NUE) | 1 | Plus at least one task-utility check |
| Recommendation picks one run and names flip triggers | 1 | "Tie" fails |
| Validation checks (k, l, t, suppression) coded as asserts, not prose | 1 | Reviewable in CI |
| Discussion of residual risk (what the chosen model does *not* protect against) | 1 | E.g., differential-privacy-style inference attacks |

Automatic failures:
- Relying on pseudonymisation alone (no generalisation) and calling it
  anonymisation.
- k ≥ 2 but l unspecified when the sensitive attribute has low diversity.
- Suppression limit set to 50 % or higher to "make the numbers work."
- No reproducible script — GUI-only outputs cannot be re-run in CI.

## 5. Common mistakes

- **Treating pseudonymisation as anonymisation.** Removing or hashing a
  name does not stop linkage by QIs. Both GDPR (Recital 26) and HIPAA's
  Expert Determination path treat pseudonymised data as personal data.
- **Forgetting the homogeneity attack.** k = 5 is useless when all 5
  records in a class share the sensitive value. l-Diversity exists for
  this.
- **Forgetting the similarity attack.** l = 3 is useless when the 3
  sensitive values are all in the same semantic family. t-Closeness
  exists for this.
- **Choosing k to make the output look nicer.** k is a privacy parameter,
  not a utility knob. If the output is too lossy, improve the
  hierarchies, drop a QI from the set after documenting why, or accept a
  smaller release.
- **Using the mean to defend a t value.** EMD is a distribution distance;
  "the mean is close" is not "the distribution is close."
- **Not reporting suppression.** A model that satisfies the constraint by
  suppressing 40 % of rows is not comparable to one that satisfies it by
  generalising. Report both.
- **Assuming k-anonymity composes.** Two k-anonymous releases of the same
  population with different QI sets can be linked back to re-identify
  individuals. Treat successive releases as a sequence, not independent
  snapshots.
- **Hand-waving at the "attacker model".** The learner must state what
  the attacker is assumed to know. k-anonymity assumes attackers know the
  QIs; it says nothing about attackers who know additional background.

## 6. References

- ARX Data Anonymization Tool — project site and documentation.
  <https://arx.deidentifier.org/>
- ARX API documentation.
  <https://arx.deidentifier.org/anonymization-tool/api/>
- L. Sweeney, "k-anonymity: a model for protecting privacy,"
  International Journal of Uncertainty, Fuzziness and Knowledge-based
  Systems, 10(5):557–570, 2002.
- A. Machanavajjhala, D. Kifer, J. Gehrke, M. Venkitasubramaniam,
  "l-Diversity: Privacy Beyond k-Anonymity," ACM TKDD, 1(1):3, 2007.
- N. Li, T. Li, S. Venkatasubramanian, "t-Closeness: Privacy Beyond
  k-Anonymity and l-Diversity," ICDE 2007.
- U.S. Department of Health and Human Services, "Guidance Regarding
  Methods for De-identification of Protected Health Information in
  Accordance with the Health Insurance Portability and Accountability Act
  (HIPAA) Privacy Rule."
  <https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/>
- NIST AI Risk Management Framework (AI RMF 1.0) — used to frame the
  governance-level decision about release risk.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS — relevant adversary models for re-identification attacks
  against released datasets. <https://atlas.mitre.org/>
