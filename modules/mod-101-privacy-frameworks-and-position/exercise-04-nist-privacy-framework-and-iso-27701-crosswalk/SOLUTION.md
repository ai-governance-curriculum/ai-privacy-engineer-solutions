# Exercise 04 — NIST Privacy Framework + ISO/IEC 27701 Crosswalk — Reference Solution

## 1. Solution overview

This is a **design exercise**. The learner is asked to crosswalk the two
dominant voluntary privacy management frameworks — the NIST Privacy Framework
(PF) v1.0 (January 2020) and ISO/IEC 27701 — and explain which parts a
privacy engineer owns vs reviews, so the engineer can (a) pick the right
framework for a given organisation and (b) show auditors how specific
controls satisfy both at once without doing the mapping twice.

The assessable output has three parts:

1. A **core-structure comparison** of the two frameworks: vocabulary,
   organising structure, and the part the engineer actually touches.
2. A **control-level crosswalk table** keyed by the NIST PF Subcategory
   (Function / Category / Subcategory), with one row per Subcategory that is
   routinely driven by engineering (not just governance), mapping to the
   equivalent ISO/IEC 27701 clause / control.
3. A **worked implementation plan** that takes a single operational
   capability — say, "model-training data minimisation" — and shows which
   PF Subcategories and 27701 controls it evidences simultaneously, with
   named engineering artifacts.

## 2. Worked answer

### 2.1 Core-structure comparison

| Dimension | NIST Privacy Framework v1.0 | ISO/IEC 27701 |
|---|---|---|
| Base | Voluntary framework published by NIST; outcome-based; references NIST SP 800-53 for privacy controls | International standard; a *PIMS extension* of ISO/IEC 27001 (and relies on ISO/IEC 27002 controls) |
| Structure | Three parts: **Core** (Functions → Categories → Subcategories), **Profiles** (current / target), **Implementation Tiers** (1–4) | Management-system requirements (clauses 5–8 extensions to 27001) + PIMS-specific guidance (clauses 6.X–6.Y mapping onto 27002) + Annex A (additional controls for PII controllers) + Annex B (for PII processors) |
| Functions (PF) / Clauses (27701) | Identify-P, Govern-P, Control-P, Communicate-P, Protect-P | Follows 27001 structure: Context, Leadership, Planning, Support, Operation, Performance Evaluation, Improvement — extended for PII; Annex A/B are the "controls catalogue" equivalents |
| Scope of "privacy" | Broader; explicitly separates security (Protect-P) from the rest | Builds *on top of* an existing information-security management system; privacy additions are the delta |
| Role distinction | Framework is role-neutral; implementer decides who owns what | Explicitly separates **PII controllers** (Annex A) from **PII processors** (Annex B) |
| Certifiability | Not certifiable | Certifiable (a UKAS/ANAB-accredited body can audit); typically layered on an ISO/IEC 27001 certification |

Engineering-ownership read: PF **Control-P** and **Protect-P** are where
engineers spend most of their time. In 27701, engineers operate against
Annex A and Annex B controls (depending on the role the organisation
plays), plus the operational clauses (8.X) that extend 27001's Operation
clause with PII-specific obligations.

### 2.2 Control-level crosswalk

Rows are limited to NIST PF Subcategories whose implementation is directly
in the engineer's hands. Governance-only Subcategories (e.g., policy
authoring under Govern-P) are omitted for brevity; the pattern of the
crosswalk is the same.

Legend:
- **PF cite** uses the official Function.Category-nn notation from NIST PF
  v1.0 (e.g., `ID.IM-P1`).
- **27701 cite** uses Annex-A-#.#.# (controller-side) or Annex-B-#.#.#
  (processor-side) numbering as published in ISO/IEC 27701:2019 (Annex A
  is numbered from A.7, Annex B from B.8). Clauses outside Annex A/B are
  cited as `27701 clause X.Y`.

