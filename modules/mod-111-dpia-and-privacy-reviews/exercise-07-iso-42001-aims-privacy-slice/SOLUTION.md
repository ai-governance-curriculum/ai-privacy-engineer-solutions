# Exercise 07 — ISO/IEC 42001 AIMS — Privacy Slice — Reference Solution

## 1. Solution overview

This is a **design + crosswalk exercise**. The learner takes
**ISO/IEC 42001:2023** — the AI Management System (AIMS) standard
— and extracts the **privacy slice**: the clauses and Annex A
controls that a privacy engineer owns (or materially contributes
to) when the organisation is standing up an AIMS alongside its
existing privacy programme. The slice tells the privacy function
(a) where ISO 42001 duplicates or extends what they already run
under GDPR/HIPAA/[[exercise-01..05]], (b) where it adds genuine
new obligations, and (c) where it collides with ISO/IEC 27701
and NIST AI RMF so the organisation does not end up maintaining
three parallel systems.

The assessable output is four artefacts:

1. **An AIMS structural map** — the ten clauses of ISO/IEC
   42001 and the thirty-eight Annex A controls (A.2–A.10)
   written in plain language, with each item tagged as
   "privacy-owned", "privacy-contributed", or "non-privacy".
2. **A clause-level crosswalk** from ISO 42001 to (a) GDPR
   articles, (b) ISO/IEC 27701 clauses, (c) NIST AI RMF
   subcategories, and (d) the earlier exercises in this
   module ([[exercise-01..06]]). Rows are limited to the
   privacy-owned and privacy-contributed items.
3. **A worked implementation plan** for one AIMS objective —
   "end-to-end personal-data minimisation across the AI life
   cycle" — showing which AIMS clauses, Annex A controls, GDPR
   articles, and existing engineering artefacts evidence it
   simultaneously.
4. **A reviewer checklist** the privacy function hands to the
   AIMS internal auditor so the auditor can inspect the privacy
   slice without a separate briefing.

Scope note — what the slice is *not*: it is not the full AIMS.
Non-privacy obligations under ISO 42001 (model-performance
monitoring, bias assessment unrelated to special categories,
operational resilience of AI systems, supplier assurance for
non-personal-data pipelines) are out of scope for the privacy
function and are explicitly tagged "non-privacy" in §2.2 so the
AIMS owner (typically the AI governance lead, not the DPO)
knows where to look.

## 2. Worked answer

### 2.1 ISO/IEC 42001 in one page

ISO/IEC 42001:2023 is a **management-system standard** of the
same family as ISO/IEC 27001 (ISMS) and ISO/IEC 27701 (PIMS
extension). It specifies:

- **Clauses 4–10** — the management-system shell, identical in
  structure to ISO/IEC 27001 (Context, Leadership, Planning,
  Support, Operation, Performance Evaluation, Improvement).
- **Annex A** — the AIMS control reference set. The controls
  are grouped A.2 (Policies) through A.10 (Third-party
  relationships). Unlike ISO 27001 Annex A, the controls are
  not mandatory; the organisation declares applicability via a
  Statement of Applicability (clause 6.1.3).
- **Annex B** — implementation guidance for each Annex A
  control.
- **Annex C** — objectives and risk sources the organisation
  may consider.
- **Annex D** — use across domains or sectors.

Engineering-ownership read: the privacy function **owns**
(a) impact-assessment controls (A.5), (b) data-for-AI controls
(A.7), (c) information-for-interested-parties controls (A.8),
and (d) a subset of life-cycle (A.6), use (A.9), and third-
party (A.10) controls where personal data is in play. The AI
governance lead owns the shell (clauses 4–10) and the non-
privacy portions of A.2–A.10.

### 2.2 Clause and control map (privacy-owned vs. contributed vs. non-privacy)

Legend:

- **P** = privacy-owned. The privacy function is the
  accountable owner; the AIMS depends on their artefacts.
- **C** = privacy-contributed. Shared ownership; the AI
  governance lead holds the pen; the privacy function provides
  one or more required inputs.
- **—** = non-privacy. Listed for completeness; the AIMS
  owner handles.

Clauses (management-system shell):

