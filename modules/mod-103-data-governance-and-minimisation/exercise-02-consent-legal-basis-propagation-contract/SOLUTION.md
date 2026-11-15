# Exercise 02 — Consent / Legal-Basis Propagation Contract — Reference Solution

## 1. Solution overview

This is a **design + implementation exercise**. The learner must write the
contract that attaches a lawful basis (GDPR Art. 6, plus Art. 9 for
special-category data) to a record at **collection time** and carries that
basis through every downstream consumer — storage, features, training,
retrieval, inference, logs — such that any consumer can, before using the
record, answer three questions:

1. Which lawful basis authorises this specific use?
2. Which specific consent / contract / legitimate-interest record proves it?
3. If that record is withdrawn, mutated, or expires, how is this consumer
   notified and what does it stop doing?

The assessable output has four parts:

1. A **propagation contract** — a schema plus semantics that every service
   agrees to honour.
2. A **worked data model** for the consent/basis store, with per-purpose
   versioned records and lifecycle events.
3. A **worked enforcement path** showing the training pipeline rejecting
   a non-compliant join.
4. A **test matrix** plus a decision table for the five commonly muddled
   cases (new user, consent withdrawal, purpose change, lawful basis
   substitution, basis expiration).

An answer that attaches consent only at the *UI* and never to the record
fails. So does an answer that attaches a flag at the record and never
expires or revalidates it.

## 2. Worked answer

### 2.1 Propagation contract — fields and semantics

Every record that is personal data must carry, or be joinable to, a
`BasisClaim`. A `BasisClaim` is the authoritative statement that
"processing operation `P` on subject `S`'s data is authorised at time `T`
by basis record `B`."

The minimum schema:

```json
{
  "basis_claim_id": "bc_01HFZ...",
  "subject_pid": "sub_pseu_8f...",
  "activity_id": "assistant-training-eu",
  "purpose_id": "improve-assistant-quality",
  "basis": "consent",
  "basis_record_id": "consent_01HFZ...",
  "basis_record_version": 7,
  "granted_at": "2024-06-01T08:12:33Z",
  "withdrawn_at": null,
  "expires_at": "2026-06-01T08:12:33Z",
  "jurisdiction": "EU",
  "evidence_uri": "s3://consent-evidence/2024/06/..."
}
```

Field semantics the contract commits every consumer to:

- **`activity_id`** is the join key into the RoPA. One processing
  activity, one `activity_id`. Training is not the same activity as
  inference; inference is not the same as retraining; on-device
  telemetry is not the same as cloud retention. The activity that has
  no RoPA row has no basis.
- **`purpose_id`** is the narrow purpose statement the user saw. "Improve
  assistant quality" is one purpose; "personalised advertising" is
  another; a product cannot silently repurpose one into the other.
- **`basis` ∈ {consent, contract, legal_obligation, vital_interests,
  public_task, legitimate_interests}.** The six Art. 6(1) bases, nothing
  else. For Art. 9 data, add `special_category_basis ∈ Art. 9(2)(a–j)`.
- **`basis_record_id`** points at the authoritative record in the
  consent / LIA / contract store. The claim is a *pointer*, not a copy
  of the state — so a withdrawal in the authoritative record invalidates
  every claim that points at it.
- **`basis_record_version`** pins the consent/LIA version the user saw.
  A notice re-write that changes material terms produces a new version
  and must *not* automatically migrate existing records.
- **`granted_at`**, **`withdrawn_at`**, **`expires_at`** are the
  lifecycle timestamps. `expires_at` is required for consent-based
  claims whose purpose is ongoing; it forces a renewal rather than
  treating consent as perpetual.
- **`jurisdiction`** controls which rights set applies (GDPR, UK GDPR,
  CCPA/CPRA, CPA, VCDPA, …). Rights fan-out logic reads this field.
