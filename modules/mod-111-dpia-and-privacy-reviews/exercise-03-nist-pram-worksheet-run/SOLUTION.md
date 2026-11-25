# Exercise 03 — NIST PRAM Worksheet Run — Reference Solution

## 1. Solution overview

This is an **analysis + worksheet-authoring exercise**. The learner
is asked to run the NIST Privacy Risk Assessment Methodology (PRAM)
end-to-end for an AI system — Worksheet 1 (Framing), Worksheet 2
(Data Action and Context Assessment), Worksheet 3 (Prioritisation),
and Worksheet 4 (Response) — and to produce a NIST Privacy Framework
(PF) Subcategory × Artefact matrix that an auditor accepts as the
evidence pack.

The PRAM is the practical operationalisation of the NIST Privacy
Framework v1.0 for US-federal and US-regulated-enterprise programmes;
it is the complement to an EU-shape DPIA ([[exercise-01]],
[[exercise-02]]). The PRAM's distinctive move is to think in **data
actions** (fine-grained operations the system performs on personal
data) and to apply the **nine problematic-data-action taxonomy**
(Appropriation, Distortion, Induced Disclosure, Insecurity,
Re-identification, Stigmatization, Surveillance, Unanticipated
Revelation, Unwarranted Restriction) to each data action. The output
chains into the DPIA risk register, the regulator-facing bundle
([[exercise-06]]), the SP 800-53 PT-family mapping ([[exercise-05]]),
and the AI RMF overlay ([[exercise-07]]).

The assessable artifact: four markdown worksheets plus the PF matrix,
in a single directory.

```
pram/
├── w1-framing.md
├── w2-data-actions.md       # 15 rows typical
├── w3-prioritisation.md     # ranked risk list, ~25–35 entries
├── w4-response.md           # 1 row per W3 entry
└── pf-matrix.md             # Subcategory × Artefact matrix
```

Grounded in: NIST Privacy Framework v1.0 (2020); NIST PRAM resource
page and workbook; NISTIR 8062 (2017); NIST AI Risk Management
Framework 1.0 (2023); NIST AI 600-1 (2024, Generative AI Profile);
NIST SP 800-53 Rev. 5 (PT and SI families). The worked example
reuses the **Support-RAG v2.1** AI system introduced in
[[exercise-01]] so the learner can hold the EU-shape DPIA and the
PRAM output side-by-side.

## 2. Worked answer

### 2.1 Worksheet 1 — Framing

```
System name:            Support-RAG v2.1
System description:     Customer-support retrieval-augmented LLM assistant
                        for agents. Agent reviews every draft before sending.

System owner:           Product Owner, CX Platform
Privacy lead:           Privacy Engineering Team
Mission relevance:      Support function efficiency; consistent
                        response quality; agent coaching.
Business value:         ~23s / ticket time saving; measured in
                        eval/ticket-time-2026-08.md.

Applicable laws and regulations (compliance perimeter):
  - GDPR / UK GDPR (EEA + UK subjects)
  - CCPA / CPRA (California subjects)
  - Virginia CDPA, Colorado CPA, Connecticut CTDPA, Utah UCPA,
    Texas TDPSA (other US state subjects, as applicable)
  - HIPAA (not currently in scope — no BAA — but noted because
    tickets occasionally include health content; the controller's
    policy is to refuse HIPAA-covered-entity business absent a
    BAA + a specific DPIA addendum)
  - FTC Act §5 (unfair/deceptive — governs representations about
    AI use)
  - SEC Regulation S-P (not in scope — no broker-dealer customer
    data)

Organisational privacy principles:  Acme internal privacy policy
                                     maps to the FIPPs; transparency,
                                     purpose limitation, data
                                     minimisation, participation,
                                     accountability.

Privacy risk statement:
  The organisation seeks to avoid harm to data subjects arising
  from the processing of support-ticket content by Support-RAG,
  including unauthorised disclosure, inferred-attribute
  disclosure, re-identification, surveillance, and loss of
  subject control; and to comply with the privacy laws listed
  above.

Stakeholders considered:
  - End users (subjects who submit tickets)
  - Non-user subjects (third parties occasionally mentioned in
    ticket content — e.g., the subject's family member)
  - Support agents (employees whose draft quality is observed)
  - Acme as controller and group-affiliate processors
  - Anthropic as sub-processor
  - Regulators (ICO, CNIL, EU-DPAs, FTC, state AGs)
  - Public interest in trustworthy AI deployment
```