| Clause | Title (paraphrase) | Role |
|---|---|---|
| 4.1 Context of the organisation | Internal/external issues affecting AIMS | C |
| 4.2 Needs and expectations of interested parties | Which parties expect what | C — privacy supplies the regulator + data-subject rows |
| 4.3 Scope | What the AIMS covers | C |
| 4.4 AIMS | Establish / maintain the system | C |
| 5.1 Leadership and commitment | Accountability | C |
| 5.2 AI policy | Published policy | C — privacy co-authors the personal-data clauses |
| 5.3 Roles, responsibilities and authorities | Named owners | C |
| 6.1.1 General | Risk framework | C |
| 6.1.2 AI risk assessment | Risk identification + analysis | C — privacy drives the personal-data risks |
| 6.1.3 AI risk treatment / SoA | Pick controls, state applicability | C |
| 6.1.4 AI system impact assessment | Impact assessment process | **P** — this is where DPIA / FRIA live |
| 6.2 Objectives | Measurable targets | C |
| 6.3 Planning changes | Change management | C |
| 7.1 Resources | Staffing, budget | — |
| 7.2 Competence | Skills | C — privacy competence requirements |
| 7.3 Awareness | Training and awareness | C |
| 7.4 Communication | Internal/external comms | C |
| 7.5 Documented information | Records | **P** for personal-data records |
| 8.1 Operational planning and control | Run the operation per plan | C |
| 8.2 AI risk assessment (operational) | Re-run assessment at operational checkpoints | C |
| 8.3 AI risk treatment (operational) | Treat residuals in operation | C |
| 8.4 AI system impact assessment (operational) | Re-run impact assessment | **P** |
| 9.1 Monitoring, measurement, analysis, evaluation | Metrics | C |
| 9.2 Internal audit | Self-audit | C |
| 9.3 Management review | Executive review | C |
| 10.1 Continual improvement | Improve the AIMS | C |
| 10.2 Nonconformity and corrective action | Fix what breaks | C |

Annex A controls (reference set) — privacy slice in **bold**:

| Group | Control (paraphrase) | Role |
|---|---|---|
| **A.2 Policies related to AI** | | |
| A.2.2 | AI policy exists | C |
| A.2.3 | Alignment with other organisational policies (incl. privacy) | **P** |
| A.2.4 | Review of the AI policy | C |
| **A.3 Internal organisation** | | |
| A.3.2 | AI roles & responsibilities | C |
| A.3.3 | Reporting of concerns | C |
| **A.4 Resources for AI systems** | | |
| A.4.2 | Resource documentation | C |
| A.4.3 | Data resources | **P** |
| A.4.4 | Tooling resources | — |
| A.4.5 | System & computing resources | — |
| A.4.6 | Human resources | C |
| **A.5 Assessing impacts of AI systems** | | |
| A.5.2 | AI system impact assessment process | **P** |
| A.5.3 | Documentation of AI system impact assessments | **P** |
| A.5.4 | Assessing impact on individuals or groups | **P** |
| A.5.5 | Assessing societal impacts | C |
| **A.6 AI system life cycle** | | |
| A.6.1.2 | Objectives for responsible development of AI systems | C |
| A.6.1.3 | Processes for responsible design and development | C |
| A.6.2.2 | Requirements and specification | **P** for personal-data requirements |
| A.6.2.3 | Documentation of AI system design and development | C |
| A.6.2.4 | AI system verification and validation | C — privacy signs off on personal-data tests |
| A.6.2.5 | AI system deployment | C |
| A.6.2.6 | AI system operation and monitoring | C |
| A.6.2.7 | AI system technical documentation | C |
| A.6.2.8 | AI system event logs | **P** for personal-data event logs |
| **A.7 Data for AI systems** | | |
| A.7.2 | Data for development and enhancement of AI systems | **P** |
| A.7.3 | Acquisition of data | **P** |
| A.7.4 | Quality of data for AI systems | C |
| A.7.5 | Data provenance | **P** |
| A.7.6 | Data preparation | **P** for minimisation + de-identification |
| **A.8 Information for interested parties of AI systems** | | |
| A.8.2 | System documentation and information for users | C |
| A.8.3 | External reporting | C |
| A.8.4 | Communication of incidents | C |
| A.8.5 | Information for interested parties | **P** — notice obligations |
| **A.9 Use of AI systems** | | |
| A.9.2 | Processes for responsible use | C |
| A.9.3 | Objectives for responsible use | C |
| A.9.4 | Intended use of the AI system | C |
| **A.10 Third-party and customer relationships** | | |
| A.10.2 | Allocation of responsibilities | **P** for Art. 28 responsibilities |
| A.10.3 | Suppliers | **P** — processor assurance |
| A.10.4 | Customers | C |

