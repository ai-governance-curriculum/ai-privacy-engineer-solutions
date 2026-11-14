# Exercise 05 — EU AI Act Article 10 (Data and Data Governance) — Data-Governance Slice — Reference Solution

## 1. Solution overview

This is a **design exercise**. Article 10 of Regulation (EU) 2024/1689 — the
EU AI Act — is the data-governance article that applies to **high-risk AI
systems**. It is not a free-standing data-protection regime; it sits *on
top* of the GDPR and the EU Data Act, and it translates "good data
governance" into obligations enforceable by market-surveillance authorities.

The learner's job is to take Article 10 paragraph-by-paragraph and produce
an engineering runbook: what each sub-paragraph requires, what the
privacy-engineering artifact is, who signs off, and where Article 10 diverges
from a generic GDPR data-governance picture.

The assessable output has three parts:

1. A paragraph-indexed **obligation → artifact** table covering Article 10(1)
   through 10(6).
2. A short note on the **Article 10(5)** carve-out for special-category
   personal data processing where strictly necessary to detect and correct
   bias, and its interaction with the GDPR.
3. A **worked implementation plan** for a single high-risk system — a
   credit-scoring model — showing where each Article 10 sub-paragraph is
   evidenced.

## 2. Worked answer

### 2.1 Where Article 10 sits in the Act

- The AI Act is Regulation (EU) 2024/1689, published in the Official
  Journal on 2024-07-12; the Regulation is in force and applies in phases.
  Article 10 applies to **high-risk AI systems** as defined in Article 6
  and listed in Annex III (and to certain Annex I-regulated products).
  The engineer should always check the operative version of the text at
  <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>.
- Article 10 is **not** about user-facing privacy rights. It is about the
  **training, validation, and testing data sets** used to build a high-
  risk system. It is an engineering-controls article aimed at model
  quality and fairness, with privacy constraints threaded through.
- Related articles the engineer must read alongside Article 10:
  - Art. 11 and Annex IV — technical documentation.
  - Art. 12 — logging.
  - Art. 13 — transparency to deployers.
  - Art. 14 — human oversight.
  - Art. 15 — accuracy, robustness, cybersecurity.
  - Art. 17 — quality management system.
  - Art. 72 — post-market monitoring.
  The Article 10 artifacts in this exercise feed each of these neighbours.

### 2.2 Paragraph-by-paragraph table

Paraphrases below are compressed; the authoritative text is the Official
Journal version. Each row lists the obligation, the engineering artifact,
and the primary owner.

| Article 10 ¶ | Obligation (paraphrase) | Engineering artifact | Primary owner · Evidence |
|---|---|---|---|
| 10(1) | Training, validation, and testing data sets shall be subject to data-governance and management practices appropriate for the intended purpose. | Written data-governance plan for the system, referenced from the technical documentation (Art. 11 / Annex IV). | ML platform + privacy engineering · The plan file, versioned, with named reviewers. |
| 10(2)(a) | Relevant design choices are documented. | Design-decision log with dated entries covering feature set, label definition, exclusions, sampling strategy, and the reason for each. | ML lead · Decision log + the model card's "design" section. |
| 10(2)(b) | Data collection processes and the origin of the data are documented. | Data-source manifest per dataset (`source_id`, `origin`, `legal_basis`, `collection_mechanism`, `collected_from`, `date_range`); for third-party sources a provenance note. | Data engineering · Manifest file + lineage edge in the catalog. |
| 10(2)(c) | Relevant data-preparation operations (labelling, cleaning, enrichment, aggregation) are documented. | Pre-processing pipeline source, including reproducible seed, label guidelines, and a QA sample. | ML platform · Pipeline file + QA report. |
| 10(2)(d) | Formulation of assumptions, in particular with respect to information the data is supposed to measure and represent. | "Measurement model" note: what each feature and label is a proxy for, and why. | ML lead · Measurement note + model card. |
| 10(2)(e) | Assessment of availability, quantity, and suitability of the data sets. | Dataset suitability report: size, coverage, representativeness vs intended population, missingness, timeliness. | ML platform · Report + stats dashboard. |
| 10(2)(f) | Examination in view of possible biases that are likely to affect health, safety, fundamental rights, or to lead to prohibited discrimination. | Bias examination report with subgroup metrics, chosen fairness definition, residual risk, and mitigation plan. | Privacy / fairness engineering · Report + metrics in CI. |
| 10(2)(fa) [as introduced in the final text] | Appropriate measures to detect, prevent, and mitigate biases identified. | Mitigation changelog: reweighting, rebalancing, feature exclusion, threshold shifts; linked to the bias-examination entries. | ML platform · Mitigation PRs + evaluation diff. |
| 10(2)(g) | Identification of relevant data gaps or shortcomings and how they can be addressed. | Gap register with issues, severity, plan, owner, due date. | ML lead · Register + closure log. |
| 10(3) | Training, validation, and testing data sets shall be relevant, sufficiently representative, and to the best extent possible free of errors and complete in view of the intended purpose. They shall have the appropriate statistical properties as regards the persons on which the system is intended to be used. | Representativeness checks (population vs training distribution); data-quality tests (duplicates, label noise, outliers). | ML platform · DQ job + representativeness metric. |
| 10(4) | Training, validation, and testing data sets shall take into account, to the extent required by the intended purpose, the characteristics or elements that are particular to the specific geographical, contextual, behavioural, or functional setting within which the system is intended to be used. | Context note describing the deployment setting and any evidence the dataset covers it (e.g., language, demography, seasonality, regulatory setting). | Product + ML · Context note + sign-off from deployer liaison. |
| 10(5) | To the extent strictly necessary for ensuring bias detection and correction in high-risk AI systems, providers may exceptionally process special categories of personal data referred to in Art. 9(1) GDPR, subject to safeguards including: use is strictly necessary; the special-category data is subject to technical limitations on re-use and state-of-the-art security; the data is pseudonymised where possible; the data is deleted once bias is corrected or the retention period ends. | Sensitive-attribute workbench — an isolated environment with its own KMS key, time-limited access, pseudonymisation, and automated deletion on exit; supporting GDPR Art. 6 and Art. 9 analysis in the DPIA. | Privacy engineering + legal · Access logs + automatic-deletion job + DPIA entry. |
| 10(6) | For high-risk AI systems that do not use techniques involving the training of models, paragraphs 2 to 5 apply only to the testing data sets. | Recorded determination of system type; if rule-based or similar, documented scope-reduction. | ML lead + legal · Scope statement in technical documentation. |