Worksheet 1 is the organisation-strategic frame. Note two
AI-specific points: the **non-user subject** category is called out
explicitly (one subject's ticket may mention another individual —
family member, business contact — whose privacy interests are
engaged); and the **HIPAA-absent posture** is documented as a
deliberate choice, not an accidental scope omission.

### 2.2 Worksheet 2 — Assessing Data Actions

For Support-RAG v2.1, the data-action decomposition yields 15 rows.
The structure of a single row is below, then the full compressed row
set follows.

#### 2.2.1 Example full row — data action B2 (Indexing / Embedding)

```
Data action ID: B2
Data action: Chunk subject-authored ticket content into 256-token
             passages; embed with bge-large-en-v1.5; persist to
             Qdrant with metadata (subject_id, ticket_id, source
             URL, created_at). ~3M chunks at ingest; ~50K new per
             week. Index refreshed hourly.

Data elements:
  - Ticket body (subject-authored free text; variable sensitivity
    — occasional health, safety, Art. 9 context)
  - subject_id (pseudonymous identifier; linkable to account)
  - Embedding vector (derived from ticket content)
  - Metadata (ticket_id, source URL, created_at)

Context:
  - Subject's reasonable expectation on ingest: agent reads the
    ticket. Expectation about use as retrieval material is weaker
    without the privacy-notice disclosure; 31% anticipated this
    use per the 2026-05 consultation panel.
  - Visibility of the action to the subject: low (background
    indexing is not surfaced).
  - Subject's control: Art. 21 objection route supported via
    opt-out on purpose #2 (topic analytics) — but the retrieval
    index for purpose #1 is service-delivery under Art. 6(1)(b)
    and is not opt-outable at the retrieval level; subject can
    close account and exercise Art. 17 erasure.

Problematic data actions (nine-taxonomy):
  - Appropriation — the use for AI retrieval is beyond the
    original ticket-handling expectation; the subject-consultation
    panel confirms this gap.
  - Insecurity — the embedding store is a confidentiality-sensitive
    asset.
  - Re-identification — nearest-neighbour search on embeddings can
    recover the subject behind a near-duplicate chunk even if the
    operator does not know subject_id; model-inversion attacks on
    the embedding space are a known research area.
  - Unanticipated revelation — RAG can surface a document the
    subject may have forgotten, to a support agent who did not
    originally handle the ticket.
  (Not applicable: Induced disclosure — not an active-interaction
  data action; Distortion — no inference here; Stigmatization —
  no attribute inference at this step; Surveillance — this is
  storage, not monitoring; Unwarranted restriction — not a decision
  point.)

Problems for individuals:
  - Loss of control over use of support-ticket content for a
    purpose the subject did not clearly expect.
  - Disclosure of sensitive content to a support agent outside the
    original support context.
  - Potential inferred-attribute exposure if the model output
    summarises patterns across tickets.
  - Risk of re-identification via embedding similarity (low under
    current access controls; non-zero).

Problems for the organisation:
  - Regulatory: GDPR Art. 5(1)(b) purpose-limitation challenge if
    retrieval is treated as a new purpose.
  - Reputational: consumer backlash if the practice is perceived
    as overreach without clear notice.
  - Operational: data-quality issues in the embedding corpus
    propagating into draft quality.
```

#### 2.2.2 Compressed row set (all 15 data actions)