The privacy slice is thus concentrated in **A.5, A.7, A.8.5,
A.10.2–10.3**, plus clauses **6.1.4, 8.4, and 7.5 (records
related to personal data)**. Every row marked **P** is a row
where a GDPR obligation already lives; the slice is how the
organisation says "we are not running two systems."

### 2.3 Crosswalk (privacy-owned / privacy-contributed only)

Legend for the right-hand columns:

- **GDPR**: specific article(s).
- **27701**: Annex A (controller) and Annex B (processor)
  control numbers from ISO/IEC 27701:2019.
- **AI RMF**: NIST AI RMF 1.0 function / category / subcategory
  (e.g., `MAP-4.1`).
- **Local artefact**: pointer to the earlier exercises in this
  module.

| ISO 42001 | Plain-English outcome | GDPR | ISO 27701 | NIST AI RMF | Local artefact |
|---|---|---|---|---|---|
| **Cl. 6.1.4** AI system impact assessment | Process to assess AI-system impacts prior to deployment | Art. 35 | 27701 A.7.2.5 / B.8.2.3 (PIA) | `MAP-4`, `MAP-5` | [[exercise-01]], [[exercise-02]], [[exercise-03]] |
| **Cl. 8.4** AI system impact assessment (operational) | Re-run impact assessment on material change | Art. 35(11) | 27701 cl. 8 (operation) | `GOVERN-5.1`, `MANAGE-4.1` | [[exercise-01]] §review cadence |
| **Cl. 7.5** Documented information | Keep records of processing | Art. 30 | 27701 A.7.2.8 / B.8.2.6 | `GOVERN-1.4` | [[exercise-06]] §2.1 §02–§05 |
| **A.2.3** Alignment with other policies | AI policy aligns with privacy policy | Art. 24 | 27701 cl. 5.2 | `GOVERN-1.1` | [[mod-112]] |
| **A.4.3** Data resources | Catalogue the data resources used | Art. 30 | 27701 A.7.2.2 / B.8.2.2 | `MAP-2.1`, `MAP-2.3` | [[mod-103]] data governance |
| **A.5.2** Impact assessment process | Documented process for AI impact assessments | Art. 35 | 27701 A.7.2.5 / B.8.2.3 | `MAP-4.1` | [[exercise-01]], [[exercise-02]] |
| **A.5.3** Documentation of impact assessments | Each assessment is retained | Art. 35(7), 35(11) | 27701 A.7.2.5 / B.8.2.3 | `MAP-4.1`, `MEASURE-1.1` | [[exercise-06]] §2.1 §02 |
| **A.5.4** Impact on individuals / groups | Specifically assess individuals and groups | Art. 35(1), Recital 75 | 27701 A.7.2.5 / B.8.2.3 | `MAP-5.1` | [[exercise-01]] §subject-rights analysis |
| **A.5.5** Societal impacts | Beyond individuals — collectives / society | — (AI Act Art. 27 FRIA) | 27701 does not cover | `MAP-5.2` | [[exercise-06]] §2.7 FRIA |
| **A.6.2.2** Requirements and specification | Personal-data requirements are specified | Art. 5(1)(b)(c), 25 | 27701 A.7.2.1 / B.8.2.1 | `MAP-3.4` | [[exercise-01]] §purposes |
| **A.6.2.4** Verification and validation | Validate on personal-data constraints | Art. 25 | 27701 A.7.4.X | `MEASURE-2.X` | [[mod-104]], [[mod-106]] |
| **A.6.2.8** Event logs | Log events relevant to AI operation; personal-data events sub-slice | Art. 30, Art. 32 | 27701 A.7.3.X; 27001 A.8.15 | `MEASURE-2.6` | [[mod-110]] exercise-07 |
| **A.7.2** Data for development and enhancement | Govern the training data | Art. 5(1)(b)(c), 6, 9, 10 | 27701 A.7.4.X | `MAP-2.3`, `MEASURE-2.7` | [[mod-103]] exercise-01 datasheet |
| **A.7.3** Acquisition of data | Lawful, documented acquisition | Art. 6, 14 (notice at indirect collection) | 27701 A.7.2.3 / B.8.2.4 | `MAP-2.2` | [[mod-103]] |
| **A.7.5** Data provenance | Where did this data come from | Art. 14, Art. 30 | 27701 A.7.3.X | `MAP-2.3` | [[mod-103]] exercise-06 ML-BoM |
| **A.7.6** Data preparation | Minimisation, pseudonymisation, de-identification | Art. 5(1)(c), 25, 32 | 27701 A.7.4.5, A.7.4.6 | `MEASURE-2.7`, `MEASURE-2.9` | [[mod-104]], [[mod-106]] |
| **A.8.5** Information for interested parties | Notice / transparency to subjects | Art. 12, 13, 14; Art. 22(3) | 27701 A.7.3.2 | `GOVERN-3.2` | [[exercise-06]] §2.1 §06 |
| **A.10.2** Allocation of responsibilities | Controller / processor / joint-controller clarity | Art. 24, 26, 28 | 27701 A.7.2.6 / B.8.2.1 | `GOVERN-6.1` | [[exercise-06]] §2.1 §05 |
| **A.10.3** Suppliers | Processor assurance | Art. 28 | 27701 A.7.2.6 / B.8.5.X | `GOVERN-6.1` | [[exercise-06]] §2.1 §05 |

