# Exercise 04 — Data Privacy Vault + Tokenisation Pattern — Reference Solution

## 1. Solution overview

This is an **implementation + architecture exercise**. A **data privacy
vault** is a bounded, independently secured store that holds sensitive
identifiers (government IDs, emails, phone numbers, payment data,
health identifiers, biometric templates) and returns an application-
scoped **token** that the rest of the system uses in place of the raw
value. Downstream services, ML pipelines, and logs never see the raw
value; they see the token, and must call the vault to detokenise when
genuinely necessary.

The assessable output has three parts:

1. A **vault architecture** diagram and component list with the trust
   boundaries made explicit.
2. A **tokenisation pattern** — format, properties, collision/rotation
   handling, and *which* tokenisation function to use for which field
   (vaulted token vs format-preserving pseudonym vs cryptographic
   pseudonym vs hash).
3. A **worked implementation** — a vault service skeleton (HTTP API,
   storage layout, access controls, audit log) plus client-side
   middleware that intercepts outbound data to apply tokenisation.

An implementation that uses "a hash" everywhere fails. So does one that
uses reversible tokens but puts the detokenisation key in the same
account as the application.

## 2. Worked answer

### 2.1 Why a vault — the single architectural claim

The vault pattern is justified by a single claim: **blast-radius
containment**. If a service is compromised or a log is exfiltrated, the
attacker should be left holding tokens, not identifiers. The vault is
the control that makes this true, which is why its *isolation* is more
important than its cryptography.

Three corollaries fall out of the claim:

1. The vault is a separate trust boundary. Its credentials, its KMS
   keys, and its audit log are not accessible from the main application
   account/plane.
2. The vault is the only path from token back to value. There is no
   "fast path" through the backing store or the KMS.
3. Every detokenisation is a logged, purpose-tagged event. "The app
   needed to see the email" is not an answer; "the email-sender
   service called detok for purpose `send_receipt` under activity
   `txn-receipts`" is.

### 2.2 Component list

- **Vault API service** — HTTPS service exposing `tokenize`,
  `detokenize`, `rotate`, `revoke`, `lookup_token_by_blind_index`, and
  `attest_token`. Deployed in an isolated project/account with its own
  IAM scope.
- **Vault store** — the backing database (e.g., a relational store
  with per-row envelope encryption). Rows store ciphertext + a blind
  index + metadata (classification, retention, purpose, created_by).
- **KMS / HSM** — the key store. The vault's data-encryption keys
  (DEKs) live under a vault-owned key-encryption key (KEK) in a
  separate account; the application plane has no access.
- **Token format library** — client-side code that constructs,
  validates, and parses tokens. Tokens are self-identifying but carry
  no payload (see 2.4).
- **Audit sink** — append-only log of every tokenise/detokenise event.
  Replication to a second region for incident response.
- **Policy engine** — authorises each detokenise call against the
  caller's workload identity, the stated purpose, and the token's
  class.
- **Rotation / retention jobs** — schedule DEK rotation, re-encrypt in
  place where required, and destroy tokens that cross the per-class
  retention clock.
- **Discovery / scrub shim** — client-side middleware that intercepts
  outbound traffic (logs, analytics, inference prompts) and refuses to
  serialise raw sensitive fields. Discussed in 2.6.

### 2.3 Trust boundaries

```
+----------------------------+        +----------------------------------+
|  Application plane         |        |  Vault plane (separate account)  |
|  (services, ML jobs, logs) |        |                                  |
|                            |        |  +--------+   +--------+         |
|  Can see tokens.           |  mTLS  |  | Vault  |---| Vault  |         |
|  CANNOT see KMS.           +------->|  | API    |   | store  |         |
|  CANNOT read vault store.  |        |  +---+----+   +--------+         |
|                            |        |      |                           |
|                            |        |      v                           |
|                            |        |  +---+----+                      |
|                            |        |  | KMS /  |  Keys never leave    |
|                            |        |  | HSM    |  this plane.         |
|                            |        |  +--------+                      |
+----------------------------+        +----------------------------------+
```

Important specifics:

- The application plane has an IAM principal that can call the vault
  API over mTLS; it has **no direct IAM path** to the vault's KMS keys
  or the vault's storage account.
- Vault-plane administrators are a separate group from
  application-plane administrators. Dual-control required for bulk
  export.
- The vault audit log ships to a destination the application plane
  cannot write or delete from.

