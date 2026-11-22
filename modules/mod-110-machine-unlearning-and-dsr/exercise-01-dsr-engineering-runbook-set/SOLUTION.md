# Exercise 01 — DSR Engineering Runbook Set — Reference Solution

## 1. Solution overview

This is a **design + runbook exercise**. The learner is asked to
author the set of engineering runbooks a privacy-engineering function
needs to serve **data-subject rights (DSR) requests** against AI
systems: access, rectification, erasure, restriction, portability,
and objection / "opt-out of profiling." The deliverable is not an
essay — it is a set of operational documents an on-call engineer can
execute against a production AI stack without re-deriving the law or
the architecture every time.

The assessable artifacts:

1. **DSR type matrix.** Each supported DSR type (GDPR Articles 15–22
   and the CCPA/CPRA analogues) is mapped to:
   (a) the statutory deadline, (b) the AI-system surfaces that must
   be touched, (c) the engineering action per surface, and (d) the
   evidence captured for the audit trail (see [[exercise-07]]).
2. **Six runbooks.** One per DSR type — Access (Art. 15), Rectify
   (Art. 16), Erase (Art. 17), Restrict (Art. 18), Port (Art. 20),
   Object / opt-out of profiling (Art. 21–22). Each runbook has:
   intake, identity verification, system-by-system steps, verification,
   response template, and rollback / escalation paths.
3. **Surface inventory.** The list of AI-system surfaces the runbooks
   address: training datasets, model artifacts, fine-tuned adapters,
   embedding / vector stores, prompt + response logs, feature stores,
   derivative caches, RAG document stores, evaluation sets, and any
   downstream analytics copies.
4. **Technique selection hook.** For erasure specifically, a one-page
   pointer into the erasure-vs-retraining decision tree
   ([[exercise-06]]) so the on-call engineer picks SISA
   ([[exercise-02]]), approximate unlearning
   ([[exercise-03]], [[exercise-04]]), or full retraining under a
   consistent rule.

The runbook set is grounded in the GDPR (Regulation 2016/679)
Articles 12–22 for timelines and rights scope, ICO guidance on
automated decision-making, and NIST AI RMF 1.0 for the
Govern/Map/Measure/Manage framing of the audit trail. Threat framing
(who could subvert a DSR, how) is kept consistent with MITRE ATLAS
and OWASP ML Top 10.

## 2. Worked answer

### 2.1 DSR type matrix

| DSR right | GDPR article | Statutory clock | AI surfaces touched | Engineering action (summary) |
|---|---|---|---|---|
| Access | Art. 15 | 1 month (extendable +2 months with reason) | Training set index, logs, derived profiles, RAG store, feature store | Export every record keyed to the verified subject; redact other-subject PII; include processing-purpose, retention, and controller/processor list |
| Rectify | Art. 16 | 1 month | Training set (if still retained), feature store, RAG store, downstream caches | Update authoritative record; mark affected derived artifacts for refresh; decide retrain-vs-defer (see [[exercise-06]]) |
| Erase ("right to be forgotten") | Art. 17 | 1 month (no-fee unless manifestly unfounded) | **All of the above**, plus model artifacts and embeddings | Delete from authoritative stores; invalidate derivatives; apply unlearning or retraining ([[exercise-02]]–[[exercise-04]]) or justify refusal under Art. 17(3) |
| Restrict | Art. 18 | 1 month | Training set, serving-time inference, logs | Flag record as "do-not-process"; freeze derivatives rather than delete; block inclusion in future training |
| Port | Art. 20 | 1 month | Data provided by the subject (direct observations), not derived inferences | Export subject-provided records in a structured, machine-readable format (JSON/CSV); do **not** export model-derived features |
| Object / opt-out profiling | Art. 21–22 | Immediate (next processing cycle) | Serving-time inference, profile store, segmentation outputs | Switch subject to non-profiling path; disable automated decision-making; offer human review if Art. 22 applies |

Notes on the statutory clock: GDPR Art. 12(3) sets one month by
default, extendable by two months for complex requests, with the
subject notified of the extension within the first month. CCPA/CPRA
sets 45 days (extendable to 90) and splits "know" vs. "delete" vs.
"correct" (CCPA §§ 1798.100, 1798.105, 1798.106). The runbooks
assume the shorter of any applicable clocks per request.

### 2.2 Surface inventory

The runbooks assume the following nine surfaces exist and that the
privacy team has an index from subject identity to records in each.
The identity-index itself is maintained upstream of the AI system
(the surface the runbooks *call* rather than build).