The crosswalk makes a non-obvious property visible: ISO 42001's
Annex A.7 is *almost entirely* already satisfied by a mature
privacy programme operating under ISO 27701 + GDPR. The AIMS
adoption, for a privacy-mature organisation, is largely about
A.5 (impact assessment extended beyond personal data), A.6
(life-cycle processes beyond the personal-data slice), A.8
(transparency about AI behaviour beyond notice), and A.9
(responsible use beyond lawful-basis analysis). The slice tells
the privacy function "here is what you already own; do not
re-author it."

### 2.4 Worked implementation — personal-data minimisation across the AI life cycle

**Objective statement** (clause 6.2): "By **`<date>`**, every
training pipeline that processes personal data enforces field-
level minimisation, de-identification where feasible, and
documented retention bounds; the per-pipeline evidence is
auditable in one place."

Simultaneous AIMS / GDPR / local-artefact coverage:

- **ISO 42001 clauses evidenced:** 6.1.4 (impact assessment);
  6.2 (objective); 7.5 (records); 8.1 (operational control);
  8.4 (operational impact assessment on material change);
  9.1 (metric).
- **Annex A controls evidenced:** A.4.3 (data resources
  catalogued); A.5.2–A.5.4 (impact assessment on
  individuals); A.6.2.2 (requirements and specification);
  A.6.2.4 (verification); A.6.2.8 (event logs); A.7.2
  (training-data governance); A.7.4 (data quality); A.7.5
  (provenance); A.7.6 (data preparation — minimisation,
  de-identification); A.8.5 (information to interested
  parties, via the system card).
- **GDPR articles evidenced:** Art. 5(1)(b) (purpose
  limitation); Art. 5(1)(c) (data minimisation); Art. 5(1)(e)
  (storage limitation); Art. 25 (data protection by design and
  by default); Art. 30 (records of processing); Art. 32
  (security of processing); Art. 35 (DPIA).
- **ISO 27701 controls evidenced:** A.7.2.1 (identify
  purposes); A.7.2.2 (determine lawful basis + records);
  A.7.4.5 (PII minimisation); A.7.4.6 (de-identification);
  A.7.4.7 (retention).
- **NIST AI RMF subcategories evidenced:** `MAP-2.3` (data
  mapping); `MAP-4.1` (impact assessment); `MEASURE-2.7`
  (data quality and privacy); `MEASURE-2.9` (de-identification);
  `GOVERN-1.4` (documentation).

Engineering artefacts a reviewer inspects (one checklist shared
across the AIMS internal auditor, the DPO, and the AI governance
lead):

- Dataset datasheet from [[mod-103]] exercise-01 for the
  training corpus — `datasheets/<corpus>.md`.
- Pipeline manifest declaring columns and joins —
  `pipelines/<name>/manifest.yaml` with
  `test_training_schema_locked` CI check.
