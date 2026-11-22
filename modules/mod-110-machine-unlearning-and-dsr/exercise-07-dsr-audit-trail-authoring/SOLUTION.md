# Exercise 07 — DSR Audit Trail Authoring — Reference Solution

## 1. Solution overview

This is a **design + schema exercise**. The learner authors the
**audit trail** that backs every data-subject-rights (DSR) action
the organisation takes against its AI systems: the schema, the
capture points, the retention and integrity controls, and the
report formats that a regulator, an internal auditor, or counsel
can read without touching production. The audit trail is what
converts a runbook run ([[exercise-01]]) and a method choice
([[exercise-06]]) into *evidence that would survive discovery*.

The assessable artifacts:

1. **A canonical audit-entry schema** (JSON + a human-readable
   rendering), with required fields, optional fields, enum values,
   and a cryptographic-integrity envelope.
2. **Capture-point map.** For each of the nine AI-system surfaces
   from [[exercise-01]] §2.2, the schema fields populated at each
   step of the runbook; where the data comes from; who is
   accountable.
3. **Retention, integrity, and access policy.** How long entries
   are kept; how they are made tamper-evident; who can read them;
   how they are produced on legal request.
4. **Example audit entry** for a representative Art. 17 erasure
   that used Path C (SISA, [[exercise-02]]) and a separate entry
   for Path D (NPO, [[exercise-04]]), so the schema's optional
   fields are exercised against real method choices.
5. **A validation checklist** for a reviewer to run against any
   candidate entry before it is committed.

The schema aligns with GDPR Art. 30 (records of processing
activities) and the Art. 12/15/17/19 obligations the runbook
operationalises, and plugs into NIST AI RMF's Govern/Map/Measure/
Manage loop so the audit record is the Measure artefact for DSR.

## 2. Worked answer

### 2.1 Schema (canonical form)

Audit entries are JSON documents validated by a pinned schema
version. Required fields are marked `(req)`; optional fields are
present when the path applies. The example shown here omits
cryptographic noise for readability; §2.4 adds the envelope.

