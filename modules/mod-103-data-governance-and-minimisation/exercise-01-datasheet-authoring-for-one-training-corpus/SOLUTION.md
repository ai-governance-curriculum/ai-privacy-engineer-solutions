# Exercise 01 — Datasheet Authoring for One Training Corpus — Reference Solution

## 1. Solution overview

This is a **design + documentation exercise**. The learner picks one training
corpus their team actually uses (or a well-scoped public corpus if they do
not) and produces a **datasheet** following the Gebru et al. "Datasheets for
Datasets" schema, extended with the fields a privacy engineer needs for
GDPR Art. 30 records of processing activities, EU AI Act Article 10
documentation, and NIST AI RMF `MAP` / `MEASURE` functions.

The assessable output has three parts:

1. A completed **datasheet** answering every section of the Gebru schema
   (Motivation, Composition, Collection, Preprocessing, Uses, Distribution,
   Maintenance).
2. A **privacy-engineering appendix** that records lawful basis, sensitive-
   category tagging, retention, access, DSR hooks, and lineage keys.
3. A short **decision rationale** explaining what was deliberately *not*
   included in the corpus and why — minimisation evidence rather than a
   finished product.

A datasheet that answers every question but leaves lawful basis, retention,
and DSR hooks implicit fails this exercise. So does a datasheet that
copy-pastes the Gebru questions and answers "N/A" to more than a handful.

## 2. Worked answer

### 2.1 Scenario used in the worked example

Fictional, tightly scoped so the learner can compare. A European retail
SaaS is training a support-chatbot assistant. The training corpus,
internally named `support-dialogues-eu-v3`, is a filtered slice of
historical customer-support tickets written by signed-in users in the
EU/UK over 2021-09-01 to 2024-08-31. Each record is a `(ticket_id,
turn_index, author_role, text, resolved_label, product_area)` tuple. No
media, no attachments. Separate consent-state and pseudonymisation
pipelines have already run — the corpus stores pseudonymous `account_pid`
instead of `account_id`, and only includes tickets whose authors granted
the "improve assistant quality" training purpose at the time the ticket
was authored.

Everything in the datasheet below is scenario-specific. The learner's
answer will differ in content; the *shape* is what the rubric scores.

### 2.2 Datasheet implementation — core sections (Gebru et al. schema)

**Motivation**

- *For what purpose was the dataset created?* To fine-tune a retrieval-
  augmented chatbot that assists human agents on English-language EU
  retail-support queues. The corpus is specifically for *style* and
  *resolution-pattern* learning, not for factual recall; retrieval is
  handled by a separate, unindexed knowledge base.
- *Who created the dataset and on behalf of which entity?* The ML
  platform team at `Acme Retail SaaS B.V.`, Amsterdam, on behalf of the
  Assistant product group.
- *Who funded the creation of the dataset?* Internal product budget.
  Line reference: `assistant-fy24-training`.

**Composition**

- *What do the instances represent?* Turn-level dialogue rows from
  resolved historical support tickets authored by signed-in EU/UK
  consumer users.
- *How many instances?* 1,284,117 turns across 192,044 tickets as of the
  2024-09-01 freeze.
- *Does the dataset contain all possible instances or a sample?* A
  sample. Filters: resolution status ∈ {resolved, closed-ok};
  language ∈ {en-GB, en-IE}; consent state at authoring time =
  granted-for-training; time window ≥ 2021-09-01.
- *What data does each instance consist of?* `ticket_id`, `turn_index`,
  `author_role ∈ {customer, agent}`, `text` (UTF-8, ≤ 4 KB per turn),
  `resolved_label` (coarse categorical), `product_area` (categorical),
  `account_pid` (pseudonymous).
- *Is there a label or target?* `resolved_label` and `product_area` are
  the categorical labels used for weak supervision.
- *Is any information missing?* By design: real `account_id`, email,
  real name, address, order IDs, payment data. These are either not
  present in the raw source or are removed in the redaction step; see
  Preprocessing.