- Consent / restriction join SQL — `pipelines/<name>/sql/*.sql`
  with the `consent_state` and `processing_restricted`
  filters.
- PII redactor config and golden tests from [[mod-104]] —
  `redactors/<pipeline>.yaml` + `tests/redactors/`.
- Pseudonymisation / DP config from [[mod-106]] where DP is
  engaged — `training/<name>/dp.yaml` with
  `epsilon`/`delta`/`accountant`.
- Retention / TTL job and dashboard —
  `jobs/<pipeline>_manifest_gc.py`; dashboard showing fraction
  of rows dropped by consent filter, drift, and retention
  queue depth.
- DPIA entry for the pipeline from [[exercise-01]] —
  `dpia/<system>.pdf` §training-data section.
- System / model card privacy slice from [[mod-109]]
  exercise-06 — `cards/<system>-privacy.md`.
- AIMS Statement of Applicability row for A.7.2, A.7.6, A.5.X
  — `aims/soa.csv` with status `applicable` and evidence
  pointer to the above.
- Clause 6.2 objective tracker — `aims/objectives.csv` with
  the metric target and the measurement cadence.

The point: the AIMS does not add new artefacts for this
objective; it adds a single row in the Statement of
Applicability pointing at existing artefacts. That is the
intended shape of an AIMS layered on a mature privacy programme.

### 2.5 Statement of Applicability — privacy slice shape

The Statement of Applicability (SoA) is the mandatory output of
clause 6.1.3. For the privacy slice, each row looks like:

```csv
control_id,title,applicable,justification,evidence_ref,owner,review_date
A.5.2,Impact assessment process,yes,"GDPR Art. 35 obligation; ICO + CNIL shapes applied","dpia/process.md; mod-111/exercise-01",DPO,2027-01-15
A.5.3,Documentation of impact assessments,yes,"Required to evidence Art. 35(7) + 35(11)","dpia/<system>.pdf; mod-111/exercise-06 bundle",DPO,2027-01-15
A.5.4,Impact on individuals / groups,yes,"GDPR + AI Act FRIA where applicable","mod-111/exercise-01 §subject-rights",DPO,2027-01-15
A.5.5,Societal impacts,yes,"AI Act Art. 27 FRIA; outside GDPR scope","mod-111/exercise-06 §09-ai-act",AI governance lead,2027-01-15
A.7.2,Data for development and enhancement,yes,"GDPR Art. 5(1)(b)(c); Art. 6/9; Art. 30","mod-103/exercise-01 datasheet; pipelines/<name>/manifest.yaml",Data platform lead + DPO,2027-01-15
A.7.3,Acquisition of data,yes,"GDPR Art. 14 (indirect-collection notice)","intake/<source>/notice.pdf",DPO,2027-01-15
A.7.5,Data provenance,yes,"GDPR Art. 30; ML-BoM","mod-103/exercise-06 ML-BoM",Data platform lead,2027-01-15
A.7.6,Data preparation,yes,"Art. 5(1)(c), 25","mod-104/*; mod-106/* where DP engaged",Data platform lead + DPO,2027-01-15
A.8.5,Information for interested parties,yes,"GDPR Art. 12–14; Art. 22(3)","privacy-notice.html; mod-109/exercise-06 card",DPO + comms,2027-01-15
A.10.2,Allocation of responsibilities,yes,"GDPR Art. 24/26/28","mod-111/exercise-06 §05-vendors",DPO + legal,2027-01-15
A.10.3,Suppliers,yes,"GDPR Art. 28","mod-111/exercise-06 §05-vendors",DPO + legal,2027-01-15
```

Non-applicable rows (e.g., controls that only apply when the
organisation offers general-purpose AI under AI Act Art. 51–55,
and this organisation does not) carry an explicit
`applicable=no,justification=<reason>`. A silent omission is a
non-conformity.

### 2.6 Governance mapping (NIST AI RMF overlay)

Every row in §2.3 carries an AI RMF function. The slice as a
whole covers:

- **Govern** — A.2.3 (policy alignment), A.10.2 / A.10.3
  (allocation + suppliers), clause 7.5 (records of processing).
- **Map** — A.4.3 (data resources), A.7.2 / A.7.3 / A.7.5
  (data origin and provenance), A.5.4 (who is impacted).