```jsonc
{
  "schema_version": "dsr-audit/1.0",                    // (req)
  "entry_id": "ae_01HJM7NKGZ3W1A9MY4R2TQV2B8",          // (req) ULID
  "created_at": "2026-11-21T14:03:21Z",                 // (req) RFC3339 UTC
  "created_by": "op_jferguson@example.com",             // (req) operator identity
  "reviewed_by": "op_privacy-lead@example.com",         // (req) reviewer; different from operator
  "controller": "Example AI Ltd.",                      // (req) legal controller
  "processors": ["cloud-provider-x", "analytics-y"],    // (opt) Art. 28 processors touched

  // ----- request block -----
  "request": {
    "request_id": "dsr_01HJM6YJ8X7W0ZH5TK6P3QFCR4",     // (req) ticket id
    "right": "erase",                                   // (req) access|rectify|erase|restrict|port|object
    "law_citations": ["GDPR Art. 17(1)(a)"],            // (req) one or more
    "subject_pseudonymous_id": "psu_ab12…ef90",         // (req) one-way hash of verified subject id
    "intake_at": "2026-11-05T09:15:00Z",                // (req) T_0 from [[exercise-01]] §2.5
    "deadline_at": "2026-12-05T23:59:59Z",              // (req) T_30 (or T_90 if extension)
    "extension_notified_at": null,                      // (opt) when extension used
    "identity_verification": {                          // (req) see [[exercise-01]] §2.3 step 2
      "method": "idp-oidc",
      "evidence_ref": "s3://privacy-vault/iv/dsr_01HJM6…/claims.json.enc"
    }
  },

  // ----- scope block -----
  "scope": {                                            // (req) which surfaces in-scope
    "surfaces": [
      { "surface": "authoritative-store",  "in_scope": true  },
      { "surface": "training-lake",        "in_scope": true  },
      { "surface": "feature-store",        "in_scope": true  },
      { "surface": "model-artifacts",      "in_scope": true, "method_path": "C-sisa" },
      { "surface": "prompt-logs",          "in_scope": true  },
      { "surface": "rag-index",            "in_scope": false, "reason": "not indexed on subject PII" },
      { "surface": "eval-sets",            "in_scope": false, "reason": "no subject records sampled" },
      { "surface": "derivative-caches",    "in_scope": true  },
      { "surface": "downstream-analytics", "in_scope": true  }
    ]
  },

  // ----- actions block: one entry per surface touched -----
  "actions": [
    {
      "surface": "authoritative-store",
      "action": "delete",
      "executed_at": "2026-11-21T13:40:02Z",
      "pre_state_hash":  "sha256:49f0…b31a",           // row-set hash before
      "post_state_hash": "sha256:9d8c…02b1",
      "evidence_ref": "s3://privacy-vault/dsr_01HJM6…/01-auth.log.enc",
      "verification": { "method": "query", "result": "0 rows for subject_key", "at": "2026-11-21T13:40:41Z" }
    },
    {
      "surface": "model-artifacts",
      "action": "unlearn",
      "method": "SISA",                                  // (req when action=unlearn or retrain)
      "method_params": { "shards": 10, "slices": 10, "shard_hit": 3, "slice_removed_at": 2 },
      "pre_artifact_hash":  "sha256:1b7e…ff20",          // all S shards
      "post_artifact_hash": "sha256:1b7e…c0a4",          // one shard changes; others identical
      "bit_identity_check": { "ran": true, "passed": true, "retrain_baseline_hash": "sha256:1b7e…c0a4" },
      "executed_at": "2026-11-21T13:52:11Z",
      "evidence_ref": "s3://privacy-vault/dsr_01HJM6…/04-model.jsonl.enc"
    },
    { "surface": "prompt-logs", "action": "tombstone", "executed_at": "2026-11-21T13:55:00Z", "evidence_ref": "…" },
    { "surface": "derivative-caches",  "action": "flush",    "executed_at": "2026-11-21T13:57:22Z", "evidence_ref": "…" },
    { "surface": "downstream-analytics", "action": "delete-with-notice", "art19_notified": true, "executed_at": "2026-11-21T14:01:00Z", "evidence_ref": "…" }
  ],

  // ----- outcome -----
  "outcome": {                                           // (req)
    "status": "fulfilled",                               // fulfilled|refused|partially-fulfilled
    "refusal_grounds": null,                             // Art. 17(3) citation when refused
    "partial_reasons": null,                             // required when partially-fulfilled
    "subject_response_sent_at": "2026-11-21T14:02:55Z",
    "subject_response_ref": "s3://privacy-vault/dsr_01HJM6…/response.pdf.enc"
  },

  // ----- governance -----
  "governance": {                                        // (req) NIST AI RMF mapping
    "function": "Measure",
    "manage_postmortem_due": null                         // set only when SLA breached
  },

  // ----- integrity envelope (populated by signer) -----
  "integrity": {                                         // (req) see §2.4
    "signer_key_id": "kms://privacy-keys/signer-2026-q4",
    "signature":    "base64…",
    "hash_chain":   { "prev_entry_id": "ae_01HJM7NG…", "merkle_leaf": "sha256:…" }
  }
}
```

### 2.2 Field notes (by block)

- **`created_by` and `reviewed_by` must differ.** Two-person
  integrity is a GDPR Art. 32 control; a single operator writing
  and reviewing is a fail.
- **`subject_pseudonymous_id`** is a one-way hash of the verified
  subject identifier *plus* a controller-held salt, so the audit
  log never contains the direct subject identifier. The map from
  pseudonym to identifier lives in a separate, access-controlled
  vault; producing it is itself a logged event.
- **`law_citations`** is a list because a request may land under
  GDPR and CCPA simultaneously; the controller is accountable
  under both and should cite both.
- **`scope.surfaces[].in_scope = false` must carry a `reason`.** A
  silently skipped surface is the most common audit gap.
- **`actions[*].pre_state_hash` and `post_state_hash`** are stable
  hashes of the affected rows / artefacts; for LLM unlearning
  they are checkpoint hashes. These hashes are what a regulator
  can replay against to confirm the action happened.
- **`actions[*].method`** is required whenever the action is an
  unlearning or retraining operation. For SISA it is exact; for
  NPO/influence it carries the measured gate metrics
  (§2.3).
- **`bit_identity_check`** (SISA only) records the result of the
  gold test from [[exercise-02]] §3.1. An unlearning action without
  a bit-identity result is not an exact-unlearning claim.