| PF Subcategory | Engineering outcome in plain English | ISO/IEC 27701 clause / control | Engineering artifact (indicative) |
|---|---|---|---|
| **ID.IM-P1** — Systems that process data are inventoried | Keep an inventory of systems processing personal data | 27701 A.7.2.8 / B.8.2.6 — Records related to processing of PII; 27701 clause 7.2 / 8.2 overall | Data catalog + RoPA join |
| **ID.IM-P2** — Owners / operators of the systems are identified | Each system has a named owner | 27701 clause 5.2.1 (leadership / roles); supports A.7.2.1 / B.8.2.1 | Service registry owner field |
| **ID.IM-P3** — Categories of individuals (subjects) are identified | Know whose data is processed | 27701 A.7.2.2 | DPIA + inventory |
| **ID.IM-P4** — Data actions (collect, use, share, retain, dispose) are identified | Know what processing happens | 27701 A.7.2.2 / B.8.2.2; records of processing | RoPA activity rows |
| **ID.IM-P5** — Purposes for the data actions are identified | Each processing has a declared purpose | 27701 A.7.2.1 (identify purposes) | Purpose tag in the catalog |
| **ID.IM-P6** — Data elements within the data actions are inventoried | Field-level inventory | 27701 A.7.2.2 (records) + A.7.4.1 (limit collection) | Catalog field list + schema |
| **ID.IM-P7** — Data processing environment is identified (e.g., geographic location) | Know where data lives and flows | 27701 A.7.5.1 / A.7.5.2 (transfers, jurisdictions); A.7.5.3 (records of transfers) | Region map + transfer register |
| **ID.IM-P8** — Data processing is mapped, illustrating… | Produce a data flow diagram | 27701 A.7.2.2 / B.8.2.2 (records); supports A.7.3.3 / B.8.3.1 | Data flow diagram per system |
| **ID.BE-P1 / P2 / P3** — Role in supply chain, priorities, dependencies | Know your role and your dependencies | 27701 clause 4.2 (interested parties), A.7.2.6 / B.8.5.1–B.8.5.8 (suppliers/customers) | Vendor register, customer register |
| **ID.RA-P1..P5** — Risks assessed | Assess privacy risk per activity | 27701 clause 6.1 (planning) and A.7.2.5 / B.8.2.3 (PIA / impact assessments) | DPIA entries + privacy risk register |
| **ID.DE-P1..P5** — Data processing ecosystem assessed | Assess third parties | 27701 A.7.2.6 (joint controllers), A.7.2.7 / B.8.5.3 (records), B.8.5.X suite | Processor agreements + due diligence |
| **GV.PO-P1..P5** — Organisational privacy policy | Governance baseline | 27701 clause 5.2 (policy) | Organisational policy |
| **GV.RR-P1..P4** — Roles & responsibilities | Named owners | 27701 clause 5.3, 7.2.1 | Org chart + RACI |
| **GV.AT-P1..P4** — Awareness and training | Training programme | 27701 clause 7.2.2 (awareness) | Training records |
| **GV.MT-P1..P7** — Monitoring and review | Measure and review | 27701 clauses 9 (performance evaluation), 10 (improvement) | Metrics + review cadence |
| **CT.PO-P1..P4** — Policies for data management | Operational policies | 27701 A.7.3.1 / A.7.3.3 — Determining and documenting legal basis / Determining when and how to provide consent | Policy + implemented config |
| **CT.DM-P1** — Data elements can be accessed for review | Access right works | 27701 A.7.3.6 (access, correction, erasure) | DSR access endpoint |
| **CT.DM-P2** — Data elements can be accessed for transmission (portability) | Export works | 27701 A.7.3.6 | DSR export endpoint |
| **CT.DM-P3** — Data elements can be accessed for alteration | Correction works | 27701 A.7.3.6 | DSR correction endpoint |
| **CT.DM-P4** — Data elements can be accessed for deletion | Deletion works | 27701 A.7.3.6 + A.7.4.7 (retention) | DSR erase endpoint + retention job |
| **CT.DM-P5** — Data are destroyed according to policy | Retention and destruction | 27701 A.7.4.7 — Retention; A.7.4.8 — Disposal | Retention / TTL / disposal jobs |
| **CT.DM-P6** — Data are transmitted using standardised formats | Portability format | 27701 A.7.3.6 implementation guidance | Export schema |
| **CT.DM-P7** — Mechanisms for transmitting processing permissions are established | Consent / permission signalling | 27701 A.7.3.4 — Obtaining and recording consent | Consent service + propagation |
| **CT.DM-P8** — Audit / log records are determined, documented, implemented, and reviewed | Audit logs for privacy events | 27701 clauses 8.X + A.7.3.X + 27001 A.8.15 (logging) | Privacy event log |
| **CT.DM-P9 / P10** — Technical measures implemented to manage processing | Field-level technical measures | 27701 A.7.4.X suite (data minimisation, de-identification, use/retention/disposal) | Minimisation + deid configs |
| **CT.DP-P1** — Data are processed to limit observability and linkability | Minimisation / linkability reduction | 27701 A.7.4.5 — PII minimisation; A.7.4.6 — De-identification | Deid pipeline, hashing, DP |
| **CT.DP-P2** — Data are processed to limit identification and reidentification | Anonymisation / pseudonymisation | 27701 A.7.4.5 / A.7.4.6 | Pseudonymisation / DP config |
| **CT.DP-P3** — Data are processed to limit the formulation of inferences about individuals' behaviour or activities | Inference-limiting controls | 27701 A.7.4.5 | Query limits, k-anonymity, DP |
| **CM.PO-P1** — Transparency policies | Transparency policy | 27701 A.7.3.2 — Determining information for PII principals | Published notice |
| **CM.AW-P1..P8** — Awareness / notice mechanisms | Notice served and accessible | 27701 A.7.3.2, A.7.3.3 | In-product notice, model card |
| **PR.PO-P1..P9** — Protective policies (security of data) | Baseline security policies | 27701 piggybacks on 27001 Annex A controls | Security policy set |
| **PR.AC-P1..P6** — Access controls | Identity and access | 27001 A.5 / A.8 controls (access, auth, MFA) | IAM config |
| **PR.DS-P1..P5** — Data security | Encryption, integrity, monitoring | 27001 A.8 controls (cryptography, data leakage); 27701 A.7.4.X where PII-specific | KMS, encryption configs |
| **PR.MA-P1..P2** — Maintenance | Patching, secure maintenance | 27001 A.8 controls | Patch schedule |
| **PR.PT-P1..P5** — Protective technology | Logging, boundary, hardening | 27001 A.8 controls | Logging / WAF / baseline configs |