- **Measure** — A.5.2 / A.5.3 (impact assessment), A.6.2.4
  (verification), A.6.2.8 (event logs), A.7.6 (data
  preparation), A.8.5 (information disclosure).
- **Manage** — clause 8.4 (operational re-assessment), clause
  10.2 (nonconformity + corrective action).

### 2.7 Internal-audit preparation

ISO 42001 clause 9.2 requires internal audits. The privacy slice
hands the auditor four things:

1. The **SoA privacy rows** (§2.5) with evidence pointers.
2. The **crosswalk** (§2.3) so the auditor sees each AIMS
   obligation side-by-side with its GDPR / 27701 / AI RMF
   neighbour. The auditor stops asking "is this covered?" and
   asks "is the evidence current?"
3. A **privacy-slice evidence trail**: the one-paragraph
   explanation per row of where the evidence lives, who signed
   it, and when it was last reviewed.
4. **Nonconformity log** filtered to privacy-slice issues —
   open items, with owners and due dates.

### 2.8 Interaction with other frameworks

- **ISO/IEC 27001 + 27701.** 42001 inherits the management-
  system shell from 27001. An organisation certified to 27001 +
  27701 has clauses 4–7, 9, 10 largely in place; it needs to
  (a) extend the ISMS/PIMS scope to the AIMS, (b) add the AIMS
  Annex A controls not already covered, and (c) author an AIMS
  SoA. The privacy slice overlaps 27701 heavily (see §2.3
  columns 3 and 4).
- **NIST AI RMF.** The Govern / Map / Measure / Manage
  functions are not a management-system standard and do not
  substitute for 42001; they are the measurement side. An
  organisation doing both uses AI RMF as the operational
  playbook under the AIMS.
- **GDPR / UK GDPR.** 42001 does not confer GDPR compliance.
  An AIMS without a DPIA, a legal-basis assessment, and the
  Art. 28 processor chain is non-conformant against clause
  6.1.4 and A.5.X. Conversely, GDPR compliance alone does not
  satisfy 42001 — the shell and A.6 life-cycle controls are
  not required by GDPR.
- **EU AI Act.** 42001 is a management-system standard; the AI
  Act is a product-regulation regime. A high-risk system under
  the AI Act must comply with Chapter III Section 2 regardless
  of whether the operator is AIMS-certified. The slice calls
  out A.5.5 (societal impacts) as the hook where FRIA (AI Act
  Art. 27) plugs in.
- **Sector regimes (HIPAA, FERPA, COPPA, GLBA).** 42001 does
  not displace them; the SoA lists sector regimes in the
  justification column of relevant rows (A.5.4, A.7.2, A.8.5,
  A.10.3).

### 2.9 Threat model (slice-specific)

- **Audit drift.** The most common failure mode: the SoA cites
  evidence that existed at certification and has since gone
  stale. Mitigated by the per-row `review_date` and by
  binding evidence pointers to pinned versions (as in
  [[exercise-06]] §2.4 manifest).
- **Parallel systems.** The organisation stands up the AIMS as
  a separate workstream and ends up maintaining two risk
  registers. Mitigated by the crosswalk (§2.3) forcing one
  evidence path per obligation.
- **Scope gap.** The AIMS scope omits a system that processes
  personal data (e.g., "it's just an internal tool"). The
  privacy slice catches this because every system processing
  personal data must appear in A.4.3 and A.7.X.
- **Non-applicability abuse.** Rows marked `applicable=no`
  without defensible justification. The reviewer checklist
  (§4) flags unjustified non-applicability.
- **MITRE ATLAS tactic `AML.T0010` (ML Supply Chain
  Compromise).** A.10.3 (suppliers) is the AIMS hook; the
  privacy slice extends the diligence to Art. 28 processor
  obligations.
- **OWASP ML Top 10 ML06 (AI Supply Chain Attacks).** Same
  hook, product perspective; the slice ties the mitigation to
  A.7.5 (provenance) and A.10.3 (suppliers).

## 3. Validation steps

A reviewer runs this checklist against the slice before it is
packaged into the regulator bundle ([[exercise-06]] §10-governance)
or offered to a certification body.

1. **Clause coverage.** Every ISO 42001 clause in §2.2 marked
   **P** or **C** has a row in the slice with an evidence
   pointer. No **P** row is missing.