- **`gate_metrics`** (for Path D; added as an optional sibling of
  `method_params`) carries the TOFU-style numbers from
  [[exercise-05]]: `fq_pvalue`, `mu_before`, `mu_after`,
  `mia_auc_before`, `mia_auc_after`, dataset hashes.
- **`art19_notified`** is required true/false on any downstream
  surface where disclosed recipients exist. "Unknown recipients"
  is not acceptable — either the recipient list is maintained or
  the data should not have left.

### 2.3 Path D example (approximate unlearning entry)

For an Art. 17 request routed to Path D (NPO on an LLM), the
`actions[*]` entry for `surface = model-artifacts` differs from the
SISA example as follows:

```jsonc
{
  "surface": "model-artifacts",
  "action": "unlearn",
  "method": "NPO",                               // Zhang et al. 2024
  "method_params": {
    "base_model": "tinyllama-1.1b-chat",
    "retain_alpha": 1.0, "npo_beta": 0.1,
    "steps": 200, "lr": 1.0e-5,
    "forget_set_hash":  "sha256:71a2…ee04",
    "retain_set_hash":  "sha256:03bd…f017",
    "seed": 17
  },
  "gate_metrics": {                              // see [[exercise-05]]
    "fq_pvalue_before": 0.0001,
    "fq_pvalue_after":  0.74,
    "mu_before":        0.94,
    "mu_after":         0.84,
    "mia_auc_before":   0.78,
    "mia_auc_after":    0.54,
    "gate_pass": true,
    "gate_thresholds":  { "fq_pvalue_min": 0.5, "mu_relative_drop_max": 0.15, "mia_auc_max": 0.60 }
  },
  "pre_artifact_hash":  "sha256:cafe…0001",
  "post_artifact_hash": "sha256:cafe…02ff",
  "executed_at": "2026-11-21T13:52:11Z",
  "evidence_ref": "s3://privacy-vault/dsr_01HJM6…/04-model.jsonl.enc"
}
```

The important property is that *the gate is in the audit entry*.
A reviewer can read the entry and confirm (a) which method ran,
(b) whether the operational gate passed, and (c) under what
thresholds the organisation claims the gate is sufficient. A
reviewer who disagrees with the thresholds can route the request
to Path E ([[exercise-06]]) retrospectively.

### 2.4 Integrity envelope

Audit entries are append-only and tamper-evident:

1. **Per-entry signature.** Every entry is signed over its canonical
   JSON form (RFC 8785 canonicalisation) with a KMS-managed key.
   The signer key is rotated quarterly; the key id is in the
   envelope so a verifier can resolve historical keys.
2. **Hash chain.** Each entry carries `prev_entry_id` and
   `merkle_leaf`. The leaf is `sha256(canonical(entry))`; the leaves
   roll into a daily Merkle tree whose root is published to a
   write-once sink (object-lock storage; optionally a transparency
   log).
3. **Append-only store.** Entries are written to immutable storage
   (S3 Object Lock, GCS Bucket Lock, or equivalent) with a legal
   hold of the retention period.
4. **Supporting evidence envelope.** Hashes in `pre_state_hash`,
   `post_state_hash`, `evidence_ref` point at encrypted blobs in a
   privacy vault. The blobs are themselves content-addressed by
   hash; the audit log carries the hash, not the content.

A regulator-facing verification tool takes an entry (or a date
range), walks the Merkle chain, verifies the signatures, and
exports a human-readable PDF with every action, hash, and evidence
pointer resolved.

### 2.5 Retention and access

- **Retention.** Audit entries are retained for the longer of the
  statutory retention (GDPR Art. 30: for the duration of
  processing + typically 3–7 years under local tax/accounting
  law) and the DPIA ([[mod-111]])-defined retention. The retention
  window applies to the audit entries themselves, not to the
  subject data — the point of audit is that the record of
  processing outlives the processing.
- **Access.** Read access to the audit log is restricted to the
  privacy function and legal counsel; operators can write entries
  for requests they handled but cannot read other operators'
  entries. Access events are themselves audit entries
  (`right: "audit-read"`).
- **Pseudonym-vault access.** The mapping from
  `subject_pseudonymous_id` to the identifier is in a separate
  vault accessible only by privacy counsel; access triggers a
  distinct audit entry.