| # | Surface | Deletion primitive | Unlearning-relevant? | Notes |
|---|---|---|---|---|
| 1 | Authoritative training record store (DB / data warehouse) | SQL delete / tombstone | Yes — forgets at source | Gold copy; downstream surfaces 2–9 rebuild from this |
| 2 | Raw training-data lake (object storage) | Object delete + shard-level reindex | Yes | Immutable snapshots complicate; see §2.4 |
| 3 | Feature / embedding store (precomputed) | Row delete + recompute key range | Yes | Vector stores index by ID; deletion must cascade |
| 4 | Model artifacts (checkpoints, adapters) | Retraining or unlearning algorithm | **The hard part** — see [[exercise-06]] | Deleting the weights file is not unlearning |
| 5 | Prompt + response logs (LLM) | Log tombstone / compaction | Partially — logs feed fine-tune jobs | Tied to retention windows (§2.4) |
| 6 | RAG / retrieval index (documents + vectors) | Document delete + vector removal | Yes (if indexed on PII) | Easier than model surgery |
| 7 | Evaluation / benchmark sets | Record delete + bench regeneration | Rare — but required if subject data leaked in | Keeps leak paths closed |
| 8 | Derivative / ML caches (e.g., Redis of pre-computed scores) | TTL + explicit flush | Yes | Easy to forget; include in runbook |
| 9 | Downstream analytics / BI copies | SQL delete + lineage check | Yes | GDPR Art. 19 notification obligations attach here |

### 2.3 Runbook template (all six runbooks follow the same shape)

The runbooks share one template so an on-call engineer never has to
learn a new format per right.

```
runbook-<right>.md
├── 0. Metadata (owner, SLA clock, legal citation)
├── 1. Intake
│     ├── Required fields in the request (subject identifier, right invoked, scope)
│     └── Deadline calculation (start-of-clock = date of verified intake)
├── 2. Identity verification
│     ├── Primary identity check (account holder, verified email, IdP token)
│     ├── Edge cases (ex-employee, deceased subject, minor, authorised agent)
│     └── Verification artefacts captured for audit (see [[exercise-07]])
├── 3. Scope resolution
│     ├── Enumerate the nine surfaces (§2.2); mark in/out of scope per request
│     └── Note Art. 17(3) exceptions, legal holds, retention obligations
├── 4. Execution steps (per surface)
│     ├── Query authoritative store
│     ├── Cascade to derivative surfaces
│     ├── Model-artifact decision (see [[exercise-06]])
│     └── Evidence captured per step (hash, timestamp, operator)
├── 5. Verification
│     ├── Automated check (post-action query returns 0 matches, or expected delta)
│     ├── Sampled manual check (for high-risk requests)
│     └── Sign-off (operator + reviewer)
├── 6. Subject response
│     ├── Response template (markdown; rendered to PDF for audit)
│     └── Timing: before SLA end; notify extension before day 30 if needed
├── 7. Audit trail write
│     └── See [[exercise-07]] for the required fields
├── 8. Rollback / escalation
│     ├── If step 4 partial: state and partial-fulfilment notice
│     ├── If identity verification fails: Art. 12(6) "reasonable doubt" response
│     └── If controller cannot comply (Art. 17(3)): written refusal with reasons
└── 9. Post-mortem trigger
      └── If SLA breached or miscategorised: post-mortem within 10 business days
```

### 2.4 The six runbooks — key differences

Each runbook is the template above with these right-specific
substitutions. A full runbook commits as one file per right in the
privacy-ops repo; the condensed form below is enough for the
exercise.

**Runbook A — Access (Art. 15).**
- Scope: surfaces 1, 3, 5, 6, 8, 9 (whatever contains personal data).
- Model artifacts (surface 4) are **not** in scope for Art. 15; a
  model weight is not "personal data about the subject" in any
  usable sense, and the Art. 15(4) carve-out for rights of others
  applies if weights could be reverse-engineered. Instead, respond
  with the *categories* of processing and the fact that the model
  was trained on data including the subject's, per Art. 15(1)(h).
- Output format: machine-readable JSON + human-readable PDF.
- Redaction: other subjects' PII removed by the same PII pipeline as
  [[mod-104]].

**Runbook B — Rectify (Art. 16).**
- Scope: authoritative store (1), then propagate via existing data
  pipelines; mark derivative surfaces for rebuild.
- Model-artifact decision: for most rectifications, defer — add the
  corrected record to the next training run rather than retrain.
  Document the deferral.

**Runbook C — Erase (Art. 17).**
- Scope: **all nine surfaces**.
- This is the only right where surface 4 (model artifacts) is
  routinely in scope, and where [[exercise-06]]'s decision tree runs.