| ID | Data action | Problematic data actions (nine-taxonomy) | Load-bearing problems |
|---|---|---|---|
| A1 | Training-data collection (for self-trained models) — **N/A here**, we rely on hosted model | — | — |
| A2 | Inference-input collection — ticket text, attachments, account context at submit time | Appropriation (occasional), Insecurity | Subject may disclose more than expected if the UI prompts them to |
| A3 | Operational telemetry collection — `llm_calls` logs prompt + completion for 14 days | Surveillance, Insecurity | Retention-driven surveillance risk; log leak |
| B1 | Training — **N/A here** (hosted model) | — | — |
| B2 | Indexing / embedding into Qdrant | Appropriation, Insecurity, Re-identification, Unanticipated revelation | See §2.2.1 above |
| B3 | Inference — retrieval + hosted LLM call | Appropriation, Distortion, Induced disclosure, Insecurity, Re-identification, Stigmatization, Unanticipated revelation | Load-bearing — most PDAs apply at the generative step |
| B4 | Logging and analytics on telemetry | Surveillance, Insecurity | Observability risk |
| B5 | Derivative-product production — topic analytics MV (purpose #2); agent coaching highlights (purpose #3) | Appropriation, Distortion (topic mislabel), Re-identification (if clusters small), Stigmatization (inferred sensitive attribute), Surveillance (purpose #3 against employees) | Re-identification at low k; employee-coaching surveillance |
| B6 | Internal sharing to downstream teams — product research reads topic MV | Appropriation, Unanticipated revelation | Internal sharing beyond originally-stated audience |
| C1 | Disclosure to the subject — agent sends the (reviewed) draft | Distortion (if inaccurate), Induced disclosure (subject might reply with more sensitive info), Unanticipated revelation | Subject-side harm from inaccurate advice |
| C2 | Disclosure to third-party sub-processor — Anthropic receives prompt | Insecurity, Re-identification | Vendor incident; cross-border transfer |
| C3 | Disclosure through model output to another subject — agent accidentally pastes retrieved content from another subject's ticket into current reply | Insecurity, Re-identification, Unanticipated revelation | Confidentiality breach across subjects |
| C4 | Release of derivative artefacts — **N/A** (no public release) | — | — |
| C5 | Cross-border transfers — EEA→US tenant leakage, or EEA data to Anthropic | Insecurity, Re-identification | Chapter V / Schrems II exposure |
| A4 (added) | Annotation / RLHF / human-in-the-loop labels on draft quality (purpose #3) | Appropriation (employee), Surveillance (employee), Insecurity | Employee-monitoring and data-handling obligations |

Fifteen data actions for a nontrivial AI system is in the right band.
Fewer would mean aggregation across phases that have distinct risk
profiles. (A1/B1/C4 are marked N/A here because this system does
not self-train and does not publish derivative artefacts; for a
different system each would be its own row.)

### 2.3 Worksheet 3 — Prioritising Privacy Risks

Each (data action, problematic data action) pair scored on
**likelihood**, **impact on individuals**, and **impact on
organisation** on a 1–5 scale. Composite rank by `max(individual,
organisation) × likelihood`; the top entries drive W4. Twenty-eight
entries total; the top 12 are shown.

| Rank | Data action | PDA | Likelihood | Impact on individuals | Impact on organisation | Composite (ind × lkl) | Composite (org × lkl) |
|---|---|---|---|---|---|---|---|
| 1 | C3 | Re-identification (cross-subject retrieval leak) | 4 | 4 | 4 | 16 | 16 |
| 2 | B3 | Insecurity (prompt-injection exfiltration) | 4 | 4 | 4 | 16 | 16 |
| 3 | B3 | Re-identification (memorisation / extraction) | 2 | 4 | 4 | 8 | 8 |
| 4 | B3 | Stigmatization (inferred Art. 9 attribute in output) | 3 | 4 | 3 | 12 | 9 |
| 5 | A3 / B4 | Surveillance (log retention > operational need) | 3 | 3 | 3 | 9 | 9 |
| 6 | B5 | Re-identification (topic cluster with low k) | 2 | 3 | 3 | 6 | 6 |
| 7 | B5 | Stigmatization (inferred health / political label) | 3 | 4 | 3 | 12 | 9 |
| 8 | B5 (purpose #3) | Surveillance (agent coaching) | 3 | 3 | 2 | 9 | 6 |
| 9 | C1 | Distortion (hallucinated product advice to health-adjacent subject) | 3 | 5 | 4 | 15 | 12 |
| 10 | C2 | Insecurity (vendor incident at Anthropic) | 2 | 4 | 4 | 8 | 8 |
| 11 | C5 | Insecurity (cross-border transfer misconfig) | 1 | 4 | 5 | 4 | 5 |
| 12 | B2 | Appropriation (RAG use beyond expectation) | 4 | 2 | 3 | 8 | 12 |

**Note on the two-impact discipline.** The PRAM keeps "impact on
individuals" and "impact on organisation" separate — the top risks
for the organisation are not always the top risks for individuals.
Here, rank 11 (cross-border transfer misconfig) is low-likelihood but
very high organisational impact (Schrems II–style enforcement); it
would not appear in the DPIA's top rows under subject-harm scoring
alone. The chapter-03 lesson insists on this separation; keep both
columns.

**Scoring traceability.** Each cell's score cites evidence:

- Likelihood 4 on rank 1 (C3): near-duplicate-retrieval monitor at
  v2.0 pre-fix measured 0.08 cross-subject leak rate; not a
  speculative number.
- Impact-individual 5 on rank 9 (C1 Distortion): wrong advice to a
  health-adjacent subject could drive medication error — severe,
  irreversible in some scenarios.
- Likelihood 2 on rank 10 (C2 Insecurity): based on Anthropic's SOC
  2 Type II + audited incident rate in comparable vendors.

A scorer who cannot explain the rationale from data fails the
traceability discipline.

### 2.4 Worksheet 4 — Response

Each prioritised entry gets one response row. The response names:
option (avoid / mitigate / transfer / accept); specific controls
cited to artefacts; the NIST SP 800-53 Rev. 5 control tags; the NIST
PF Subcategory tags; residual (likelihood × impact); owner; due
date.

| W3 rank | Response | Specific controls (artefact) | SP 800-53 | NIST PF | Residual (L × I_ind) | Owner | Due |
|---|---|---|---|---|---|---|---|
| 1 | Mitigate | Retrieval reranker with per-subject-partition penalty (`eval/retrieval-partition-2026-09.md`); output filter (`eval/output-filter-v2.1.md`); near-duplicate monitor | AC-3, AC-16, SI-15, SI-19 | CT.DM-P1, CT.DP-P4, PR.AC-P1 | 2 × 3 | ML Lead | 2026-09-11 (done) |
| 2 | Mitigate | System-prompt hardening; jailbreak classifier; indirect-injection probe set quarterly (`eval/red-team-2026-09.md` ASR 0.07) | SI-10, SI-15 | CT.DP-P4, PR.PO-P5 | 2 × 4 | Sec Eng | 2026-09-12 (done); v2.2 second-stage detector |
| 3 | Mitigate | Output filter blocks ≥ 50-token verbatim matches; MIA audit (`reports/mia-2026-09.md` TPR@FPR=0.01 0.04 vs threshold 0.10); hosted-model no-training | SI-15, SI-19 | CT.DP-P4, PR.PO-P5 | 1 × 4 | Priv Eng | 2026-09-10 (done) |
| 4 | Mitigate | Special-category output filter + topic-cluster k ≥ 50 + severe-advice classifier gate on C1 | SI-15, SI-19 | CT.DP-P4, PR.PO-P5 | 1 × 4 | Priv Eng | 2026-09-11 (done) |
| 5 | Mitigate | Retention reduced 180 → 14 days (`docs/retention-change-2026-08.md`); access-logged; monthly review | AU-11, SI-12 | PR.DS-P1, PR.DS-P2 | 2 × 3 | Sec Eng | 2026-09-10 (done) |
| 6 | Mitigate | k ≥ 50 cluster floor; small-cluster suppression; k-measure log 2026-06 | SI-19 | CT.DP-P4 | 1 × 3 | Priv Eng | 2026-06-30 (done) |
| 7 | Mitigate | As rank 4 + explicit disclosure in Art. 13/14 notice | SI-15, PT-5 | CT.PO-P1, CM.PO-P1 | 2 × 4 | Priv Eng | 2026-09-11 (done) |
| 8 | Mitigate | Workforce LIA (Appx E); WP29 2/2017 engagement; union agreement (Appx E-2); limit coaching signal to aggregate + opt-in for named highlights | AC-3, PT-3 | GV.PO-P1, GV.AT-P1, CT.DP-P4 | 2 × 2 | HR Ops + Priv Eng | 2026-08-15 (done) |
| 9 | Mitigate | Severe-advice classifier; health-adjacent prompts receive hard gate (agent-explicit-confirm + supervisor review); weekly sample audit | SI-10, SI-15 | PR.PO-P5, CT.PO-P1 | 2 × 4 | CX Ops + Priv Eng | 2026-09-12 (done) |
| 10 | Transfer (contract) + Mitigate | Anthropic DPA with breach-notification clause; SCCs Module 2; EU residency; vendor-risk annual review | AC-21, IR-6 | ID.DE-P1, PR.AC-P1 | 2 × 4 | Legal + Priv Eng | 2026-05-14 (done) |
| 11 | Mitigate | Tenant-isolation automated monthly test (`tests/tenant-isolation-2026-09.md`); region-locked buckets; CMEK per tenant | AC-3, AC-16, SC-7 | CT.DM-P1, PR.AC-P1 | 1 × 4 | SRE Lead | 2026-09-10 (done) |
| 12 | Mitigate (notice) | Art. 13/14 notice updated to disclose AI; opt-out on purpose #2; consultation panel annually | PT-5 | CM.PO-P1, GV.PO-P1 | 3 × 2 | Priv Eng | 2026-05-30 (done) |

For each rank 13–28 (not shown here for brevity), the controller
fills the same row structure. The W4 output is what the DPIA Step 6
mitigation table ([[exercise-02]]) distils; the mapping is
one-to-many because a single engineering control usually addresses
multiple ranked entries.

**Response discipline.**

- **Implemented, not promised.** Every "Mitigate" row cites a
  present-tense artefact. Rows whose control is on the roadmap are
  flagged "Planned — target date" rather than described as "Applied".
  Rank 2's v2.2 second-stage detector is correctly represented as
  Planned.
- **Avoid vs Mitigate.** We did not use "Avoid" responses for the
  ranks shown; a different system might (e.g., "Avoid: do not log
  prompts at all"). Transferring via contract (rank 10) is paired
  with Mitigate — contractual transfer does not substitute for
  engineering controls on information you still handle.
- **Accept with signer.** Where acceptance is the response, the
  signer and the reason must be named; we have no "Accept" rows here
  because every ranked entry has either a mitigating control or a
  pre-committed engineering plan.

### 2.5 The NIST Privacy Framework matrix — Subcategory × Artefact

The PF matrix is the evidence pack the auditor expects. Each
applicable Subcategory is a row; each engineering artefact is a
column; cells contain the W4 response entries that bind the two.

The table below shows a compressed form; a full matrix for a mature
programme is 30–60 rows (all applicable Subcategories across the
five Functions). Non-applicable Subcategories are explicitly marked
N/A with a one-line justification so the auditor can see the scope
was considered.

| Subcategory | Artefact | W3 ranks served |
|---|---|---|
| **ID.IM-P1** systems/products/services inventoried | RoPA entry; DFD | — (foundation) |
| **ID.IM-P2** owners/operators of systems inventoried | DPIA header; bundle manifest | — |
| **ID.BE-P3** priorities for organisational mission established | W1 framing | — |
| **ID.RA-P1** context relevant to privacy risk assessed | W1 framing; subject consultation Appx B | 12 |
| **ID.RA-P3** data processing ecosystem is mapped | DFD; processor register | 10 |
| **ID.RA-P4** data processing-related problems for individuals and the organisation identified | W2 + W3 | all |
| **GV.PO-P1** organisational privacy values and policies | Organisational privacy policy | 8, 12 |
| **GV.PO-P6** privacy procedures regularly reviewed | DPIA refresh-trigger list | — |
| **GV.AT-P1** workforce informed | Training records; WP29 2/2017 engagement | 8 |
| **GV.MT-P6** policies/processes reviewed and updated | DPIA change log; incident-review memo | — |
| **CT.PO-P1** data processing policies, processes, procedures | Per-purpose feature gating; LIA Appx D, E | 7, 9 |
| **CT.DM-P1** data elements can be accessed, reviewed, corrected, deleted | DSR runbooks (mod-110 ex-01); DSR audit trail (mod-110 ex-07) | 11 |
| **CT.DP-P1** data processing managed consistent with risk strategy | PET deployment claims | — |
| **CT.DP-P4** disassociated processing (DP, k-anonymity, etc.) | Redaction report; k-anon on topic clusters | 1, 2, 3, 4, 6, 7 |
| **CM.PO-P1** transparency policies | Art. 13/14 notice; subject-facing consent / opt-out | 7, 12 |
| **CM.PO-P3** data processing-related awareness activities | Subject consultation panel; privacy-notice uplift | 12 |
| **PR.PO-P5** protective technology vulnerabilities identified and remediated | Red-team report; MIA audit | 2, 3, 4, 9 |
| **PR.AC-P1** identity management + access control | IAM + MFA + audit log | 1, 10, 11 |
| **PR.DS-P1** data at rest protected | CMEK; encryption policy | 5, 10, 11 |
| **PR.DS-P2** data in transit protected | TLS; SCCs for vendor transit | 10, 11 |
| **PR.PO-P9** response plans in place | Breach-response plan ([[exercise-05]]) | 10 |
| **PR.MA-P1** maintenance is performed | Retention enforcement logs | 5 |

**N/A Subcategories with justification:**

- **PT-6 / System of Records Notice** — not a federal system; SORN
  does not apply.
- **PT-8 / Computer Matching** — no computer-matching under the US
  Privacy Act.
- **CT.DP-P2 / third-party data-processing is authorised** — scoped
  by the processor register under Art. 28; cross-tagged there
  instead.

### 2.6 AI RMF overlay

For each W4 entry, cross-tag the applicable NIST AI RMF Subcategory:

| W3 rank | AI RMF Subcategory tag |
|---|---|
| 1 | MANAGE 2.3 (plan for handling risks from the AI system in production) |
| 2 | MEASURE 2.6 (monitor and quantify AI risks); AI 600-1 "info security" |
| 3 | MEASURE 2.6; AI 600-1 "data privacy" |
| 4 | MEASURE 2.6; AI 600-1 "harmful bias" |
| 5 | MANAGE 2.3; GOVERN 1.5 |
| 6 | MEASURE 2.6; MANAGE 2.3 |
| 7 | MEASURE 2.6; AI 600-1 "data privacy" |
| 8 | GOVERN 2.1 (roles and responsibilities); MANAGE 2.3 |
| 9 | MEASURE 2.7 (performance in pre-deployment and operation); AI 600-1 "confabulation" |
| 10 | MANAGE 4.3 (supplier risk management) |
| 11 | MANAGE 2.3 |
| 12 | GOVERN 1.4 (external stakeholder engagement) |

The AI RMF overlay doubles the matrix columns (PF Subcategory + AI
RMF Subcategory) and is what an audience that uses AI RMF as its
primary framework expects.

### 2.7 Divergence from the DPIA — and how to run both

The PRAM and the DPIA are not substitutes. The PRAM's distinctive
contributions beyond the DPIA:

- **The nine-PDA taxonomy** names problems (e.g., Appropriation,
  Stigmatization, Surveillance) the DPIA's Art. 35(7)(c) risk-to-
  rights-and-freedoms section can gloss. The DPIA's feared-event
  register benefits from the PDA tag per row.
- **The two-impact separation** (individual vs organisation) keeps
  organisational-harm-only risks visible; the DPIA can under-weight
  them.
- **The explicit SP 800-53 control tagging** gives the US-federal
  audience an artefact in its native vocabulary.

The engineer runs the PRAM as the working document, distils the DPIA
from it. The DPIA's Appendix-3 citation points at the PRAM; a
divergence between the two is a red flag an auditor will exploit.

## 3. Implementation / artifact shape

```
pram/
├── w1-framing.md
├── w2-data-actions.md
├── w3-prioritisation.md
├── w4-response.md
├── pf-matrix.md
└── ai-rmf-overlay.md
```

Each file is markdown; versioned alongside the system in git; hashed
into the bundle manifest ([[exercise-06]]); refreshed on the same
cadence as the DPIA. If a GRC tool is in use (OneTrust / Drata /
Vanta), the markdown is the authoritative source and the GRC tool
imports it, not the reverse — a drift between a controller's
engineering source of truth and a GRC tool's state is a tag that
will surface in audit.

## 4. Validation steps

1. **W1 framing complete.** System description, owner, mission,
   business value, applicable-laws list, organisational principles,
   privacy risk statement, and stakeholder enumeration (including
   non-user subjects). A W1 that omits the law list or the
   stakeholder set is incomplete.
2. **W2 exhaustive on data actions.** Check: every point where
   personal data enters (A*), moves (B*), or leaves (C*) appears
   somewhere. For an AI system the typical count is 10–20. Fewer
   suggests aggregation; many more suggests over-decomposition.
3. **W2 PDAs applied intellectually.** No row has every PDA ticked;
   each tick carries a one-sentence rationale. "All nine apply"
   is a sign of lazy work and a reviewer will push back.
4. **W3 scoring is traceable.** Each likelihood / impact cell cites
   a measurement or a comparable-system baseline. Opinions without
   evidence are a fail.
5. **W3 two-impact separation.** The individual-impact and
   organisation-impact columns are distinct; the composite ranking
   uses both.
6. **W4 present-tense controls.** Every "Mitigate" cites an
   artefact that exists at ingestion time; roadmap items are marked
   "Planned".
7. **W4 cross-tags to SP 800-53 and PF Subcategories.** Every row
   carries at least one tag from each; where a control does not fit
   either catalogue, the row documents why.
8. **PF matrix is the output.** Subcategory rows cross-cite
   artefacts; N/A Subcategories are explicitly justified.
9. **AI RMF overlay present.** Each W4 row has at least one AI RMF
   Subcategory tag; AI 600-1 threat tags added where applicable.
10. **DPIA cross-reference.** The DPIA's §(c) register
    ([[exercise-01]]) or Step 5 ([[exercise-02]]) cites the PRAM
    output as its Appendix 3; row IDs reconcile between the two.

## 5. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| W1 framing has system identity, mission relevance, applicable-laws list (GDPR + applicable state laws + sectoral), organisational privacy principles, privacy risk statement, and stakeholders including non-user subjects | 2 | — |
| W2 lists ≥ 10 data actions; each row has data elements, context, nine-PDA tagging with one-sentence rationale, individual / organisation problem rows | 2 | — |
| W3 scores every (data action, PDA) pair on likelihood / individual impact / organisation impact; scores are traceable to measurements or baselines; the two impact columns are kept separate | 2 | — |
| W4 contains one response per W3 entry; present-tense cites; SP 800-53 control tags and NIST PF Subcategory tags per row; roadmap items flagged "Planned" | 2 | — |
| PF matrix (Subcategory × Artefact) is present with ≥ 15 Subcategory rows applicable, including N/A with justification | 1 | — |
| AI RMF overlay (and, where relevant, AI 600-1 threat tags) is present, cross-tagged per W4 row | 1 | — |

Automatic failures:

- W2 omits one of the C* (disclosure) data actions — common failure
  mode is to forget C3 (output-to-another-subject) and C5
  (cross-border).
- PDA tagging is uniform (all nine on every row) — signals clerical
  rather than analytical work.
- W3 scores assigned without evidence columns.
- W4 response column promises controls not implemented.
- PF matrix missing; or built from Subcategory names alone without
  binding to specific artefacts.
- PRAM diverges from the DPIA on measured parameters or risk
  ratings.

## 6. Common mistakes

- **Treating the PRAM as a renamed DPIA.** The PRAM's distinctive
  decomposition into data actions and its PDA taxonomy do analytical
  work the DPIA alone does not; collapsing them loses the value.
- **Omitting C* (disclosure) data actions.** C3 (output-to-another-
  subject) and C5 (cross-border) are the two most commonly forgotten.
  Both are present in any nontrivial AI system.
- **Marking every PDA on every data action.** Lazy. The nine-tag
  taxonomy is a cognitive tool; forcing a tag that does not apply
  dilutes the signal. Mark only what applies, with a one-line
  rationale.
- **Collapsing the two impact columns in W3.** The PRAM keeps
  individual and organisation impact separate for a reason: the two
  can diverge and both are real. A composite that averages them
  loses the hierarchy that drives different responses.
- **Scoring without evidence.** Likelihood "Medium" is not a score;
  likelihood "2 — Anthropic SOC 2 Type II + three comparable vendor
  incidents in 24 months" is. Opinions without data are a fail at
  audit.
- **Roadmap items dressed up as "Applied".** A control that is not
  in production fails audit sampling. Flag Planned honestly.
- **PF matrix produced from Subcategory names alone.** The matrix is
  Subcategory × Artefact; a row without a named artefact is vapourware.
- **N/A Subcategories hidden.** The auditor wants to see the full
  Subcategory list with N/A explicit. Hiding non-applicable
  Subcategories makes it look like the controller did not consider
  them.
- **Running the PRAM once.** It is a living document with the same
  refresh cadence as the DPIA. Run it on every major model-version
  change and quarterly at minimum.
- **AI RMF overlay missing.** NIST explicitly designed PF and AI
  RMF to compose. The overlay is cheap once the PRAM is in place
  and is what an AI RMF–speaking auditor expects.

## 7. References

- NIST — *Privacy Framework v1.0* (2020).
  <https://www.nist.gov/privacy-framework>.
- NIST — *Privacy Risk Assessment Methodology (PRAM)* resource page
  and workbook.
  <https://www.nist.gov/privacy-framework/resource-repository/browse/privacy-risk-assessment-methodology-pram>.
- NIST — NISTIR 8062, *An Introduction to Privacy Engineering and
  Risk Management in Federal Systems* (2017).
  <https://nvlpubs.nist.gov/nistpubs/ir/2017/NIST.IR.8062.pdf>.
- NIST — *AI Risk Management Framework 1.0* (2023).
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- NIST — *AI RMF Generative AI Profile*, NIST AI 600-1 (2024).
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- NIST — *SP 800-53 Rev. 5* (PT family specifically).
  <https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf>.
- NIST — *SP 800-53B, Control Baselines for Information Systems and
  Organizations*. <https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53B.pdf>.
- NIST — *SP 800-122, Guide to Protecting the Confidentiality of
  Personally Identifiable Information (PII)*.
  <https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-122.pdf>.
- MITRE ATLAS. <https://atlas.mitre.org/>.
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