- **Legal hold.** On litigation hold, entries referenced by a
  matter are flagged non-deletable even past the retention
  window.

### 2.6 Governance mapping (NIST AI RMF)

- **Govern.** Audit schema and retention policy are published in
  the privacy-program charter; schema changes go through legal and
  privacy review.
- **Map.** The capture-point map (§2.2 of [[exercise-01]]) is the
  surface inventory; audit entries cite surfaces from that list.
- **Measure.** DSR audit entries are the Measure artefact for the
  DSR control. Dashboards roll up: entries per week, SLA compliance,
  Art. 17(3) refusal rate, method-mix (Path A..E), gate-failure
  rate.
- **Manage.** SLA breaches and gate failures trigger
  post-mortems; schema drift (fields deprecated, new paths added)
  is reviewed quarterly.

### 2.7 Threat model

- **MITRE ATLAS `AML.T0020` (Data from Local System).** A malicious
  operator could tamper with the evidence blobs. The integrity
  envelope and two-person integrity mitigate.
- **ATLAS `AML.T0024` (Exfiltration via ML Inference API).** The
  audit log itself is a secondary target: it can reveal which
  subjects requested erasure (a privacy signal). Pseudonymising
  the subject identifier limits the leak; access control bounds
  the disclosure.
- **OWASP ML Top 10 ML04 (Data Poisoning).** Attackers submitting
  mass erasure requests can poison the audit log and starve
  engineering capacity; identity verification and the batching
  cap in [[exercise-06]] bound this.
- **Insider — audit coverage.** An operator who omits a surface
  from the `scope` block hides the omission. The two-person
  review and the capture-point map (§2.2) are the control.

## 3. Implementation / artifact shape

The committed artefact:

```
mod-110/exercise-07-dsr-audit-trail-authoring/
├── SOLUTION.md                     # this file
├── schema/
│   ├── dsr-audit-v1.0.schema.json  # JSON Schema (draft-2020-12)
│   └── canonicalisation.md         # RFC 8785 note
├── examples/
│   ├── erase-sisa.json             # §2.1
│   └── erase-npo.json              # §2.3
├── policy/
│   ├── retention-and-access.md     # §2.5
│   └── integrity.md                # §2.4
├── capture-points.md               # §2.2 × [[exercise-01]] §2.2
└── review-checklist.md             # §4
```

The JSON Schema constrains:

- `schema_version` to `dsr-audit/1.x`;
- `right` to the enum from [[exercise-01]] §2.1;
- `actions[*].surface` to the nine surfaces from [[exercise-01]]
  §2.2;
- `actions[*].method` to the Path A..E method taxonomy of
  [[exercise-06]];
- `outcome.status` to `fulfilled | refused | partially-fulfilled`;
- `integrity` requires `signer_key_id`, `signature`, and
  `hash_chain`.

## 4. Validation steps

A reviewer runs the checklist against any candidate entry
before committing.

1. **Required-fields check.** The JSON validates against the pinned
   schema; no required field is null.
2. **Two-person integrity.** `created_by ≠ reviewed_by`. Entries
   failing this check are rejected.
3. **Scope completeness.** Each of the nine surfaces appears in
   `scope.surfaces`; out-of-scope surfaces carry a `reason`.
4. **Action/hash parity.** Every in-scope surface has at least one
   `actions[*]` entry with a non-null `pre_state_hash` and
   `post_state_hash`.
5. **Method evidence.** Any `action = unlearn | retrain` entry
   either contains a `bit_identity_check` (exact methods) or a
   `gate_metrics` block with threshold pass (approximate methods).
6. **SLA arithmetic.** `deadline_at ≥ intake_at + 30 days`; if
   `extension_notified_at` is set, it is ≤ `intake_at + 30 days`
   and `deadline_at ≤ intake_at + 90 days`.
7. **Art. 19 cover.** Every downstream surface action has
   `art19_notified` set true/false explicitly.
8. **Refusal citation.** If `outcome.status = refused`,
   `outcome.refusal_grounds` cites an Art. 17(3) subparagraph by
   letter.
9. **Integrity envelope present.** `integrity.signer_key_id`,
   `integrity.signature`, and `integrity.hash_chain.prev_entry_id`
   are populated; the signature verifies against the referenced
   key.