Note: Sub-paragraph letters may shift slightly between the OJ version and
common summaries; when citing in deliverables, always cite the OJ text. The
substance of the obligations above is stable.

### 2.3 Article 10(5) — special-category carve-out, in engineering terms

Article 10(5) **does not create a general permission** to collect sensitive
data for ML. It is a narrow, additional legal basis — on top of GDPR Art.
9(2) — for **the specific purpose of detecting and correcting bias in high-
risk AI systems**, and only when strictly necessary. The engineer must:

1. **Prove necessity.** Document why fairness assessment cannot be done
   without the special-category attribute. Proxy attributes, synthetic
   labels, and aggregate statistics must be considered first.
2. **Isolate processing.** Place the sensitive attribute in a dedicated
   workspace with its own encryption key, its own access list, and its
   own audit log. Do not merge the attribute into the general training
   store.
3. **Pseudonymise where possible.** Store the attribute keyed by a
   pseudonymous subject identifier; keep the join key in a re-identification
   enclave.
4. **Time-box.** Set a retention clock tied to the assessment, not to the
   dataset lifetime. When bias correction is complete, destroy the special-
   category data.
5. **Do not propagate.** Prevent the special-category attribute from
   leaving the fairness-assessment context; block it from inference-time
   features, logs, and model serialisation.
6. **Record in the DPIA.** The GDPR DPIA under Art. 35 must explicitly
   address the Art. 10(5) processing and the Art. 9 basis relied on.
7. **Document and be ready to demonstrate** the above to the relevant
   national competent authority on request.

### 2.4 Worked implementation — credit scoring (Annex III high-risk)

A bank subsidiary is building a credit-scoring model for retail consumer
loans. The system is high-risk under Annex III (creditworthiness
evaluation of natural persons). The deployer liaison is the retail-bank
product owner. Article 10 duties fall on the **provider** (whoever puts
the system into service under their name).

Implementation plan, keyed to Article 10:

- **10(1), 10(2)(a-c):** write `docs/data-governance/credit-score.md` with
  purpose, intended population, feature list, label definition, sampling
  strategy, and preprocessing pipeline paths. CI check that the file exists
  and is current.
- **10(2)(b) data origin:** manifest in `data/manifests/credit_core.yaml`
  with internal-source references and a signed data-share agreement for the
  bureau feed.
- **10(2)(c) preparation:** pipeline `pipelines/credit/preprocess.py`
  pinned to a release tag; QA report on `quality/credit/preprocess_qa.md`.
- **10(2)(d) assumptions:** "measurement note" explaining that "default"
  is defined as 90+ days past due within 24 months of origination; that
  the training set is drawn from the 2017–2023 cohort and the production
  cohort is 2026+; and the risks this gap creates.
- **10(2)(e) suitability:** `reports/credit/suitability.md` with coverage
  stats by region, product, and application channel.
- **10(2)(f) bias examination:** subgroup performance (approval rate,
  false-negative rate) across protected attributes available under Article
  10(5) or via a pseudonymised sensitive-attribute join. The chosen
  fairness definition and threshold are recorded with reasons.
- **10(2)(fa) mitigation:** "credit-score-fairness-mitigation.md" tracks
  each mitigation attempt, its measurable effect on subgroup metrics, and
  the final decision.