- Refusal grounds (Art. 17(3)) must be cited explicitly, not
  presumed: freedom of expression, legal obligation to retain, public
  interest in public health, archiving in the public interest, or
  establishment / defence of legal claims. "Business continuity" is
  not on the list.

**Runbook D — Restrict (Art. 18).**
- Scope: surfaces 1, 3, 4, 5, 6, 8.
- Freeze, do not delete. The record stays but is tagged
  `processing_restricted: true` and excluded from new training runs,
  new feature computations, and serving-time inference.
- Downstream notification (Art. 19) still applies.

**Runbook E — Port (Art. 20).**
- Scope: **only** the data the subject actively provided (surface 1
  columns flagged `provided_by_subject: true`, plus surface 5
  prompts if the subject authored them).
- Derived inferences (surface 3 embeddings, surface 4 model outputs)
  are explicitly *out of scope* under Art. 20 and WP29 guidance
  (WP242).
- Output: open structured format (JSON, CSV, XML); include schema.

**Runbook F — Object / opt-out of profiling (Art. 21–22).**
- Scope: surfaces 4 (serving path), 3 (profile-store rows), 8
  (derivative caches).
- Operational action: route subject to the non-profiling serving
  variant and freeze any automated decision that has legal or
  similarly significant effect per Art. 22.
- Human-review option: wire the escalation path explicitly in step 8.

### 2.5 Statutory-clock worksheet

Every runbook must compute three dates at intake. Writing them down
once at the top of the ticket prevents the most common SLA miss.

| Field | Value | Source |
|---|---|---|
| `T_0` — verified intake | Date identity-verified | Runbook §2 |
| `T_30` — primary deadline | `T_0 + 1 month` (GDPR default) | Art. 12(3) |
| `T_90` — extended deadline | `T_0 + 3 months` (used only if extension notified ≤ `T_30`) | Art. 12(3) |
| `T_notify` — latest extension-notice date | `T_30 − 1 day` | Art. 12(3) |

If the subject is in a jurisdiction with CCPA/CPRA applicability,
`T_45` and `T_90` replace the GDPR fields; whichever clock is
*shorter* for the subject governs.

### 2.6 Governance mapping

Each runbook includes a "governance mapping" block so a reviewer can
see how the operational steps roll up to the AI RMF.

- **Govern.** DSR SLAs are published in the privacy-program charter;
  runbook owners are named; escalation paths route to a designated
  DPO or privacy counsel.
- **Map.** The surface inventory (§2.2) is kept current as a data-flow
  diagram; new surfaces must be added to this list before they go to
  production.
- **Measure.** Runbook execution is instrumented — ticket age, SLA
  compliance %, Art. 17(3) refusal rate, verification failure rate.
- **Manage.** Breached SLAs or refused requests generate a
  post-mortem. Patterns feed into the DPIA ([[mod-111]]).

MITRE ATLAS techniques the DSR surface touches:
`AML.T0024 Exfiltration via ML Inference API` (an Art. 15 flood can
be a reconnaissance vector) and
`AML.T0020 Data from Local System` (an unverified Art. 15 request can
be a social-engineering vector). OWASP ML Top 10 `ML01 Input
Manipulation` and `ML05 Membership Inference` are both why Art. 15
responses must be scoped narrowly.

## 3. Implementation / artifact shape

The deliverable committed to the privacy-ops repo:

```
dsr-runbooks/
├── README.md                 # the matrix (§2.1) and the surface inventory (§2.2)
├── template.md               # §2.3
├── runbook-access.md         # §2.4 A
├── runbook-rectify.md        # §2.4 B
├── runbook-erase.md          # §2.4 C (+ link to [[exercise-06]])
├── runbook-restrict.md       # §2.4 D
├── runbook-port.md           # §2.4 E
├── runbook-object.md         # §2.4 F
├── clock-worksheet.md        # §2.5
└── audit-schema.md           # link into [[exercise-07]]
```

Each runbook is a markdown file, versioned in git; changes go through
privacy review the same way a model-card change does. Operational
tooling (ticketing templates, automation scripts) sits in a parallel
tree and references the runbooks by stable section anchor.

## 4. Validation steps

1. **Deadline sanity.** Pick a sample intake date. Compute `T_0`,
   `T_30`, `T_notify`. The runbook's clock-worksheet must produce the
   same three dates by hand — no implicit "today" calls.
2. **Surface coverage check.** For each runbook, walk the nine
   surfaces in §2.2; every surface is either explicitly in-scope with
   a step or explicitly out-of-scope with a one-line reason. No
   surface is silently skipped.
3. **Erase cross-check.** The erase runbook references
   [[exercise-06]] (decision tree) explicitly. "Delete the weights
   file" without the decision tree is a fail.