### 2.4 Tokenisation pattern — format and properties

A vault token is **random**, not derived from the value, because derived
tokens leak via dictionary attack whenever the value space is small
(phone numbers, ZIP codes, government IDs).

- **Length:** 128 bits of random input, base32-encoded with class and
  version prefix. Example: `tok_v1_email_01HFZQYKD6ZT2V5W0K3A1Q6K7X`.
- **Prefix:**
  - `tok_` — namespace separator.
  - `v1_` — token-format version (for format migration).
  - `email_` — class, so routing / scrubbers do not need to call the
    vault to know what a field *is*.
- **Collision:** 128 bits is comfortably large. The vault still checks
  uniqueness at insert time and retries on collision.
- **Rotation:** on key rotation, re-encrypt the value under a new DEK;
  the *token* stays the same. On token revocation, the token becomes
  invalid; subsequent detok calls return `410 Gone`, and the audit
  trail retains the mapping for incident response.
- **Determinism / joinability:** vault tokens are random — the same
  email tokenised twice yields two different tokens. When determinism
  is required (e.g., to deduplicate a bulk import), the vault exposes
  a **blind index**: HMAC(secret_key, normalised_value), stored on
  the row, searchable without decryption. The secret key never leaves
  the vault plane.
- **Format-preserving (FPT):** avoid unless an external system
  requires the original format (e.g., a card-processing legacy that
  demands a 16-digit PAN). FPT constrains entropy and often fails
  closed under rotation. If used, cite the NIST SP 800-38G caveats.
- **Attestation:** `attest_token` returns a signed statement that the
  caller legitimately tokenised the value at a given time, without
  returning the token itself. Useful for third-party processors that
  need to prove possession without re-emitting the token.

### 2.5 Choosing the right transformation per field

Not every sensitive field belongs in the vault. The decision tree is:

- **Does the value need to be reversible for a legitimate purpose?**
  - Yes → **vault token**. The vault holds the raw value and issues a
    random token. Examples: email (must send mail), phone (must send
    SMS), full name on a receipt, payment card data (plus PCI DSS
    scope).
  - No → go to the next question.
- **Does the value need to be joined across systems but not
  reversible?**
  - Yes → **blind index** (HMAC with a vaulted secret) or a
    cryptographic pseudonym under a vaulted key. Examples: user
    identifier for cross-service joins where the services should not
    see the actual identifier.
  - No → go to the next question.
- **Is the value needed only as a bucket (deduplication,
  cardinality)?**
  - Yes → **salted hash**, stored and used only for that bucketing
    purpose; the salt is vaulted and rotated.
  - No → the value probably should not be processed; revisit
    collection.

The result is a per-field table. A fragment for the chatbot scenario:

| Field | Purpose | Transform | Reversible? | Key / salt owner |
|---|---|---|---|---|
| `email` | Send transactional mail | Vault token | Yes, via vault | Vault KEK |
| `account_id` | Cross-service join | Cryptographic pseudonym under a vaulted key | Yes, but only in the DSR enclave | Vault KEK |
| `government_id` | KYC check only | Vault token | Yes, KYC service only | Vault KEK |
| `ip_address` | Abuse signal bucket | Salted hash to `/24` | No | Vault salt |
| `postal_code` | Geo-coarse analytics | Truncate to first three characters | Not applicable | — |
| `payment_pan` | Settlement | PCI-compliant tokenisation via PCI token service | Yes, in PCI zone only | PCI provider |

Design rule: **put only what you must detokenise in the vault**. The
vault is expensive — fewer round-trips is better. For analytics-only
fields, prefer a hash / truncation that never needs detokenisation.

### 2.6 Worked implementation — HTTP API and schema

API:

```
POST   /v1/tokenize
  body: { class: "email", value: "alice@example.com",
          purpose: "support-identity", retention_days: 1825 }
  200:  { token: "tok_v1_email_01HFZ..." }

POST   /v1/detokenize
  body: { token: "tok_v1_email_01HFZ...",
          purpose: "send-receipt",
          activity_id: "txn-receipts" }
  200:  { value: "alice@example.com", audit_id: "aud_..." }
  403:  policy denial — purpose/caller not permitted for class

POST   /v1/lookup
  body: { class: "email", blind_index_hex: "..." }
  200:  { tokens: ["tok_v1_email_..."] }

POST   /v1/attest
  body: { token: "tok_v1_email_...", nonce: "..." }
  200:  { signed_attestation_jws: "..." }

POST   /v1/rotate (admin only, dual-control)
  body: { class: "email", from_kek_version: 3, to_kek_version: 4 }

POST   /v1/revoke
  body: { token: "tok_v1_email_...", reason: "dsr-erasure sub_..." }
```