- **`evidence_uri`** is the minimum: a signed artifact proving the
  consent action (checkbox event, session trace, contract PDF
  reference, LIA sign-off). Required by Art. 7(1) ("the controller shall
  be able to demonstrate that the data subject has consented").

### 2.2 Lifecycle events

The consent/basis store emits five event types; every downstream
consumer MUST subscribe to the events relevant to its activity.

- `basis.granted` — a new `BasisClaim` was created.
- `basis.withdrawn` — a consent record was withdrawn or the LIA was
  retracted. From the event timestamp, this subject is no longer a valid
  participant in that activity.
- `basis.expired` — `expires_at` passed without renewal. Equivalent to
  withdrawn for propagation, distinct for analytics.
- `basis.version_bumped` — the basis record version increased (e.g.,
  updated consent copy). Existing claims pin to the old version;
  downstream behaviour depends on activity policy.
- `basis.purpose_remapped` — the purpose text changed within a
  compatibility boundary approved by legal. Rare; requires a specific
  review.

Consumers MUST:

- Process `basis.withdrawn` and `basis.expired` within the activity's
  revocation SLA (default: 24 hours for training-data filters; 72
  hours for cold stores; faster for marketing bases).
- Treat an unknown `activity_id` on an event as an error (not a no-op).
- Record the event in a per-consumer audit log.

### 2.3 Collection-time attach

At the moment data is collected, the collector:

1. Resolves the subject identifier and the pseudonymous `subject_pid`.
2. Reads the active `BasisClaim` for the `(subject_pid, activity_id)`
   it is collecting for. If none exists, it refuses to collect (GDPR
   Art. 5(1)(a), Art. 6(1)).
3. Writes `basis_claim_id` into the *envelope* of the record, not the
   payload. The envelope travels with the record through every store.
4. Emits a `collection.recorded` event tying the record identifier to
   the `basis_claim_id`.

Envelope example (JSON, abbreviated):

```json
{
  "record_id": "rec_01HFZ...",
  "subject_pid": "sub_pseu_8f...",
  "activity_id": "assistant-training-eu",
  "basis_claim_id": "bc_01HFZ...",
  "collected_at": "2024-06-02T10:00:00Z",
  "payload": { "...": "..." }
}
```

### 2.4 Propagation across pipeline stages

The contract names six propagation stages. Every stage inherits the
envelope's `activity_id` and `basis_claim_id`. If a stage is a *new
processing activity*, it writes a new `BasisClaim` (and must have its
own RoPA row, its own purpose, and possibly its own basis) rather than
inheriting silently.

| Stage | Inherits? | Writes new `BasisClaim`? | Enforcement point |
|---|---|---|---|
| Raw storage (warehouse) | Yes | No | Row-level filter on withdrawal; TTL per activity |
| Feature store | Yes | Only if the feature is used for a different activity | Materialisation filter at join time |
| Training corpus build | Yes | Yes, if training is a distinct activity | Corpus-rebuild filter + consent-delta job |
| Model artifact | N/A — model is not personal data per se, but a *memorisation risk* | No; model carries *dataset lineage*, not per-subject basis | Unlearning / re-training trigger on withdrawal spike |
| Retrieval index | Yes | Only if retrieval is a different activity than source | Index-rebuild filter + per-subject tombstone |
| Inference-time logging | Yes | Yes if logs are used for training | Log retention + consent filter at ingest to training |

Design rule: **inference-time logs that may feed training require their
own `BasisClaim` and their own user-visible notice.** Treating them as
"just operational logs" and then quietly feeding them into training is
the pattern most frequently flagged by regulators.

### 2.5 Worked data model — consent/basis store

```sql
CREATE TABLE basis_record (
  basis_record_id   TEXT PRIMARY KEY,
  subject_pid       TEXT NOT NULL,
  activity_id       TEXT NOT NULL,
  purpose_id        TEXT NOT NULL,
  basis             TEXT NOT NULL CHECK (basis IN (
      'consent','contract','legal_obligation',
      'vital_interests','public_task','legitimate_interests')),
  special_category_basis TEXT,
  jurisdiction      TEXT NOT NULL,
  version           INT  NOT NULL,
  granted_at        TIMESTAMPTZ NOT NULL,
  withdrawn_at      TIMESTAMPTZ,
  expires_at        TIMESTAMPTZ,
  evidence_uri      TEXT NOT NULL,
  prior_record_id   TEXT REFERENCES basis_record(basis_record_id),
  UNIQUE (subject_pid, activity_id, version)
);

CREATE INDEX basis_record_active ON basis_record (subject_pid, activity_id)
  WHERE withdrawn_at IS NULL AND (expires_at IS NULL OR expires_at > now());

CREATE TABLE basis_event (
  event_id        TEXT PRIMARY KEY,
  basis_record_id TEXT NOT NULL REFERENCES basis_record(basis_record_id),
  event_type      TEXT NOT NULL,
  occurred_at     TIMESTAMPTZ NOT NULL,
  payload_json    JSONB NOT NULL
);
```

`BasisClaim` as emitted on the bus is derived from `basis_record` plus
the collection event; it is immutable once published.

### 2.6 Worked implementation — training-pipeline join refusal

```python
# pipelines/training/build_corpus.py
from basis_client import get_active_claim, BasisMissing, BasisWithdrawn

def row_is_authorised(row):
    try:
        claim = get_active_claim(
            subject_pid=row["subject_pid"],
            activity_id="assistant-training-eu",
            at=row["collected_at"],
        )
    except (BasisMissing, BasisWithdrawn):
        return False
    return (
        claim.purpose_id == "improve-assistant-quality"
        and claim.jurisdiction in {"EU", "UK"}
        and (claim.expires_at is None or claim.expires_at > row["collected_at"])
    )

def build(rows):
    authorised = (r for r in rows if row_is_authorised(r))
    yield from authorised

# Unit test — rejects rows with no claim or a withdrawn claim.
def test_training_join_rejects_withdrawn():
    row = {"subject_pid": "sub_test", "activity_id": "assistant-training-eu",
           "collected_at": "2024-06-02T10:00:00Z"}
    stub_withdrawn(row["subject_pid"], "assistant-training-eu")
    assert not row_is_authorised(row)
```

The test is a CI gate. A training build that cannot show a passing test
is not allowed to start.

### 2.7 Decision table — the five muddled cases

| # | Situation | Correct handling |
|---|---|---|
| 1 | **New user signs up and immediately asks to be included in training** | Create `basis_record` for the training `activity_id` with basis=`consent`, version=current notice version, `expires_at` set per policy. Downstream consumers see the claim on the next `basis.granted` event. |
| 2 | **Existing user withdraws consent for training** | Write `withdrawn_at` on the `basis_record`. Emit `basis.withdrawn`. Training pipeline filter rebuilds the corpus excluding the subject within the activity SLA. Model-side response follows the unlearning decision in mod-110. |
| 3 | **Product proposes to use training data for personalised advertising** | *Purpose change*, not a reinterpretation. Requires a new `activity_id`, a new consent prompt with the new purpose text, and a new `basis_record`. Existing training consent does NOT propagate. |
| 4 | **Legal recommends replacing consent with legitimate interests for a downstream analytics use** | Write a new `basis_record` with basis=`legitimate_interests`, pointing to an approved LIA. Do NOT reuse the old `basis_record_id` under a new basis. Notify users per Art. 13(2)(d) / Art. 14 where triggered. |
| 5 | **Consent record expires without renewal** | Treat identically to withdrawal for propagation (`basis.expired`). Do not auto-renew; auto-renewal defeats Art. 7(3) freely-revocable requirements. The user must be asked again. |

## 3. Validation steps

1. **Round-trip test.** Take a single subject, grant consent, exercise
   each downstream consumer, then withdraw. Within the activity SLA,
   every consumer stops using the record and the audit log records the
   stop.
2. **Collection-time refusal test.** Delete the subject's basis
   record; attempt collection; assert collection is refused and the
   refusal is logged.
3. **Replay test.** Publish `basis.withdrawn` with a stale timestamp.
   Consumers MUST still react idempotently; late events do not cause
   erroneous *re-inclusion*.
4. **Purpose-change test.** Rename `purpose_id` in a feature build to
   a new purpose without a new `basis_record`. The pipeline must fail
   the CI gate.
5. **Evidence check.** `evidence_uri` is reachable for every active
   consent record; a sampled daily check surfaces broken URIs.
6. **Jurisdiction filter test.** A US subject's `BasisClaim` flows to
   CCPA-rights handlers, not GDPR Art. 15 pipelines, and vice versa.
7. **Negative test — logs as training feed.** Attempt to pipe
   inference logs into the training corpus without a dedicated
   `activity_id`. The schema check must reject the pipe.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Contract names the six Art. 6(1) bases and the Art. 9(2) sub-bases | 1 | A flag that says "consent / no consent" fails |
| `BasisClaim` carries `activity_id`, `purpose_id`, `basis_record_id`, versioning, and evidence pointer | 2 | Missing versioning or evidence fails the item |
| Lifecycle event set includes granted, withdrawn, expired, version_bumped, purpose_remapped | 1 | Missing withdrawn or expired fails |
| Collection-time refusal path is specified and testable | 1 | "We check downstream" fails |
| Downstream consumers each have a named enforcement point and an SLA | 2 | Stage table required |
| Decision table resolves the five muddled cases correctly | 2 | Each wrong answer costs a point to a floor of zero |
| Audit log entries exist for every stop/start and are named | 1 | Silent stops fail |

Automatic failures:

- Using "consent" as a universal basis. Workforce analytics, security
  logs, and some retention obligations have other bases.
- Letting logs silently feed training without a separate activity.
- Treating purpose change as a notice update.
- "Withdrawal is honoured next time we rebuild" with no SLA.

## 5. Common mistakes

- **One flag at collection time, never revisited.** Consent is a
  lifecycle, not a boolean. Without events, downstream consumers rot.
- **Attaching basis to the UI, not the record.** The record moves
  through many systems; the UI does not.
- **Reusing consent across purposes.** Consent is purpose-specific
  (GDPR Art. 6(1)(a), Art. 7, Recital 32). A purpose change is a new
  basis.
- **Auto-renewing consent.** Automatic renewal violates "freely given"
  (Art. 7(3) in effect). Renewal requires a fresh action.
- **Treating legitimate interests as a free pass.** LIA must be
  documented; balancing-test documented; opt-out accessible; purpose
  compatible; and the user informed (Art. 13/14).
- **Letting the ML pipeline cache ignore consent withdrawals.** A
  feature store built once and served forever is a withdrawal
  violation waiting to happen. Rebuild or filter at serve.
- **Ignoring the Art. 9 layer.** Special-category data needs an Art. 9
  basis in addition to the Art. 6 basis, not instead.
- **Treating US and EU identically.** CCPA/CPRA uses an opt-out model;
  GDPR largely an opt-in model. The contract honours both via
  `jurisdiction`.

## 6. References

- Regulation (EU) 2016/679 (General Data Protection Regulation),
  Arts. 5–7, 9, 13–14, 21, 30. <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- EDPB Guidelines 05/2020 on consent under Regulation 2016/679, final
  version. <https://edpb.europa.eu/>
- EDPB Guidelines 07/2020 on the concepts of controller and processor
  — relevant for where the basis claim is authored. <https://edpb.europa.eu/>
- California Consumer Privacy Act (as amended by CPRA), Cal. Civ. Code
  §§ 1798.100 et seq., and the CCPA Regulations. <https://cppa.ca.gov/regulations/>
- NIST Privacy Framework v1.0 — Core function "Control" (CT) covers
  consent management as a control activity. <https://www.nist.gov/privacy-framework>
- NIST AI Risk Management Framework (AI RMF 1.0), especially the
  `Govern` and `Map` functions. <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST AI 600-1 — Generative AI Profile, section on data provenance.
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
- OWASP Machine Learning Security Top 10. <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — adversary techniques around data-source manipulation
  that a basis-propagation design should defend against. <https://atlas.mitre.org/>
