# Exercise 01 — GDPR Obligation → Engineering Artifact Map — Reference Solution

## 1. Solution overview

This is a **design exercise**. The learner is asked to take a set of concrete GDPR
obligations (not a vague "comply with GDPR") and map each one to the engineering
artifact(s) that a privacy engineer actually owns, co-owns, or reviews. The
assessable output is a table with four columns:

| GDPR cite | Obligation (plain English) | Engineering artifact(s) | Primary owner · Evidence a reviewer can inspect |

A passing map must (a) cite the article or recital correctly, (b) name an artifact
the engineer can point at (file, system, ticket queue, model card field — not a
policy PDF alone), and (c) state who signs off and what evidence proves the
control works. Rows that stop at "we have a policy" fail.

The reference below covers the obligations most often triggered by AI/ML systems
(training, retrieval, inference). It is not exhaustive — Articles 36, 40–43,
and 47 BCRs are intentionally out of scope because they are governance
instruments the engineer *consumes*, not *produces*.

## 2. Worked answer

### 2.1 Core mapping table

| GDPR cite | Obligation | Engineering artifact(s) | Owner · Evidence |
|---|---|---|---|
| Art. 5(1)(a) — lawfulness, fairness, transparency | Processing has a lawful basis and is transparent to the subject | Lawful-basis register per processing activity; in-product privacy notice; model card "intended use" + "training data sources" sections | Legal signs lawful basis; Engineering signs the notice copy is served before collection · Screenshot + served-notice log |
| Art. 5(1)(b) — purpose limitation | Data collected for one purpose is not silently repurposed | Data-catalog `purpose_tag` field; access-request diff check (CI gate that fails if a new consumer reads a dataset whose declared purpose doesn't include the new use) | Data platform · Catalog diff + CI run |
| Art. 5(1)(c) — data minimisation | Only data necessary for the purpose is processed | Feature schema review; column-level allowlist in the training pipeline; prompt redaction rules for inference-time logs | ML platform · Pipeline manifest showing columns and the review ticket that approved them |
| Art. 5(1)(d) — accuracy | Inaccurate data is rectified or erased | Rectification endpoint wired to the DSR service; downstream propagation job that re-materialises features/embeddings after rectification | DSR team · End-to-end test that rectifies a seeded record and asserts the embedding store updates within SLA |
| Art. 5(1)(e) — storage limitation | Data is kept no longer than necessary | Retention policy per dataset (`retention_days`); TTL / cold-archive job; training-copy expiry hook | Data platform · Deletion job logs + sample record TTL check |
| Art. 5(1)(f) — integrity and confidentiality | Appropriate security | Encryption at rest + in transit configs; IAM roles; secret rotation runbook | Security · IaC diffs + KMS audit logs |
| Art. 5(2) — accountability | Controller can *demonstrate* compliance | All of the artifacts in this table have an owner, a last-reviewed date, and an audit-log reference | Privacy program lead · The Art. 30 RoPA, which indexes the rest |
| Art. 6(1) — lawful basis | A specific lawful basis is identified per activity | Lawful-basis register entry per activity in the RoPA; code path that refuses to process if the basis flag is empty | Legal + Engineering · Unit test that rejects a processing call with `lawful_basis=None` |
| Art. 7 — conditions for consent (where Art. 6(1)(a) is used) | Consent is freely given, specific, informed, unambiguous; withdrawable as easily as it was given | Consent service: per-purpose record (`subject_id`, `purpose_id`, `granted_at`, `withdrawn_at`, `version`); downstream enforcement that filters training joins on consent state | Consent service team · Consent record + the training-pipeline join that filters on it |
| Art. 9 — special categories | Extra protection for sensitive data (health, biometrics, race, etc.) | Classification tag in the catalog (`sensitive=true`); separate KMS key; stricter access review cadence; prohibition-with-override in the feature registry | Data governance · Access reviews + catalog tag |
| Art. 12 — transparent information | Information to subjects in concise, intelligible form | Privacy notice served at point of collection, model card published, layered-notice UX in app | Product + Legal · URL + last-reviewed date |
| Art. 13 / 14 — information at collection / from third parties | Subjects are told who, why, legal basis, retention, rights | Notice template populated per collection point; third-party source register | Legal + Engineering · Notice served check in E2E test |
| Art. 15 — right of access | Subject can get a copy of their data | DSR access endpoint; data-subject export pipeline covering primary stores, feature store, vector store, and training-set membership flag | DSR team · Successful seeded-identity export + SLA report |
| Art. 16 — right to rectification | Subject can correct inaccurate data | Rectification endpoint; downstream recompute job (features, indexes, embeddings) | DSR team · Same as Art. 5(1)(d) evidence |
| Art. 17 — right to erasure ("right to be forgotten") | Subject data erased on request (subject to exceptions) | Deletion endpoint; cascading delete across stores; training-data removal ticket (and, if used, the unlearning / retraining decision) | DSR + ML platform · Erasure receipt with per-store confirmations; see mod-110 for unlearning depth |
| Art. 18 — restriction | Processing is halted without deletion | `processing_restricted` flag honoured by jobs; training joins exclude restricted subjects | DSR team · Flag + filter test |
| Art. 20 — portability | Machine-readable export of data provided by the subject | Export in structured, commonly used format (JSON/CSV) | DSR team · Export sample + schema |
| Art. 21 — objection | Subject can object to processing based on legitimate interests / direct marketing | Objection endpoint; propagation to the consent/legitimate-interest register | DSR team · Flag + filter test |
| Art. 22 — automated decision-making, including profiling | Right not to be subject to solely automated decisions producing legal/significant effects; safeguards (meaningful info, human intervention, contest) | ADM inventory entry per system; "meaningful information about the logic" section on the user-facing explanation; human-review queue + SLA for contest | Product + ML · Inventory entry linking the model card, the user explanation copy, and the human-review queue |
| Art. 25 — data protection by design and by default | Technical and organisational measures appropriate to risk, defaulted-on | Privacy review gate in the SDLC; default-deny access; pseudonymisation where feasible | Privacy engineering · Review ticket + the default config |
| Art. 28 — processors | Written contract (DPA) with each processor; processor acts only on instructions | Vendor register; signed DPA per vendor; sub-processor list in the product trust page | Procurement · DPA PDF + register |
| Art. 30 — records of processing activities | Controller (and processor) maintains a record of each activity | RoPA as a living artifact (not a one-off PDF); one row per activity with the fields in Art. 30(1); tied to the data catalog by `activity_id` | Privacy program · The RoPA file and the catalog join |
| Art. 32 — security of processing | Appropriate security given state of the art and risk | Threat model; encryption; access controls; backup + restore drill; log integrity | Security + Engineering · Threat model doc + incident-response runbook |
| Art. 33 — breach notification to authority (72 h) | Notify the lead supervisory authority without undue delay | Incident runbook with the 72-hour decision tree; paging rota; evidence-preservation checklist | Security + Legal · Latest tabletop exercise report |
| Art. 34 — breach notification to subject | Notify affected subjects when high risk | Notification template + delivery mechanism | Security + Comms · Template + delivery log |
| Art. 35 — DPIA | DPIA required for high-risk processing | DPIA template and completed DPIAs per high-risk activity (see mod-111) | Privacy engineering · Completed DPIA with reviewer sign-off |
| Art. 37 — DPO | Appoint a DPO where required | Published DPO contact; DPO on the privacy review distribution list | Legal · Published contact |
| Art. 44–49 — international transfers | Use an approved transfer mechanism (adequacy, SCCs, derogations) | Transfer register per data flow; SCC clause set in the DPA; region-pinning in the data-plane config | Legal + Infra · Transfer register row + region flag in IaC |

### 2.2 Rationale: why these artifacts and not others

Three design choices justify the mapping above.

1. **One obligation → one *inspectable* artifact.** The regulation repeatedly uses
   "appropriate" and "demonstrate." Reviewers cannot inspect "appropriate" — they
   inspect the configuration, the test, the log, the receipt. The right-hand
   column is always something a reviewer can open.
2. **Rights rows terminate at the DSR service.** Erasure, rectification,
   restriction, portability, objection, and access all funnel through the same
   subject-identifier resolution and store-fan-out. Do not duplicate per-right
   plumbing — duplicate per-right **endpoints and tests**, sharing the
   resolution step.
3. **Training and inference are separate consumers of the same lawful basis.**
   Lawful basis is not "the system has a basis"; it is "each distinct
   processing operation has a basis." Row Art. 6 therefore binds to the RoPA
   per-activity row, not to the product as a whole. This matters for AI
   systems that train on user data under a different basis (e.g., legitimate
   interests with opt-out) than they serve under (e.g., contract).

### 2.3 Worked implementation — example row, fully populated

- **GDPR cite:** Art. 17(1)(a)
- **Obligation:** Erase personal data when it is no longer necessary for the
  purpose for which it was collected.
- **Engineering artifact:**
  - `dsr-service` endpoint `POST /v1/subjects/{id}/erase`
  - cascading delete worker `erase-fanout` with store adapters for
    `postgres-users`, `s3-raw-events`, `feature-store`, `vector-index`,
    `training-corpus-manifest`
  - erasure receipt row in `dsr_receipts` table
- **Owner · Evidence:** DSR team owns the endpoint and the fanout; ML platform
  owns the training-corpus-manifest adapter. Evidence a reviewer inspects: a
  seeded-identity erasure that returns a receipt listing a successful call per
  adapter within the documented SLA, and a follow-up re-query of each store
  returning zero rows for that subject.

## 3. Validation steps

A learner's map can be self-checked without an instructor:

1. **Citation check.** Open each row's cited article in the official
   consolidated text at <https://eur-lex.europa.eu/eli/reg/2016/679/oj>. The
   obligation column must paraphrase the article, not restate the article
   number.
2. **"Who-owns-what" check.** Every row has exactly one primary owner. If a
   row has "everyone" or "privacy team," split it.
3. **Inspectability check.** Re-read the Evidence column with the question
   "could a new engineer open this on their first day?" If the answer is "they
   would need to ask someone," the artifact is not inspectable — replace it.
4. **AI-specific check.** The map must include at least one row that is
   specifically about model training data and one row that is specifically
   about automated decision-making. If both are absent, the map has been
   copied from a generic SaaS template.
5. **Negative test.** Pick a row and ask "what breaks if this artifact is
   missing?" If the answer is "nothing observable," the artifact is probably
   decorative.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| All thirteen "AI-triggering" articles present (5, 6, 7, 9, 13, 15, 17, 22, 25, 28, 30, 32, 35) | 3 | One point per missing cluster, floor zero |
| Every row cites a specific article/sub-article, not just "GDPR" | 1 | Deduct for naked "GDPR" references |
| Every row names an artifact a reviewer can open | 2 | "Policy" or "training" alone scores zero |
| Primary owner is a single named team/role per row | 1 | "Everyone" fails |
| Evidence column names a concrete log, test, receipt, or file | 2 | Not a policy document |
| Map distinguishes training-time from inference-time processing in at least one row | 1 | AI-specific signal |

Automatic failures:
- No row for Art. 30 (RoPA). The RoPA is the index; a map without it is incoherent.
- Any row that claims a "policy" is the artifact without a matching enforceable
  config/test/log.
- Any row that claims consent is the lawful basis for model training without
  also naming the consent-state join in the training pipeline.

## 5. Common mistakes

- **Using "we comply with GDPR" as the evidence.** Compliance is a conclusion;
  evidence is what justifies the conclusion. The artifact column must contain
  something that can be inspected.
- **Confusing lawful basis with consent.** Consent is one of six lawful bases
  under Art. 6(1); using "consent" everywhere (especially for employee data,
  internal analytics, or security logs) is usually wrong and often unlawful.
- **Treating "right to be forgotten" as a single endpoint.** Art. 17 erasure
  has to propagate to every store that received the data, including the
  feature store and the training-set manifest. A table that lists only the
  user-DB delete will fail audit.
- **Ignoring Art. 22 for ranking and scoring systems.** "It's just a ranker"
  is not a defence if the output has legal or similarly significant effects on
  the user.
- **Making the RoPA a PDF.** Art. 30 requires a *record*, not a document.
  Treat the RoPA as the join key between the catalog, the consent register,
  the DPIA index, and the DSR service.
- **Pushing international-transfer obligations onto Legal alone.** Art. 44–49
  bite the engineer through region-pinning, log-destination config, and
  sub-processor selection. The engineer owns the enforcement; Legal owns the
  mechanism.

## 6. References

- Regulation (EU) 2016/679 (General Data Protection Regulation), consolidated
  text. <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- European Data Protection Board (EDPB) Guidelines on Data Protection by
  Design and by Default (Guidelines 4/2019), final version 2.0, adopted
  2020-10-20. <https://edpb.europa.eu/>
- Article 29 Working Party / EDPB Guidelines on Automated individual
  decision-making and Profiling for the purposes of Regulation 2016/679
  (wp251rev.01), endorsed by the EDPB. <https://edpb.europa.eu/>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework> — used in this
  module as the operational vocabulary for *governance* roles that the GDPR
  text leaves implicit.