Storage layout:

```sql
CREATE TABLE vault_row (
  token            TEXT PRIMARY KEY,
  class            TEXT NOT NULL,
  blind_index      BYTEA NOT NULL,
  ciphertext       BYTEA NOT NULL,
  dek_version      INT  NOT NULL,
  purpose          TEXT NOT NULL,
  activity_id      TEXT,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  retention_days   INT NOT NULL,
  destroy_after    TIMESTAMPTZ NOT NULL,
  revoked_at       TIMESTAMPTZ,
  revoked_reason   TEXT
);
CREATE INDEX vault_blind ON vault_row (class, blind_index);
CREATE INDEX vault_destroy ON vault_row (destroy_after)
  WHERE revoked_at IS NULL;

CREATE TABLE vault_audit (
  audit_id      TEXT PRIMARY KEY,
  event_type    TEXT NOT NULL, -- tokenize / detokenize / lookup / attest / rotate / revoke / destroy
  token         TEXT,
  class         TEXT,
  purpose       TEXT NOT NULL,
  activity_id   TEXT,
  caller_spiffe TEXT NOT NULL,
  caller_ip     INET,
  decision      TEXT NOT NULL, -- allow / deny
  reason        TEXT,
  happened_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Policy engine (pseudocode):

```python
def authorise_detokenize(req, caller):
    policy = load_class_policy(req.class)
    if caller.spiffe_id not in policy.allowed_callers:
        return deny("caller not in allowlist")
    if req.purpose not in policy.purposes[caller.spiffe_id]:
        return deny("purpose not authorised for caller")
    if rate_exceeded(caller, policy):
        return deny("rate limit")
    if token_is_revoked(req.token):
        return deny("token revoked")
    return allow()
```

Client middleware (outbound scrub, Python-ish sketch):

```python
SENSITIVE_PATTERNS = {
    "email": re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}"),
    "phone_e164": re.compile(r"\+\d{8,15}"),
    # ...
}

def outbound_scrub(record: dict) -> dict:
    for key, value in list(record.items()):
        if not isinstance(value, str):
            continue
        for class_name, pattern in SENSITIVE_PATTERNS.items():
            if pattern.fullmatch(value):
                record[key] = vault_client.tokenize(class_name, value,
                                                   purpose="scrub-outbound")
                break
    return record