4. **Portability scope test.** The port runbook refuses to export
   model-derived features. If a reviewer requests "my
   recommendations," the runbook responds with "derived data; see
   Art. 20 scope" and offers the Art. 15 access path instead.
5. **Identity-verification rigor.** Pick two edge cases from §2.3
   step 2 (authorised agent, deceased subject). The runbook must have
   a procedure for each; "ask legal" is acceptable only if the
   escalation path is named.
6. **Refusal cover.** The erase runbook enumerates Art. 17(3) grounds
   and does not use "business reasons" as a shortcut for any of them.
7. **Audit-trail link.** Every runbook's step 7 links into the audit
   schema from [[exercise-07]]. A runbook that captures evidence to
   an ad-hoc doc instead of the schema fails the audit property.
8. **Downstream notification.** Rectify, erase, and restrict runbooks
   address Art. 19 (notification of recipients) explicitly, including
   the "unless this proves impossible or involves disproportionate
   effort" exception, which must be *justified* in writing.

## 5. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| DSR matrix lists all six rights with correct GDPR citations and deadlines | 2 | Art. 15–22; clock = 1 month default |
| Surface inventory covers all nine surfaces; each is either in or explicitly out of scope per runbook | 2 | §2.2 |
| All six runbooks use the shared template (§2.3) | 1 | Operator shouldn't learn a new format per right |
| Erase runbook links to [[exercise-06]] and does not conflate "delete weights file" with unlearning | 1 | — |
| Portability runbook refuses to export model-derived features | 1 | Art. 20 scope; WP242 |
| Statutory-clock worksheet present and arithmetic-ready | 1 | §2.5 |
| Audit-trail step references [[exercise-07]] fields explicitly | 1 | — |
| Governance mapping (§2.6) present: NIST AI RMF function per runbook | 1 | Govern / Map / Measure / Manage |

Automatic failures:

- Treating "delete the row" as sufficient for erasure of an AI
  system (ignores surfaces 3–9).
- Listing Art. 17(3) exceptions as "business reasons."
- Allowing Art. 20 portability to export model outputs.
- Not computing a deadline at intake.
- Missing identity-verification procedures (DSR fulfilment to the
  wrong subject is itself a GDPR breach — Art. 32).

## 6. Common mistakes

- **Treating DSR as a legal-only process.** The runbooks are
  engineering artifacts. If engineers only receive a Jira ticket
  saying "GDPR Art. 17" with no surface list, they will miss
  surfaces 3–9. The matrix in §2.1 and inventory in §2.2 are the
  minimum interface.
- **Conflating "delete" with "forget."** Deleting a row from the
  authoritative store does not remove the subject from the model.
  The runbook must call out which surfaces are source-truth
  deletions and which require unlearning or retraining
  ([[exercise-06]]).
- **Ignoring immutable snapshots.** Training-data lakes often use
  append-only snapshots. The runbook must say how long a snapshot
  retains deleted records and either (a) rebuild the snapshot or
  (b) document the retention as a legal-basis-recorded exception.
- **Treating Art. 15 as "export everything."** Art. 15 is
  targeted — processing purposes, categories, recipients, retention,
  the data itself. Exporting an undigested database dump
  over-collects *other* subjects' data and is itself a breach.
- **Missing Art. 19 notifications.** Rectify, erase, and restrict
  trigger "notification to recipients." If the organisation
  processes data for third-party controllers, that notification is
  mandatory.
- **Over-fulfilling Art. 20.** Portability is for *subject-provided*
  data. Exporting embeddings or recommendation scores leaks the
  model's learned structure (a risk per OWASP ML05) and is outside
  the right's scope.
- **Not timing the extension notice.** Art. 12(3) requires the
  controller to *inform* the subject of an extension within one
  month — so the extension notice is itself on a clock. Missing that
  turns a compliant three-month response into a non-compliant one.
- **Running "verify" by reading the ticket again.** Verification must
  query the surfaces after the action. A ticket note that says
  "deleted" is not evidence; a database query that returns 0 rows
  for the subject key is.

## 7. References

- Regulation (EU) 2016/679 — General Data Protection Regulation, Arts.
  12–22.
  <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- ICO Guidance — Right of access.
  <https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/right-of-access/>
- ICO Guidance — Right to erasure.
  <https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/right-to-erasure/>
- Article 29 Working Party, *Guidelines on the right to data
  portability* (WP242 rev.01).
  <https://ec.europa.eu/newsroom/article29/items/611233/en>
- California Consumer Privacy Act / CPRA, §§ 1798.100, 1798.105,
  1798.106.
  <https://oag.ca.gov/privacy/ccpa>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