- **10(2)(g) gaps:** open issues in the gap register: e.g., limited data
  for one demographic subgroup in one region; plan to collect more; owner
  and due date.
- **10(3) representativeness and quality:** DQ test suite in CI;
  representativeness metric against the application population.
- **10(4) contextual fit:** deployment context note — markets, channels,
  product tiers covered; carve-outs for channels not covered.
- **10(5) special-category processing:** sensitive-attribute enclave with
  the safeguards in 2.3; documented in the DPIA and in the technical
  documentation.
- **Documentation chain:** all of the above referenced from Annex IV
  technical documentation and summarised in the model card.

Owners: ML platform lead (pipeline and reports), privacy engineering
(Article 10(5) enclave, bias-examination infra), data engineering
(manifests and lineage), legal (lawful basis and GDPR DPIA), product
owner (deployment context and intended purpose).

## 3. Validation steps

1. **Text check.** Open Article 10 at
   <https://eur-lex.europa.eu/eli/reg/2024/1689/oj> and read each
   paragraph the deliverable cites. The paraphrase must preserve the
   paragraph's binding verb ("shall").
2. **Scope check.** Verify the system is actually a high-risk AI system.
   Article 10 does not apply to non-high-risk systems. If the system is a
   general-purpose AI model, read Articles 51–55 instead.
3. **Provider vs deployer check.** Confirm the organisation is acting as a
   provider for Article 10 purposes. Deployers have Article 26 duties,
   which are different.
4. **Article 10(5) necessity check.** For any sensitive-attribute
   processing, re-ask whether proxies or aggregates would work. If yes,
   the Article 10(5) basis is not available.
5. **Documentation chain check.** Every Article 10 artifact should also be
   discoverable from the Annex IV technical documentation and from the
   model card.
6. **Interaction check.** Article 10 does not displace the GDPR. Confirm
   that lawful basis, data-subject rights, international transfers, and
   DPIA remain handled separately.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Paragraph table covers 10(1) through 10(6) with correct paragraph numbers | 2 | Omission or mis-citation per paragraph loses a point |
| Each row names an engineering artifact a reviewer can inspect | 2 | Not a policy |
| Article 10(5) treatment includes isolation, pseudonymisation, time-boxing, non-propagation, and DPIA entry | 2 | Missing any of these three essentials loses a point each |
| Worked implementation names the provider and verifies system is high-risk | 1 | Provider/deployer confusion fails |
| Worked implementation chains Article 10 artifacts into Annex IV technical documentation | 1 | Isolated Article 10 docs that never feed the dossier fail |
| Bias-examination step uses a named fairness definition and a measurable metric | 1 | "Looks fair" fails |
| Rationale distinguishes Article 10 from GDPR data-protection duties | 1 | Treating them as one loses the point |

Automatic failures:
- Treating Article 10 as a GDPR summary; it is a separate obligation set.
- Using Article 10(5) as a general permission to gather sensitive data.
- Omitting Article 10(6) (the carve-out for non-training systems).
- Citing Article 10 as applicable to a non-high-risk system.

## 5. Common mistakes

- **"Article 10 is the AI privacy article."** It is a **data-governance**
  article aimed at training, validation, and testing data. User-facing
  privacy rights come from the GDPR, not from Article 10.
- **Collapsing provider and deployer duties.** Article 10 is a provider
  duty. Deployers have Article 26 duties — especially around input data
  monitoring. Mixing the two produces duplicated or missing controls.
- **Using Article 10(5) as a free-floating consent to process
  special-category data.** It is narrow, strict-necessity, isolated,
  pseudonymised, time-boxed, and recorded.
- **Ignoring Article 10(6).** Rule-based or retrieval-only systems may be
  in scope as high-risk AI systems but with lighter Article 10
  obligations — specifically on testing data sets.
- **Writing a one-off dossier.** The Article 10 artifacts are living
  artifacts; the data-governance plan must be re-executed when the data,
  feature set, or deployment context changes.
- **Confusing Article 10 with Article 15 (accuracy, robustness,
  cybersecurity).** Accuracy and robustness have their own article; the
  Article 10 bias and representativeness duties are a separate set that
  feeds into (but is not the same as) Article 15.
- **Ignoring the Data Act and GDPR interactions.** Article 10 presupposes
  that the underlying processing has a GDPR legal basis. Article 10(5)
  adds a narrow Art. 9 permission; it does not remove the Art. 6
  requirement.

## 6. References

- Regulation (EU) 2024/1689 — Artificial Intelligence Act, OJ L 1689,
  2024-07-12. <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- European Commission — AI Act overview.
  <https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai>
- Regulation (EU) 2016/679 — General Data Protection Regulation.
  <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST AI RMF Generative AI Profile (NIST AI 600-1).
  <https://www.nist.gov/itl/ai-risk-management-framework>
