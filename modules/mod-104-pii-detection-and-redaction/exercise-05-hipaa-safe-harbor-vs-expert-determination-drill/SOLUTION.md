# Exercise 05 — HIPAA Safe Harbor vs. Expert Determination — Reference Solution

## 1. Solution overview

This is a **design exercise**. The learner is presented with one or more
scenarios in which a covered entity or business associate wants to use or
release health data for secondary purposes (research, analytics, or model
training), and must decide — per scenario — whether to de-identify under
HIPAA's **Safe Harbor** method (45 CFR 164.514(b)(2)) or the **Expert
Determination** method (45 CFR 164.514(b)(1)), carry the chosen method
through to an engineering plan, and document the residual risk.

The assessable output is a repository or document that contains, for each
scenario:

1. A "pathway decision" with written rationale citing the specific
   regulatory subsection.
2. If Safe Harbor: a per-identifier removal / generalisation plan covering
   all 18 identifiers in 164.514(b)(2)(i)(A)–(R) plus the
   actual-knowledge clause in (b)(2)(ii).
3. If Expert Determination: an expert-engagement plan including the
   expert's qualifications, the statistical method they will use, the
   small-number threshold, and how the determination will be documented.
4. A re-identification risk register (what remains, who could attack,
   which linkage datasets are in scope).
5. A data-use agreement (DUA) or equivalent downstream control plan if
   the data is being released externally.
6. A "re-evaluation trigger" list — what change to the data, population,
   or law would reopen the decision.

The reference below gives the regulatory anchors, a worked pathway choice
for a representative scenario, a Safe Harbor checklist, an Expert
Determination skeleton, and the rubric. It does not restate the
regulation verbatim; it points to the authoritative text and HHS
guidance.

## 2. Worked implementation

### 2.1 Pathway comparison (what the learner must internalise)

| Dimension | Safe Harbor (164.514(b)(2)) | Expert Determination (164.514(b)(1)) |
|---|---|---|
| Regulatory text | Remove the enumerated 18 identifiers; have no actual knowledge that remaining information can identify an individual. | A person with appropriate knowledge and experience of statistical and scientific principles and methods applies them to determine the risk of re-identification is "very small." |
| Who signs off | The covered entity/BA, against the checklist. | A qualified expert, in a dated, written determination. |
| Evidence a reviewer inspects | The 18-item checklist, per field, mapped to the schema. | The expert's written opinion, including methodology, data used, and assumptions; the expert's CV. |
| Typical data loss | High and uniform — ages > 89 collapsed to a single bucket; ZIP codes truncated to the first three digits (with population rules); all dates reduced to year. | Tunable — the expert balances utility and risk for the specific use. |
| Re-use flexibility | Low — any later join can trigger "actual knowledge." | Higher — if the risk assessment covers the new linkage. |
| Cost / cycle time | Low. | Higher — expert time, documentation. |
| Right fit | Low-stakes internal analytics, broad research datasets, external release with coarse data. | High-utility releases, novel data types (genomic, imaging, unstructured text), narrow release populations. |

The "actual knowledge" clause in (b)(2)(ii) is where Safe Harbor fails for
AI teams. Model training that joins "de-identified" clinical notes with a
provider's own patient registry puts the covered entity on actual notice;
Safe Harbor is no longer a defence.

### 2.2 The 18 Safe Harbor identifiers (checklist form)

Each row must be completed with "removed," "generalised to <X>," or
"not present" **before** the dataset is considered de-identified. The
regulatory wording governs — paraphrased here for legibility only.

| # | Identifier | Standard treatment |
|---|---|---|
| A | Names | Removed |
| B | Geographic subdivisions smaller than a state; street address, city, county, precinct, ZIP code and equivalent codes | ZIP: first three digits only, with the "20,000-population and 00 rule" from 164.514(b)(2)(i)(B); street / city / county / precinct: removed |
| C | All elements of dates (except year) directly related to an individual, including birth date, admission date, discharge date, date of death; and all ages over 89 and all elements of dates (including year) indicative of such age | Dates: year only; age > 89: single category |
| D | Telephone numbers | Removed |
| E | Fax numbers | Removed |
| F | Email addresses | Removed |
| G | Social Security numbers | Removed |
| H | Medical record numbers | Removed or replaced by a non-reversible token (and document whether tokenisation introduces re-identifiability through the token system) |
| I | Health plan beneficiary numbers | Removed |
| J | Account numbers | Removed |
| K | Certificate / license numbers | Removed |
| L | Vehicle identifiers and serial numbers, including license plate numbers | Removed |
| M | Device identifiers and serial numbers | Removed |
| N | Web URLs | Removed |
| O | IP addresses | Removed |
| P | Biometric identifiers, including fingerprints and voiceprints | Removed |
| Q | Full-face photographs and any comparable images | Removed |
| R | Any other unique identifying number, characteristic, or code | Assessed and removed — this is the catch-all that bites AI datasets with model-generated IDs, cohort IDs, or granular feature combinations |
| + | 164.514(b)(2)(ii) actual knowledge | No actual knowledge that the remaining information could be used alone or in combination with other information to identify an individual |