- *Are relationships between individual instances made explicit?* Yes,
  via `ticket_id`. Turns within one ticket are the dialog thread.
- *Are there recommended data splits?* Random 80/10/10 by `ticket_id`
  (not by turn) to keep dialog threads intact; the splits are fixed at
  corpus-freeze time for reproducibility.
- *Are there errors, sources of noise, or redundancies?* Yes. Known
  issues are tracked in `datasets/support-dialogues-eu-v3/known-issues.md`:
  (a) customer turns sometimes include quoted automated emails from
  external merchants; (b) agent macros appear in roughly 4% of turns;
  (c) a known burst of test tickets during 2023-03-14 to 2023-03-16 is
  filtered out at ingestion.
- *Does the dataset contain personal data?* Yes, by construction.
  Pseudonymised. See 2.3 for the privacy-engineering appendix.
- *Does it contain data that might be considered sensitive?* Potentially,
  in free text. The preprocessing step includes a named-entity and
  secret-scrub pass (see Preprocessing); residual leakage is monitored
  by the sampled audit loop described in 2.3. No field is intentionally
  a special-category attribute under GDPR Art. 9.
- *Does it contain data about children?* No. The product requires age
  attestation at account creation; the training filter joins on
  `is_adult_at_auth = true`.

**Collection process**

- *How was the data acquired?* Collected by the support product itself
  during normal operation. The corpus is a snapshot of the warehouse
  tables `support.ticket_v2` and `support.ticket_turn_v2`.
- *Who was involved in data collection?* Support agents authored agent
  turns. Customers authored customer turns. Platform engineering owns
  the warehouse tables.
- *Over what timeframe was the data collected?* 2021-09-01 to 2024-08-31.
- *Were any ethical review processes conducted?* Yes. A DPIA covering
  the training purpose was completed 2024-06-10 and approved by the DPO
  2024-06-20. The DPIA entry is at `dpias/assistant-training-v1`.
- *Does the dataset relate to people?* Yes; see Composition.
- *Did the individuals in question consent to the collection and use of
  their data?* Collection of the underlying tickets rests on contract
  necessity (Art. 6(1)(b)) for support. Use for training rests on
  **consent** for the "improve assistant quality" purpose, recorded at
  the time of ticket authoring. The corpus join filters on that consent
  state, including withdrawals.
- *Were the individuals notified about the data collection?* Yes, via
  the layered privacy notice at authoring time and the product notice
  at signup.
- *Was the data collected from the individuals in question directly, or
  obtained via third parties?* Directly, through the product.

**Preprocessing / cleaning / labeling**

- *Was any preprocessing / cleaning / labeling done?* Yes.
  1. Pseudonymisation: replace `account_id` with `account_pid` via the
     consent-respecting mapping service.
  2. Redaction pipeline: regex + model-based scrub for emails, phone
     numbers, IBANs, card numbers, API keys, URLs with embedded
     credentials. Each hit is replaced with a typed placeholder
     (`<EMAIL>`, `<PHONE>`, etc.).
  3. Secret scan: `trufflehog` / `gitleaks`-style high-entropy scan on
     each turn; matches route to redaction + an alert on the data-team
     channel.
  4. Language detection: keep `en-GB`/`en-IE`; drop others.
  5. Label backfill: `resolved_label` and `product_area` are copied
     from the ticket row.
  6. De-duplication: exact hash dedup on `text`; cross-ticket near-
     duplicate detection with MinHash at 0.9 Jaccard.
- *Was the raw data saved in addition?* Yes. The source tables remain
  in the support warehouse under their own retention schedule, separate
  from this corpus. The training corpus itself has its own retention
  clock (see Maintenance).
- *Is the software that was used to preprocess available?* Yes:
  `pipelines/support-dialogues-eu-v3/preprocess.py` pinned to tag
  `v3.0.0`, Dockerfile `images/preprocess:v3.0.0`.

**Uses**

- *Has the dataset been used for any tasks already?* Yes, to train the
  assistant style-and-resolution fine-tune (`assistant-ft-v3.*`).