```

This middleware runs on the log / analytics / inference egress path. A
value that reaches a log line without having been tokenised is a bug;
the middleware reduces the blast radius while the collection-time code
is fixed.

### 2.7 Lifecycle specifics

- **DSR erasure (GDPR Art. 17):** vault row is revoked and value
  destroyed. The *token* remains in the audit trail for incident
  reconstruction; the mapping does not. Downstream references become
  "unresolvable," which is what the data model should assume.
- **Retention (GDPR Art. 5(1)(e)):** `destroy_after` is set at
  tokenise time; a nightly job revokes + destroys expired rows.
- **Breach (GDPR Art. 32–34):** since application-plane services hold
  only tokens, a compromise of the application plane does not
  (standalone) create a notifiable breach. Vault-plane compromise is
  catastrophic — plan incident response accordingly.
- **Transfers (GDPR Art. 44–49):** vault-plane residency is pinned to
  a jurisdiction; tokens may cross borders freely because they carry
  no personal data.

## 3. Validation steps

1. **Blast-radius test.** Simulate an application-plane compromise.
   What leaks? If raw values leak, the vault is in the wrong account
   or the client-side scrub is missing.
2. **Key-isolation test.** Confirm that application-plane IAM cannot
   assume any role in the vault account, and cannot read the KMS keys
   even in read-only form.
3. **Audit test.** Call `detokenize`. Confirm that the audit row names
   the caller SPIFFE id, the purpose, and the decision. Try to call
   without a purpose; confirm `400` and no decryption.
4. **Policy test.** Attempt a cross-class detok (an email service
   trying to read `government_id` tokens). Confirm `403` and a denial
   audit row.
5. **Rotation test.** Rotate the KEK for one class. Confirm that
   existing tokens still resolve, with `dek_version` updated on each
   touched row, and that previous KEK versions are retired on
   schedule.
6. **Determinism / blind-index test.** Tokenise the same email twice.
   Confirm that two distinct tokens are issued, and that the blind
   index allows lookup to find both.
7. **FPT caveat test.** If FPT is used, run the test suite the NIST
   SP 800-38G guidance recommends and keep the result in the audit
   artefacts.
8. **DSR revocation test.** Revoke a token; confirm detok returns
   `410 Gone` and the destruction job removes the row after the
   configured grace period.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Vault isolated in a separate account / project with its own KMS | 2 | Shared KMS fails this item |
| Tokens are random (not value-derived) by default; alternatives justified | 1 | Hash-as-token fails unless reversibility is explicitly not needed |
| Per-field transform table present and reasoned | 2 | Treating every field identically fails |
| API includes tokenize / detokenize / lookup (blind index) / rotate / revoke / attest with policy enforcement | 1 | Missing revoke / rotate fails |
| Audit log is append-only, carries purpose, caller, decision, and ships out of plane | 1 | Silent decryption fails |
| Client-side scrub middleware present, tested | 1 | Reliance on developer discipline alone fails |
| DSR erasure path specified: revoke + destroy + audit | 1 | "We delete" without sequencing fails |
| Rotation path specified: token stable, DEK versioned, KEK rotation plan | 1 | No rotation plan fails |

Automatic failures:

- Vault keys in the application-plane account.
- Derived tokens (hash of value) presented as a reversible mechanism
  when the value space is small.
- Audit log that the application plane can delete.
- No scrub on log / analytics / inference egress.
- No concept of purpose on detokenisation.

## 5. Common mistakes

- **Vault-as-a-table.** A vault is a plane, not a table. A table with
  "sensitive values" inside the application account is encryption
  theatre.
- **Reversibility you never use.** If a field never needs to be
  reversed, a vault token is overkill. Use a hash or truncation —
  less work, fewer calls, smaller audit trail.
- **FPT everywhere.** Format-preserving tokenisation looks convenient
  and silently degrades security for low-entropy fields. Reach for it
  only when an external system forces the format.
- **Deterministic tokens as the default.** Deterministic tokens leak
  joinability across services; use blind indexes when joins are
  needed, and keep the join-secret in the vault.
- **Purpose stuffed into logs rather than enforced.** Purpose must be
  a *required input* to detokenise and a *policy-engine key*, not a
  comment field.
- **Rotating tokens instead of keys.** Rotating tokens breaks every
  foreign key in the application. Rotate the key under the token.
- **Shared service account.** One IAM identity that can tokenise and
  detokenise everything removes the whole point of least privilege.
- **Treating the vault as the whole story.** The vault reduces blast
  radius for the fields it holds. It does not minimise collection
  (see exercise 03), does not manage consent (see exercise 02), and
  does not replace a PCI token provider when PCI DSS applies.

## 6. References

- PCI Security Standards Council — PCI DSS v4.0, and the PCI SSC
  Information Supplement on Tokenisation Product Security Guidelines.
  <https://www.pcisecuritystandards.org/document_library>
- NIST SP 800-57 Part 1, Revision 5 — Recommendation for Key
  Management. <https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf>
- NIST SP 800-38G Revision 1 — Recommendation for Block Cipher Modes
  of Operation: Methods for Format-Preserving Encryption (draft /
  current status to be checked on NIST site). <https://csrc.nist.gov/publications/sp800>
- NIST SP 800-188 (final) — De-identification of Government Datasets
  Containing Personally Identifiable Information.
  <https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-188.pdf>
- Regulation (EU) 2016/679 (General Data Protection Regulation),
  especially Arts. 4(5) on pseudonymisation, 5(1)(c) on minimisation,
  25 on by-design, 32 on security of processing, 34 on breach
  notification. <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- NIST AI Risk Management Framework (AI RMF 1.0). <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST AI 600-1 — Generative AI Profile.
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
- OWASP Machine Learning Security Top 10 — ML06 AI Supply Chain
  Attacks and ML07 Model Theft; vaulted architectures reduce the
  utility of exfiltrated logs. <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — Exfiltration via ML Artifact techniques (e.g.,
  `AML.T0025`-class entries, current IDs on the ATLAS site).
  <https://atlas.mitre.org/>
- VeriSwarm Trust Center — example of a published trust-center page
  describing vaulted architectures as a practitioner reference.
  <https://veriswarm.ai/trust>