2. **Annex A coverage.** Every Annex A control marked **P** in
   §2.2 appears in the SoA with `applicable=yes` or a
   defensible `applicable=no` + justification.
3. **Crosswalk completeness.** Every **P** row has at least
   one GDPR article, one ISO 27701 control (if the
   organisation is 27701-certified), and one NIST AI RMF
   subcategory in the crosswalk.
4. **Evidence pointers resolve.** Every `evidence_ref` in the
   SoA points to an artefact that exists and is on its
   current version.
5. **Owner is a role, not a person.** Each row lists the role
   accountable; the role resolves to a named individual in a
   separate RACI.
6. **Review dates are ≤ 12 months out.** Any date past 12
   months is a slice-level failure; privacy evidence stales
   fast.
7. **Objective is measurable.** The clause 6.2 objective
   (§2.4) has a target value, a metric definition, and a
   measurement cadence.
8. **Non-applicability justified.** Every `applicable=no` row
   carries a justification that cites a specific condition
   (e.g., "organisation does not deploy GPAI models").
9. **AIMS scope matches privacy scope.** Every system in the
   organisation's privacy-processing inventory ([[mod-103]])
   that uses AI is in the AIMS scope statement (clause 4.3).
10. **Impact assessment loop closed.** For each system in
    scope, the slice points to a DPIA / impact assessment and
    to the next-review date (A.5.3 + clause 8.4).
11. **Processor coverage.** Every Art. 28 processor the
    privacy function tracks ([[exercise-06]] §05-vendors)
    appears as a supplier in A.10.3; any gap is a
    non-conformity.
12. **Information-to-subjects coverage.** A.8.5 points to the
    privacy notice + any AI-specific disclosures (model card
    privacy slice from [[mod-109]] exercise-06).

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| ISO 42001 clauses and Annex A controls tagged **P/C/—** with reasoning | 2 | §2.2 — missing the tagging is the most common submission failure |
| Crosswalk present with GDPR + ISO 27701 + NIST AI RMF + local artefact columns for every **P** row | 2 | §2.3 |
| Statement of Applicability for privacy slice with evidence pointers, owners, review dates | 2 | §2.5 |
| Worked implementation of one clause-6.2 objective with simultaneous AIMS / GDPR / 27701 / AI RMF coverage | 1 | §2.4 |
| Governance overlay (NIST AI RMF function per row) | 1 | §2.6 |
| Internal-audit preparation package | 1 | §2.7 |
| Interaction with 27001/27701, NIST AI RMF, GDPR, AI Act, sector regimes explicit | 1 | §2.8 |

Automatic failures:

- No Statement of Applicability for the privacy slice.
- Any **P** control missing from the SoA.
- SoA rows with person names instead of roles as owners.
- Review dates > 12 months.
- Any `applicable=no` without a defensible justification.
- "GDPR compliant → ISO 42001 compliant" or vice versa claimed
  anywhere in the slice.
- Scope statement (clause 4.3) that excludes an AI system
  processing personal data.

## 5. Common mistakes

- **Rewriting GDPR evidence for the AIMS.** The whole point of
  the slice is to reuse existing privacy artefacts. If the
  AIMS evidence package contains fresh DPIAs that duplicate
  the ones in [[exercise-01]], the organisation is maintaining
  two systems. The SoA points at the existing artefacts.
- **Treating 42001 as a privacy standard.** 42001 is an AI
  management-system standard; the privacy slice is a subset.
  A DPO who takes over the full AIMS mis-scopes the project —
  non-privacy A.6 and A.9 controls belong to the AI governance
  lead.
- **Writing the SoA with person names.** Owners are roles.
  People change jobs; roles persist. The RACI resolves roles
  to people out of band.