- *Is there a repository that links to any or all of these?* Internal
  only: `models/assistant-ft-v3/MODEL_CARD.md`.
- *What (other) tasks could the dataset be used for?* Agent-side style
  transfer, intent-classification bootstrapping, prompt-template
  evaluation. Any new task requires a new entry in the RoPA and a new
  privacy review.
- *Is there anything about the composition of the dataset or how it was
  collected that might impact future uses?* Yes: (a) only EU/UK adults,
  so models trained on it generalise poorly outside that population and
  should not be deployed to US or APAC queues without re-evaluation;
  (b) consent-filtered, so unlearning follows a bulk-filter rebuild
  rather than per-record deletion from model weights; (c) pseudonymised,
  so re-identification requires the mapping service and is blocked at
  the data boundary.
- *Are there tasks for which the dataset should not be used?* Any
  downstream decision that materially affects a user (credit, pricing,
  warranty, fraud flags). The corpus is for style and resolution-pattern
  learning only; using it as a signal for user-facing decisioning would
  be a purpose change and would need a fresh lawful-basis assessment and
  DPIA.

**Distribution**

- *Will the dataset be distributed to third parties outside of the
  entity on behalf of which the dataset was created?* No. The corpus is
  internal only. If this changes, the distribution would require a new
  processor contract under GDPR Art. 28 and a transfer-mechanism
  decision under Art. 44–49.
- *How will the dataset be distributed?* Not applicable; internal object
  storage only. Access through the ML platform's catalog-aware readers.
- *When will the dataset be distributed?* Not applicable.
- *Will the dataset be distributed under a copyright or other IP
  licence?* Not applicable; no external distribution.

**Maintenance**

- *Who is supporting / hosting / maintaining the dataset?* ML platform
  team, team alias `ml-platform-eu`.
- *How can the owner be contacted?* Via `ml-platform-eu@acme.example`,
  linked on the catalog entry.
- *Is there an erratum?* Yes: `datasets/support-dialogues-eu-v3/known-issues.md`.
- *Will the dataset be updated?* Snapshot is immutable. New releases
  (`v4`, …) are produced on a quarterly cadence; each release is a new
  catalog entry.
- *If the dataset relates to people, are there applicable limits on the
  retention of the data?* Yes. The corpus has a `retention_days = 730`
  TTL, with an earlier-destruction trigger if the training model it
  supports is retired or if consent withdrawals cross a 5% threshold
  that requires a rebuild.
- *Will older versions of the dataset continue to be supported?* The
  current + previous frozen version are kept for reproducibility;
  earlier versions are destroyed on their TTL.
- *If others want to extend / contribute to the dataset, is there a
  mechanism for them to do so?* Internal contribution via a PR against
  the preprocessing pipeline plus a datasheet revision and DPIA review.

### 2.3 Privacy-engineering appendix

This appendix is **not** optional for an AI/ML training corpus. It is what
turns a Gebru datasheet into an artifact a privacy reviewer and an auditor
can act on.

- **Lawful-basis register entry:** `ropa/assistant-training/consent` —
  Art. 6(1)(a) consent for the purpose "improve assistant quality,"
  bound to the corpus by `activity_id = assistant-training-eu`.
- **Special-category tagging:** `sensitive = false` at the field level.
  Residual risk of Art. 9 data in free text is monitored by a sampled
  review (see "Known residual-risk monitor").
- **Pseudonymisation boundary:** Mapping `account_id ↔ account_pid`
  lives in the pseudonymisation service; access to the mapping requires
  a privacy-engineering approved role and logs every lookup. The
  training environment does not have this role.
- **Retention clock:** `retention_days = 730`, enforced by the storage
  TTL job `training-corpus-expiry`; a monthly job verifies the TTL is
  applied.
- **Access:** training-corpus readers restricted to the service
  accounts that back the fine-tune jobs plus named ML-platform
  engineers on the on-call roster. Access reviewed quarterly.