Processor-specific mapping (when the organisation's role is processor, the
27701 column swaps to Annex B equivalents, e.g., B.8.2.1–B.8.5.X, and the
PF rows about consent, legal basis, and notice shift to "support the
controller's control"). This is the single most common error in a learner
submission — mapping as if the organisation were always the controller.

### 2.3 Worked implementation — model-training data minimisation

Feature: the training pipeline for a recommender model is instrumented so
that (a) only columns declared in the model card's training-data schema
are joined, (b) free-text fields are pre-redacted using a PII detector,
(c) joins exclude subjects with `consent_state != granted` or
`processing_restricted = true`, and (d) the resulting manifest is retained
for 180 days alongside a hash of the data.

- **PF subcategories evidenced simultaneously:** ID.IM-P4 (data actions),
  ID.IM-P6 (data elements), CT.DM-P9/P10 (technical measures),
  CT.DP-P1/P2/P3 (de-identification and inference-limiting), PR.DS-P1
  (data at rest protection), GV.MT-P1 (monitoring).
- **27701 controls evidenced simultaneously (controller role):**
  A.7.2.1 (identify purposes), A.7.2.2 (determine lawful basis — joined
  to CT.PO-P1), A.7.4.5 (PII minimisation), A.7.4.6 (de-identification),
  A.7.4.7 (retention), A.7.3.6 (data-subject rights via the restriction
  filter), clauses 9.X (performance evaluation) via the pipeline metrics.
- **Engineering artifacts a reviewer inspects:**
  - Model card, "training data" section — lists schema and sources.
  - Pipeline manifest in `pipelines/train_recommender/manifest.yaml` —
    enumerates columns and joins.
  - CI test `test_training_schema_locked` — fails if an undeclared column
    appears in the join.
  - Consent / restriction join SQL — includes the `consent_state` and
    `processing_restricted` filters.
  - PII redactor config and golden tests.
  - Retention / TTL job `jobs/train_manifest_gc.py` with the 180-day rule.
  - Dashboard: fraction of rows dropped by consent filter, drift over time.

### 2.4 Decision rationale: when to pick which framework

- **Pick NIST PF first** when the organisation is US-centric, already
  aligned to the NIST CSF for cybersecurity, and does not need a
  certification. The PF Profiles are an excellent engineering-planning
  tool.
- **Pick ISO/IEC 27701 first** when the organisation is already running an
  ISO/IEC 27001 ISMS, when enterprise customers demand a certificate, or
  when operating across jurisdictions where a formal management system is
  expected.
- **Run both in parallel** by maintaining a single implementation keyed by
  PF Subcategory with a 27701 cross-reference column (as in this
  crosswalk). The physical artifacts should not be duplicated — only the
  mapping.

## 3. Validation steps

1. **Vocabulary check.** Re-read the NIST PF v1.0 Core and locate every
   Subcategory the learner cited. The Core is downloadable from NIST at
   <https://www.nist.gov/privacy-framework>.
2. **27701 Annex check.** Confirm every cited Annex A control number
   corresponds to a controller-side control, and every Annex B number to a
   processor-side control. ISO/IEC 27701:2019 is purchased from ISO
   (<https://www.iso.org/standard/71670.html>); the learner must have a
   legitimate copy.
3. **Role test.** For each row, confirm whether the organisation is acting
   as controller or processor and that the 27701 column reflects the right
   Annex.
4. **Artifact test.** Every engineering artifact column should be a file,
   service, or config that a reviewer could ask to see.
5. **Non-duplication test.** No single engineering artifact should be
   invented twice to satisfy "the same control" in both frameworks — if
   the learner created two inventory systems, the mapping is wrong.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Core-structure comparison correctly distinguishes PF (voluntary, outcome, tiers) from 27701 (PIMS, certifiable, controller/processor Annexes) | 2 | Collapsing the two loses both points |
| Crosswalk includes at least 20 PF Subcategories across all five Functions | 2 | Missing a whole Function fails |
| Crosswalk maps to both Annex A and Annex B of 27701 where the role differs | 2 | Single-sided mapping loses both |
| Each row names an engineering artifact a reviewer can inspect | 2 | Not a policy |
| Worked implementation evidences at least three PF Subcategories and three 27701 controls through *one* set of artifacts | 1 | Prevents duplication |
| Decision rationale states a clear criterion for choosing one framework over the other | 1 | Not a shrug |

Automatic failures:
- Treating the PF Core as a control list (it is an outcomes list; controls
  come from SP 800-53 or equivalent).
- Treating 27701 as a standalone standard (it is a PIMS extension of
  27001).
- Using Annex A controls when the organisation is a processor.
- Any row that claims a certification is earned by filing the PF Profile
  (the PF is not certifiable).

## 5. Common mistakes

- **Confusing the PF with the CSF.** Many PF Subcategories have CSF
  look-alikes; the PF-specific ones (`-P` suffix) are the ones that matter
  here.
- **Mixing up Annex A and Annex B.** Annex A is for PII controllers,
  Annex B is for PII processors. Role determines which applies.
- **Treating the Tiers as capability maturity levels.** NIST explicitly
  says the Tiers are not maturity levels; they characterise an
  organisation's approach to managing privacy risk.
- **Mapping to "27001 Annex A" as if that covered privacy.** 27001 covers
  information security management; 27701 is the privacy delta. The two
  are complementary, not interchangeable.
- **Inventing per-framework artifacts.** The point of the crosswalk is
  reuse. One data-flow diagram evidences ID.IM-P8 in the PF and A.7.2.2
  in 27701 — do not build two.
- **Missing the training / minimisation controls for ML.** NIST PF's
  Control-P (CT.DP-P1..P3) and 27701's A.7.4.5 / A.7.4.6 are the direct
  hooks for ML data minimisation and de-identification. Learners often
  skip these because they do not feel "AI-specific."

## 6. References

- NIST Privacy Framework v1.0 (January 2020).
  <https://www.nist.gov/privacy-framework>
- NIST Privacy Framework Core (official downloadable).
  <https://www.nist.gov/privacy-framework/getting-started>
- NIST SP 800-53 Rev. 5 — Security and Privacy Controls for Information
  Systems and Organizations.
  <https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- ISO/IEC 27701:2019 — Security techniques — Extension to ISO/IEC 27001
  and ISO/IEC 27002 for privacy information management — Requirements and
  guidelines. <https://www.iso.org/standard/71670.html>
- ISO/IEC 27001:2022 — Information security management systems —
  Requirements.
- ISO/IEC 27002:2022 — Information security controls.