10. **Evidence references resolve.** Every `evidence_ref` is
    present in the privacy vault and the hash of the referenced
    blob matches the `pre_state_hash` / `post_state_hash` it
    stands behind.
11. **Linkage.** The entry references the `dsr_*` ticket in a
    system (ticketing, case management) where the subject-facing
    exchange lives; the ticket id matches `request.request_id`.
12. **Governance field present.** `governance.function` is one of
    Govern / Map / Measure / Manage. For DSR fulfilments the
    normal value is `Measure`.

## 5. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Schema is a JSON-Schema document covering the required fields | 2 | §2.1 |
| Both SISA and approximate-unlearning examples validate against the schema | 2 | §2.1, §2.3 |
| Pseudonymised subject identifier; separate vault for the pseudonym-to-identifier map | 1 | §2.2 |
| Integrity envelope: signature + hash chain + immutable storage | 1 | §2.4 |
| Capture-point map ties fields to [[exercise-01]] §2.2 surfaces | 1 | §2.2 |
| Retention / access policy present; access to the log is itself audited | 1 | §2.5 |
| Governance mapping (NIST AI RMF function) per entry | 1 | §2.6 |
| Review checklist present and runnable by a reviewer in minutes | 1 | §4 |

Automatic failures:

- Single-person integrity (`created_by = reviewed_by`).
- Storing the direct subject identifier in the audit entry.
- Any unlearning action without `bit_identity_check` or
  `gate_metrics`.
- No integrity envelope (signature, hash chain).
- No `reason` on an out-of-scope surface.
- Art. 17 refusal without an Art. 17(3) citation.

## 6. Common mistakes

- **Writing to a mutable log.** A log on ordinary storage with
  rolling writes is not an audit trail; it is a diary. Object-lock
  or an equivalent immutability control is mandatory.
- **Carrying the subject's direct identifier.** Pseudonymise with a
  salted hash held in a separate vault. The audit log is a long-
  retention artefact; the identifier is not.
- **Hashing nothing.** `pre_state_hash` and `post_state_hash` are
  the primitive that lets a regulator verify the action happened.
  Entries without them are narratives, not evidence.
- **Omitting the gate in Path D entries.** A Path D entry that says
  "method: NPO" without the gate metrics is a method label with
  no finding. The gate is the finding.
- **Single-person integrity.** Operator and reviewer must differ.
  The two-person rule is cheap and catches both honest omissions
  and malicious edits.
- **Skipping Art. 19.** Downstream recipients must be notified on
  rectify / erase / restrict. The entry must record whether the
  notification was sent and, if not, cite the "impossible or
  disproportionate effort" exception with reasoning.
- **Refusing without citing a subparagraph.** Art. 17(3) has five
  specific grounds. "Legal obligation" by itself is not enough;
  cite 17(3)(b) — "compliance with a legal obligation under Union
  or Member State law" — and name the law.
- **Treating the audit log as read-only for ops.** Operators must
  write their own entries; reading the whole log is a legal /
  privacy function. Confusing the two inflates access and
  weakens separation of duties.
- **No linkage to the ticket.** Audit entries without a
  `request_id` cross-reference leave the subject-facing exchange
  dangling. A regulator asking "what was the subject told?" needs
  the ticket.
- **Hash drift.** If `pre_state_hash` is computed over a different
  row set than `post_state_hash`, the entry's claim collapses.
  Define the canonicalisation of the hashed artefact once per
  surface and reuse.
- **Over-logging.** The entry should not carry raw PII. Every
  field in the entry should be either machine-identifiable
  metadata, a pseudonym, or a hash / reference. The
  evidence blobs in the vault hold the PII; the log points to
  them.

## 7. References

- Regulation (EU) 2016/679 — General Data Protection Regulation,
  Articles 12, 15, 17, 19, 22, 28, 30, 32.
  <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- ICO Guidance — Documentation (Records of processing).
  <https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/documentation/>
- RFC 8785, JSON Canonicalisation Scheme (JCS).
  <https://www.rfc-editor.org/rfc/rfc8785>
- RFC 7515, JSON Web Signature.
  <https://www.rfc-editor.org/rfc/rfc7515>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST SP 800-92, *Guide to Computer Security Log Management*.
  <https://csrc.nist.gov/pubs/sp/800/92/final>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