- **DSR hooks:**
  - *Access* (Art. 15): the corpus is indexable by `account_pid`; the
    DSR service resolves `account_id → account_pid` in the enclave and
    returns the subject's turns.
  - *Erasure / restriction* (Art. 17, 18): a deletion in the corpus is a
    filter-on-rebuild; the next scheduled rebuild drops the subject's
    rows, and the current snapshot is reindexed with the subject
    excluded. For the already-trained model, follow mod-110's unlearning
    decision path; the default is wait-for-next-retrain.
  - *Rectification* (Art. 16): the corpus is append-only snapshots; the
    user's current content is rectified in the source, and the next
    snapshot picks it up.
- **Consent withdrawal:** daily job `consent-delta` re-runs the corpus
  filter against the current consent register; when drift crosses 2% of
  rows, the corpus is scheduled for rebuild; crossing 5% triggers an
  automatic rebuild.
- **Known residual-risk monitor:** a 0.5% random-sample review every
  week; reviewer rotates among privacy-engineering oncall. Hits trigger
  a PR against the redaction pipeline and a re-run.
- **Lineage keys:** every downstream model card references
  `dataset_id = support-dialogues-eu-v3`, `version = v3.0.0`,
  `freeze_date = 2024-09-01`, and the preprocess image digest. The
  ML-BOM slice from exercise-06 is produced from these lineage keys.
- **Transfer mechanism:** none; EU-pinned storage and EU-pinned
  training clusters. Deployment outside the EU would re-trigger
  Art. 44–49.
- **Related artifacts:**
  - DPIA at `dpias/assistant-training-v1`
  - RoPA row `assistant-training-eu`
  - Model card `models/assistant-ft-v3/MODEL_CARD.md`
  - ML-BOM slice `sboms/assistant-ft-v3-mlbom.json`

### 2.4 Decision rationale — what was deliberately excluded

Minimisation is best evidenced by what you refused to collect. For this
corpus the exclusions are:

1. **Non-signed-in traffic.** Anonymous visitor interactions are not
   joined in, because the training purpose requires consent that can be
   attributed to a subject. Including them would weaken the lawful-
   basis claim and the DSR hook.
2. **Attachments (PDFs, images).** Removed because the training target
   is turn-level style, not multimodal reasoning. Attachments routinely
   contain IDs and financial data; the cost/benefit is negative.
3. **Agent demographic fields.** Agents are employees. Adding agent
   metadata would add workforce-data processing under a different
   lawful basis (legitimate interests with a works-council
   consultation) and would not improve model quality on the target
   task.
4. **Session transcripts beyond 24 months old.** Older transcripts
   predate the current product and the current notice copy; including
   them would re-raise the lawful-basis question. 24 months is set as
   the cold-cutoff deliberately.
5. **Any retrieval corpus.** RAG is served from a separate,
   non-indexed knowledge base; mixing the two would blur the "style"
   vs "facts" boundary and would change the memorisation risk profile.

Each exclusion is a row in `datasets/support-dialogues-eu-v3/exclusions.md`
with the reason and the reviewer who signed it. Reviewers can inspect the
file to verify minimisation was a decision, not an accident.

## 3. Validation steps

1. **Schema completeness.** Every Gebru section is answered with a
   dataset-specific statement, not "N/A" or a boilerplate paragraph.
   More than three "N/A" answers in Composition or Collection is a red
   flag.
2. **Lawful-basis check.** The lawful basis is named with the specific
   GDPR Art. 6 (and Art. 9 if sensitive) sub-paragraph *and* the
   enforcement point in the pipeline is referenced (join, filter, or
   gate). "Consent" alone is not enough; name the record and the join.
3. **DSR hook check.** For each GDPR data-subject right likely to
   trigger on this corpus (access, erasure, restriction, rectification),
   the appendix names the mechanism, not a policy page.
4. **Retention check.** A retention clock exists, has a numeric value,
   and is tied to an enforceable job (not a policy statement).
5. **Minimisation-by-exclusion check.** At least three excluded fields /
   populations are listed with reasons. If every candidate field made it
   into the corpus, minimisation is implicit at best.