The learner's checklist must include item R and the actual-knowledge
clause. Both are the parts most often overlooked.

### 2.3 Worked pathway choice (representative scenario)

> **Scenario (local to this exercise):** A hospital system wants to share
> 500,000 de-identified clinical notes with an internal AI team to fine-
> tune a note-summarisation model. The AI team is inside the covered
> entity. Notes include free-text provider impressions, structured vital
> signs, admission/discharge dates, and a stable tokenised MRN.
>
> **Pathway: Expert Determination.**
>
> **Why not Safe Harbor.** Free-text notes cannot be mechanically scrubbed
> of all 18 identifiers with confidence (R — "other unique identifying
> number, characteristic, or code" — regularly appears in notes as rare
> diagnoses, employer names, uncommon procedures, place-of-birth
> mentions, etc.). Removing all dates would destroy clinical temporality,
> which is the signal we want to train on. The tokenised MRN keeps the
> dataset joinable to the live record system; under Safe Harbor this is
> allowed only if the covered entity does not disclose the token system,
> but the AI team is inside the covered entity and has actual knowledge
> of the mapping — so Safe Harbor fails per (b)(2)(ii).
>
> **Expert Determination plan.**
> - Engage a qualified expert (statistician or privacy professional with
>   documented experience in health data re-identification) within the
>   compliance organisation or on contract.
> - Method: a combination of (1) removing the 18 identifiers from
>   structured columns, (2) a Presidio-based unstructured scrubber over
>   the notes with the custom recognisers from exercise 01, (3) date
>   shifting per patient (consistent offset within a patient, different
>   offset across patients) to preserve temporality while breaking
>   calendar linkage, and (4) a quantitative re-identification risk
>   model (e.g., the k-anonymity / population-uniqueness approach from
>   exercise 04 applied to the quasi-identifiers that remain).
> - Threshold: the expert sets a numeric "very small" risk bound (common
>   formulations use a maximum re-identification risk of 0.09 for
>   public release and lower values for closed releases; the specific
>   threshold is the expert's call and must be documented).
> - Deliverable: a signed, dated written opinion describing the methods,
>   information used, analytic justification, and the determination.
>
> **Flip triggers.**
> - A new note type is added (e.g., psychiatric notes), because the
>   base-rate statistics change.
> - The MRN tokenisation scheme changes or the mapping leaves the covered
>   entity.
> - The population covered by the dataset becomes smaller (e.g., single
>   rare-disease cohort), because small cohorts increase uniqueness.
> - The dataset moves from internal fine-tuning to external release.

### 2.4 Scenario pattern — when Safe Harbor *is* the right call

> **Scenario:** The same hospital wants to publish quarterly aggregate
> statistics (counts of admissions by three-digit ZIP, by age band, by
> admission year) to a public dashboard.
>
> **Pathway: Safe Harbor.**
>
> **Why.** The aggregates are structured. Each of the 18 identifiers can
> be deterministically removed or generalised: ZIPs collapse to the three-
> digit rule (suppressing areas with population < 20,000 per
> 164.514(b)(2)(i)(B)); dates to year; ages > 89 to a single category.
> No free text; no join key to the covered entity's live system. The
> actual-knowledge clause is satisfiable without statistical argument.
>
> **Caveat.** The small-cell problem persists: a cell with count = 1 or 2
> by (three-digit ZIP, age band, diagnosis) can re-identify. Apply a cell
> suppression threshold (e.g., suppress cells with count < 11, the HHS
> guidance's commonly cited threshold for public release). Document the
> threshold.

### 2.5 Downstream controls (both pathways)

De-identification is necessary but not sufficient. The learner must also
document:

- **DUA / downstream control.** If the dataset leaves the covered entity,
  a Data Use Agreement (for Limited Data Sets) or equivalent contract
  restricting re-identification, re-sharing, and use beyond the agreed
  purpose. For de-identified data, a DUA is not legally required by HIPAA
  but is still used as a contractual control.
- **Access controls.** De-identified ≠ public. The dataset is still
  access-controlled, encrypted at rest, and logged.
- **Monitoring for re-identification attempts.** Log and alert on queries
  that attempt to join with live PHI stores.
- **Retention and destruction.** De-identified data follows a separate
  retention policy from PHI; document it.

## 3. Validation steps

1. **Pathway citation.** The decision document cites the specific sub-
   section of 164.514(b)(1) or 164.514(b)(2) and identifies which
   requirement was the deciding factor.
2. **Safe Harbor completeness.** If Safe Harbor: produce a 19-row table
   (A–R plus actual-knowledge) with a per-field disposition. Any row
   marked "not applicable" is challenged until the learner shows why.
3. **Expert Determination documentation.** If Expert Determination: the
   written opinion exists as a dated artifact, names the expert and their
   qualifications, states the method, states the numeric risk threshold,
   and is reproducible — someone else with the same data and method
   would reach the same determination.
4. **Residual-risk register.** At least three named re-identification
   vectors (e.g., rare-disease uniqueness, adjacent-dataset linkage,
   quasi-identifier concentration) with mitigations.
5. **Actual-knowledge check (Safe Harbor only).** The covered entity
   affirms in writing that it does not have actual knowledge that the
   remaining data could identify an individual. "We removed the 18
   identifiers" alone is not a disposition of (b)(2)(ii).
6. **Small-cell check (aggregates).** Published aggregates include a
   documented cell-suppression threshold.
7. **Flip triggers.** The decision document lists at least three events
   that would reopen the pathway decision.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Pathway choice cites the correct subsection and names the deciding factor | 2 | "Safe Harbor is simpler" is not a deciding factor |
| All 18 Safe Harbor identifiers addressed (if applicable) with a disposition | 2 | Missing item R is an automatic failure |
| Actual-knowledge clause (b)(2)(ii) explicitly addressed | 1 | Separate from the 18-item list |
| Expert Determination plan (if applicable) names a method, a threshold, and documentation | 2 | "Trust me" fails |
| Re-identification risk register with at least three vectors and mitigations | 1 | Linkage, uniqueness, small cells at minimum |
| Downstream controls (DUA / access / monitoring / retention) documented | 1 | De-identified ≠ public |
| Flip triggers named as measurable events | 1 | "If things change" fails |

Automatic failures:
- Treating pseudonymisation of names as HIPAA de-identification.
- Safe Harbor pathway applied to free-text notes without additional
  controls and without engaging the actual-knowledge clause.
- Expert Determination with no named expert, no stated method, no
  documented threshold.
- Confusing a Limited Data Set (164.514(e)) with a de-identified data
  set (164.514(a)); the two have different disclosure rules.
- Publishing aggregates without a documented small-cell threshold.

## 5. Common mistakes

- **Treating pseudonymised data as de-identified.** HIPAA distinguishes
  between pseudonymisation (still PHI) and de-identification. Hashing an
  MRN is pseudonymisation; it does not satisfy either Safe Harbor or
  Expert Determination on its own.
- **Forgetting item R (the catch-all).** Any unique identifying
  characteristic — a rare ICD-10 code combined with a small-population
  ZIP, a model-generated cohort ID, an "employer" field — can
  re-identify. Item R exists to catch the ones the drafters did not list.
- **Ignoring the actual-knowledge clause.** Safe Harbor is not automatic.
  If the covered entity *could* re-identify a subject using information
  reasonably available to it, Safe Harbor fails even with all 18
  identifiers removed.
- **Using Safe Harbor on unstructured notes.** Free text cannot be
  mechanically guaranteed to be identifier-free; Expert Determination,
  combined with a scrubber and a quantitative risk model, is the usual
  pathway.
- **Confusing the Limited Data Set with de-identification.** The Limited
  Data Set (164.514(e)) permits release of some identifiers (dates,
  city, state, five-digit ZIP, age) under a DUA; it is *not* the Safe
  Harbor data set and has different downstream rules.
- **Applying Expert Determination without an expert.** The statute
  requires "a person with appropriate knowledge of and experience with
  generally accepted statistical and scientific principles and methods."
  A product manager with Excel does not qualify.
- **One-and-done.** The determination is time-bound. New data types, new
  joins, new recipients, and shifts in the population all trigger
  re-evaluation.
- **Assuming "very small" is defined by HHS.** HHS deliberately did not
  set a numeric threshold; the expert does, and must defend it.
- **Treating de-identified data as outside HIPAA forever.** If a
  recipient re-identifies or combines with other data such that
  individuals can be identified, the covered entity's actual-knowledge
  posture changes and the downstream obligations reappear.

## 6. References

- 45 CFR § 164.514 (de-identification of protected health information).
  <https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164/subpart-E/section-164.514>
- U.S. Department of Health and Human Services, "Guidance Regarding
  Methods for De-identification of Protected Health Information in
  Accordance with the Health Insurance Portability and Accountability
  Act (HIPAA) Privacy Rule."
  <https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/>
- 45 CFR § 164.514(e) — Limited Data Set and Data Use Agreement.
  <https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164/subpart-E/section-164.514>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — relevant to re-identification
  as a membership-inference-adjacent risk when de-identified datasets are
  used to train models.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — adversary techniques for linking de-identified releases
  with external datasets. <https://atlas.mitre.org/>