- **Marking A.5.5 as not applicable.** Societal impacts are
  rarely zero for AI systems that touch personal data. The
  slice should justify any `applicable=no` with a specific
  condition (e.g., "closed-loop B2B system with no external
  deployment and no decisions affecting individuals outside
  the contracting organisation") — otherwise flip to `yes`
  and point at the FRIA or at a societal-impact memo.
- **Misreading 42001's certifiability.** 42001 is certifiable
  but, like 27001, certification is of the management system,
  not of individual AI products. A "42001-certified system"
  is nonsense; a "42001-certified organisation operating the
  system under its AIMS scope" is correct. The slice should
  not promise what the standard cannot deliver.
- **Letting AIMS scope drift from privacy scope.** If the AIMS
  scope is narrower than the privacy processing inventory,
  there is an AI system processing personal data outside the
  AIMS — a scoping defect. If the AIMS scope is wider, the
  privacy slice lists it and marks the extra as "non-
  personal-data processing" so the slice does not inflate.
- **Omitting the AI Act hook.** A.5.5 (societal impacts) is
  where FRIA plugs in, and A.6.2.X / A.8.5 are where AI Act
  Chapter III (high-risk) obligations cross. A slice that
  does not reference the AI Act where applicable is
  incomplete.
- **Pinning to the wrong 42001 edition.** ISO/IEC 42001:2023
  is the current edition at the time of writing; any future
  amendments change clause/control numbering. The SoA must
  state the edition.
- **Confusing Annex A with mandatory.** 42001's Annex A is a
  reference control set; applicability is declared in the SoA.
  Treating every Annex A control as mandatory inflates the
  slice and creates spurious non-conformities. Conversely,
  treating applicability as discretionary without justification
  is also a defect.
- **Skipping clause 8.4.** The operational re-assessment loop
  is the AIMS equivalent of "the DPIA is a living document."
  A slice without a clause-8.4 trigger (material change →
  re-assessment) silently passes the first audit and fails the
  second.
- **No measurable objective.** Clause 6.2 wants targets, not
  aspirations. "Improve privacy posture" is not an objective;
  "By **`<date>`**, 100% of training pipelines processing
  personal data have an SoA row with evidence ≤ 12 months
  old" is.
- **Letting the slice stand alone.** The slice is a *view* of
  the AIMS, not a parallel document. It lives inside
  `10-governance/iso-42001-slice.pdf` of the regulator
  bundle ([[exercise-06]] §2.1) and references the SoA in the
  AIMS proper.

## 6. References

Primary standards (versions pinned in the SoA front-matter):

- ISO/IEC 42001:2023 — Information technology — Artificial
  intelligence — Management system.
  <https://www.iso.org/standard/81230.html>
- ISO/IEC 27001:2022 — Information security management
  systems.
  <https://www.iso.org/standard/27001>
- ISO/IEC 27701:2019 — Extension to ISO/IEC 27001 and ISO/IEC
  27002 for privacy information management.
  <https://www.iso.org/standard/71670.html>
- NIST AI Risk Management Framework 1.0 — Govern / Map /
  Measure / Manage.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST Privacy Framework 1.0 — Identify-P / Govern-P /
  Control-P / Communicate-P / Protect-P.
  <https://www.nist.gov/privacy-framework>
- Regulation (EU) 2016/679 — General Data Protection
  Regulation (Articles 5, 6, 9, 12, 13, 14, 22, 24, 25, 26,
  28, 30, 32, 35).
  <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- Regulation (EU) 2024/1689 — Artificial Intelligence Act
  (Articles 9–15, 27, 51–55).
  <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- OWASP Machine Learning Security Top 10 (ML06 AI Supply
  Chain, ML04 Data Poisoning, ML10 Model Theft among others).
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS (`AML.T0010 ML Supply Chain Compromise`,
  `AML.T0019 Publish Poisoned Datasets`,
  `AML.T0024 Exfiltration via ML Inference API`).
  <https://atlas.mitre.org/>

Related solutions in this track: [[exercise-01]] (Art. 35
DPIA), [[exercise-02]] (ICO DPIA shape), [[exercise-03]] (NIST
PRAM), [[exercise-04]] (EDPB Opinion 28/2024 + CNIL sheets),
[[exercise-05]] (HIPAA trigger map), [[exercise-06]] (regulator-
facing bundle). Cross-references: [[mod-101]] exercise-04 (NIST
Privacy Framework + ISO 27701 crosswalk); [[mod-103]] (data
governance, datasheets, ML-BoM); [[mod-104]] (PII detection);
[[mod-106]] (differential privacy); [[mod-109]] exercise-06
(system card privacy slice); [[mod-110]] exercise-07 (DSR audit
trail); [[mod-112]] (privacy program and communication).