6. **Lineage check.** The datasheet carries the identifiers
   (`dataset_id`, `version`, `freeze_date`, preprocess image digest)
   that the model card and the ML-BOM slice will reference.
7. **Known-issues honesty check.** The Composition and Preprocessing
   sections admit real limitations (duplicates, macros, test bursts,
   residual leakage audits). A datasheet without any known issues is
   either the first draft or dishonest.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Every Gebru section answered with scenario-specific detail | 2 | More than three "N/A" answers in Composition / Collection fails this item |
| Lawful basis cites a specific GDPR article and a specific enforcement point | 1 | "Consent" alone fails |
| Privacy-engineering appendix present with lawful basis, retention, access, DSR hooks, lineage | 2 | Missing any one essential (basis, retention, DSR) loses the whole point |
| Decision rationale lists ≥ 3 deliberate exclusions with reasons | 1 | Exclusions without reasons fail |
| Lineage identifiers tie to the model card and ML-BOM slice | 1 | Dataset version and preprocess digest required |
| Known-issues / errata section is non-trivial | 1 | At least two concrete issues |
| EU AI Act / NIST AI RMF cross-reference present when applicable | 1 | For high-risk systems only; otherwise ignored |
| Child-data, special-category, and sensitive-attribute status is explicitly stated | 1 | Silence on these three fails |

Automatic failures:

- Omitting lawful basis or citing "GDPR" without an article.
- Omitting retention.
- Omitting any DSR hook when the corpus contains personal data.
- Claiming anonymisation when the corpus is in fact pseudonymised. A
  dataset that can be rejoined via a mapping service is pseudonymous,
  not anonymous.

## 5. Common mistakes

- **Treating pseudonymisation as anonymisation.** Pseudonymous data is
  still personal data under GDPR; the mapping service is the join
  point, and calling it "anonymous" is a legal error with engineering
  consequences (retention, DSR, basis all remain in scope).
- **Datasheet as a one-time PDF.** The datasheet is a living artifact
  tied to the dataset version. A new freeze is a new datasheet row (or
  a new section) and a new catalog entry.
- **"N/A" as the main answer.** A datasheet full of "N/A" means the
  author answered the questionnaire, not described the dataset. Each
  "N/A" needs a reason.
- **Mixing training and retrieval.** If the corpus is the training set,
  do not describe it as if it were also the retrieval corpus. The two
  have different memorisation risks and different purposes.
- **Burying residual-risk monitoring.** A random-sample review of
  residual leakage should be a named job with an owner, not a
  commitment in prose.
- **Forgetting the model-card hand-off.** The datasheet feeds the
  model card. If the model card cannot be regenerated from the
  datasheet's lineage identifiers, the datasheet is incomplete.
- **Claiming consent for employee data.** Agent turns are workforce
  data; consent is rarely a valid basis for workforce processing.
  Either justify legitimate interest with the usual LIA test, or
  exclude agent turns.

## 6. References

- Gebru, T., Morgenstern, J., Vecchione, B., Vaughan, J. W., Wallach,
  H., Daumé III, H., and Crawford, K. "Datasheets for Datasets."
  Communications of the ACM, 64(12), 2021. <https://dl.acm.org/doi/10.1145/3458723>
- Regulation (EU) 2016/679 (General Data Protection Regulation),
  consolidated text, in particular Arts. 5, 6, 7, 9, 13–18, 25, 30,
  32, 35, 44–49. <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- Regulation (EU) 2024/1689 — Artificial Intelligence Act, in
  particular Article 10 on data governance. <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- NIST AI Risk Management Framework (AI RMF 1.0), Govern/Map/Measure
  functions. <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST AI 600-1 — Generative AI Profile, data-provenance and
  content-provenance subcategories. <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
- OWASP Machine Learning Security Top 10. <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — the ATLAS matrix covers data-staging adversary
  techniques that a datasheet should force the team to acknowledge.
  <https://atlas.mitre.org/>
