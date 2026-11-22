# Exercise 06 — RAG System Card — Privacy Slice — Reference Solution

## 1. Solution overview

This is a **design exercise**. The learner must produce the
**privacy slice** of a RAG system card — the public (or
reviewer-facing) document that describes, in enough detail for a
privacy reviewer or an external auditor to evaluate, how the system
handles personal and tenant data from ingest to completion.

A "system card" in this track is the artefact the organisation
publishes alongside a RAG / agent product: what it does, what data
it uses, how it is governed, what it cannot do safely, and what
the operator commits to. The privacy slice is the subset of that
document focused on data flows, retention, access, controls, and
residual risks. It is the document a DPO or an external reviewer
reads first.

Scope — what belongs in the slice:

- Data types and categories handled (what, at what sensitivity).
- Where data lives at each lifecycle stage and who can touch it.
- The specific controls from [[exercise-01]], [[exercise-03]],
  [[exercise-04]], and [[exercise-05]] that back the claims, named
  explicitly so a reviewer can pull threads.
- Retention, deletion, DSR workflow, and cross-border handling.
- Residual risks that are *not* controlled by the system (so the
  reviewer knows what to cover elsewhere).
- Governance mapping to NIST AI RMF functions so the card plugs
  into the operator's existing risk register without re-work.

What does not belong (so the card remains honest):

- Product marketing. The system card is a control document.
- Metrics without method. Any quantitative claim (e.g., recall,
  latency, P(leak)) names how it was measured and over which
  corpus.
- Claims the system does not actually enforce. If the card says
  "no training on customer data" and the training pipeline has no
  mechanical block, that is a false claim, not a goal.

The assessable output is one Markdown document,
`PRIVACY_SYSTEM_CARD.md`, structured around the nine sections in
§2. The reference below shows the content at the level of
specificity a reviewer should accept.

## 2. Worked implementation — reference system card content

### 2.1 Product and version identity

Required fields:

- Product name and version (semver).
- The exact base model(s) and embedding model(s) used, including
  whether fine-tuned, and against what corpus.
- The retrieval engine (name, version), the reranker (if any),
  and the agent framework version.
- Build identifier (commit hash) for the RAG orchestration code.
- Change-log pointer; who maintains the card; when it was last
  reviewed.

Why this section matters: a card that cannot be pinned to a
specific build is not a control; it is marketing. The build hash
is what lets a reviewer tie a claim ("prompt redaction is enforced
at shipper v1.3") to a provable artefact.

### 2.2 Data categories and sensitivity

A table of every category of data the system may handle, with the
sensitivity classification and the lawful basis / relevant
regulation.

| Category | Source | Sensitivity | Relevant basis |
|---|---|---|---|
| Tenant knowledge-base documents | Tenant upload | Confidential / Restricted | Data processing agreement |
| End-user prompts | Live user traffic | Confidential | Contract performance |
| Retrieved chunks | Derived from KB | Inherits KB sensitivity | DPA |
| Completions | Model output | Derived; may contain any of the above | DPA |
| Tool-call arguments / results | Agent activity | Confidential | DPA |
| Operational telemetry (no content) | System | Internal | Legitimate interest |
| Verbatim eval sample | Opt-in tenants only | Confidential; `debug` retention | Separate consent |

Minimum fidelity the card must reach: per-category sensitivity,
whether the category ever leaves the tenant's region, and whether
it ever enters a training pipeline.

### 2.3 Data flow

A diagram or an equivalent bullet list covering every hop from
user input to persistent store. The reference (same shape as
[[exercise-01]] §2.3):

```
user -> gateway (OIDC / mTLS, request-scoped identity)
      -> agent (no service-identity for tool calls)
      -> retriever (tenant partition pinned; pre-ANN metadata filter)
      -> reranker (tenant-pure batches)
      -> prompt builder (per-chunk tenant check)
      -> LLM (stateless; no raw prompt retained in model state)
      -> response filter
      -> user

side channels:
  - log pipeline: event -> shipper -> redactor -> log store
  - eval sampling: per-tenant opt-in, 7-day retention, per-read audit
  - DSR workflow: identity-lookup by keyed HMAC, erasure by key rotation
```

The card annotates each arrow with: whose identity flows, what
gets logged there (and at what event class), and which [[mod-109]]
exercise contains the control contract.

### 2.4 Named controls

Every claim in later sections must be backed by a named control in
this section. The reference card includes at minimum:

| Control | Source | Claim it backs |
|---|---|---|
| C1 Gateway AuthN, bound principal | [[exercise-01]] | No caller can impersonate a tenant |
| C2–C3 Pre-ANN partition + metadata filter | [[exercise-01]] | Retrieval does not cross tenants |
| C4 Prompt-builder re-check | [[exercise-01]] | Defence in depth at prompt assembly |
| C6 Per-tenant embedding cache key | [[exercise-01]] | Caches do not leak queries |
| L1–L4 negative-test harness | [[exercise-02]] | Controls are tested in CI |
| Logging policy taxonomy | [[exercise-03]] | No raw prompts/completions are retained |
| Redacting shipper, fail-closed | [[exercise-04]] | Unredacted events do not reach the log store |
| Keyed HMAC per tenant, rotatable | [[exercise-04]] | DSR erasure is a key operation, not line-by-line |
| Capability tokens per tool, audience-locked | [[exercise-05]] | Agent is not a confused deputy |
| Step-up auth for irreversible tools | [[exercise-05]] | Model cannot drive high-impact actions alone |

A reviewer can walk from any §2.5 claim to its backing control
with no further research.

### 2.5 Privacy properties the system commits to

Each property is paired with how it is enforced, how it is
measured, and what would falsify it.

| Property | Enforcement | Measurement | Falsification signal |
|---|---|---|---|
| No cross-tenant content in retrieval output | C2–C3 (pre-ANN partition + metadata filter) | Negative-test harness [[exercise-02]] on every PR | Any `rag_log_redact_fail_total` event tagged `cross_tenant` |
| No raw prompts, completions, or tool_args in the central log store | C7, shipper of [[exercise-04]] | Conformance tests + CI grep for direct logger calls | Any `redact_fail` or any field in schema with action `raw` where it should be `redact_text` |
| Tenant can erase a subject within DSR SLA (default 30 days) | Keyed-HMAC per-tenant key rotation ([[exercise-04]] + [[mod-110]]) | DSR drill run quarterly; time-to-irreversibility < SLA | A drill that cannot reduce a subject's hashes to non-recoverable within the window |
| Agent cannot act beyond the user's consented scopes | Capability tokens from [[exercise-05]]; scopes are intersection of principal and tool | Scope-intersection test in [[exercise-02]] shape, plus authZ unit tests | Any tool invocation whose `token.scopes ⊄ principal.scopes` |
| Irreversible agent actions require fresh user AMR | Step-up + dry-run + idempotency | Integration test proving `preview=false` without step-up returns 403 | Any high-impact tool configured without `step_up_auth: true` |
| Customer data is not used to train shared models | Pipeline-level block on training ingest from tenant corpora, confirmed by code review | Build-time test that fails if training ingest paths reference tenant stores | Any commit adding a tenant-store path to a training ingest config |

Each falsification signal is a monitorable condition, not an
inspection-only statement.

### 2.6 Retention, deletion, and DSR

- **Retention classes.** As defined by the logging policy in
  [[exercise-03]]: `debug` 7 d, `operational` 30 d, `security`
  90 d, `audit` 1 y (or per regulation).
- **Tenant data.** Tenant-uploaded documents are retained per the
  tenant's contract. Removal from the vector index uses a
  tombstone plus re-index on a documented schedule; the card
  states the time-to-irreversibility for removed documents and the
  mechanism (soft-delete vs. hard-delete vs. re-embed).
- **Data-subject access.** Metadata-only response: list of
  requests (times, tool calls, hashed doc_id touches) under the
  SLA. Raw completion text is included only if the request falls
  in the sampled verbatim window.
- **Data-subject erasure.** Keyed-HMAC rotation plus corpus
  deletion. The card states the SLA, the drill cadence, and the
  responsible role.
- **Backups.** Explicitly named: retention of backups, whether
  backups are subject to redaction (they must be), and how long
  before a deletion propagates through backup rotation.

### 2.7 Access, operator duties, and residual trust

- **Operator access.** Who in the operator's organisation can
  read `debug.*`, `operational.*`, and `audit.*`. Each access is
  logged to `audit.policy_decision`.
- **Break-glass path.** Explicit, time-bound, ticketed. Surfaced
  to the affected tenant within 24 h.
- **Vendor / subprocessor list.** Every third party that can
  touch any of the categories in §2.2: cloud providers, model
  providers, retrieval engines, log stores, PII-detection APIs.
  Linked to the operator's subprocessor register.
- **Residual trust.** What the card does *not* claim to control:
  cloud-provider operators with privileged access; LLM-vendor
  internal logging (name the vendor and point to their published
  policy); third-party embedding-model vendors.

### 2.8 Threats and residual risk

Three threat classes the card's controls address, mapped to the
authoritative inventories:

- **Confidentiality.** Cross-tenant leakage via retrieval, caches,
  or logs — mitigated by [[exercise-01]], [[exercise-04]],
  [[exercise-05]]. Mapped to OWASP ML Top 10 ML06 (AI Supply
  Chain) and MITRE ATLAS AML.T0024 (Exfiltration via ML Inference
  API). NIST AI RMF function: Measure.
- **Integrity of action.** Prompt injection driving tool actions
  outside user intent — mitigated by [[exercise-05]] capability
  tokens and step-up. Mapped to OWASP ML Top 10 ML01 (Input
  Manipulation) and MITRE ATLAS AML.T0043 (Craft Adversarial
  Data). NIST AI RMF function: Manage.
- **Availability of DSR.** Failure to erase on request — mitigated
  by keyed-HMAC rotation and corpus deletion. NIST AI RMF
  function: Govern.

Residual risks the card explicitly does not cover (these belong on
the operator's risk register regardless):

- **Memorisation extraction** against the base model. Covered in
  [[mod-105]]; the card points there and says whether DP
  fine-tuning ([[mod-106]]) is engaged.
- **Model-vendor-side logging** of inference calls. Named vendor,
  linked vendor policy, operator's contractual position.
- **Legitimate-but-misused user intent.** The system cannot
  distinguish a user asking for their own data from a user
  asking for their own data under coercion; this is an
  identity-assurance concern, not a RAG control.

### 2.9 Change and review cadence

- The card is reviewed on every minor release and at a fixed
  minimum cadence (quarterly).
- Changes are versioned alongside the code; the card's version
  pins to a specific build hash.
- A reviewer owner (DPO or equivalent) signs off; the signer is
  named.
- A public changelog records control-affecting changes.

### 2.10 Worked example claim → control chain

To make the "every claim is backed" property concrete, the card
closes with one or two walked chains:

> **Claim.** "A document uploaded by tenant A will never appear
> in a completion returned to tenant B."
>
> **Chain.**
> 1. Enforced by C2–C3 (pre-ANN partition + metadata filter) in
>    the retriever — see [[exercise-01]] §2.3.
> 2. Defended in depth by C4 (prompt-builder re-check) — see
>    [[exercise-01]] §2.3.
> 3. Tested on every PR by the negative-test harness — see
>    [[exercise-02]] §2.4–§2.5.
> 4. Logged with hashed `doc_id`s only; a leak would appear as a
>    cross-tenant entry in `rag.retrieval` — see [[exercise-03]].
> 5. Falsification: any `rag_log_redact_fail_total{reason="cross_tenant"}`
>    event is a page.

> **Claim.** "A refund cannot be issued by the agent without the
> user's fresh consent."
>
> **Chain.**
> 1. `order_refund` has `step_up_auth: true`, `dry_run_first: true`,
>    `idempotent: true` in the capability registry — see
>    [[exercise-05]] §2.2.
> 2. The tool rejects `preview=false` without a fresh AMR claim.
> 3. The dispatcher only mints a `payments:refund` capability
>    token when the principal token's AMR is < 5 min old.
> 4. Falsification: an integration test issues a refund in tests
>    with a stale principal; it must 403.

## 3. Validation steps

Reviewer walks the card through:

1. **Build-pin test.** The card names a product version and a
   build hash. A card that does not pin cannot be audited.
2. **Claim-to-control test.** Every claim in §2.5 and §2.10 has a
   named control in §2.4 and a falsification signal.
3. **Measurement test.** Every quantitative claim names how it is
   measured and over which corpus. No "approximately 99.9%" lines
   without method.
4. **Retention table test.** Every data category in §2.2 has a
   retention class in §2.6.
5. **DSR test.** The card names SLA, mechanism (key rotation +
   corpus deletion), drill cadence, and a time-to-irreversibility.
6. **Residual-risk test.** Memorisation ([[mod-105]]), vendor-side
   logging, and legitimate-intent abuse are listed as residual
   risks — the card does not claim to defeat them.
7. **Vendor-chain test.** Every subprocessor that can touch the
   categories in §2.2 is named.
8. **Cross-reference test.** The card links to [[exercise-01]],
   [[exercise-02]], [[exercise-03]], [[exercise-04]], and
   [[exercise-05]] for the controls it rests on.
9. **Governance mapping.** The card names at least one NIST AI
   RMF function (Govern / Map / Measure / Manage) for each control
   area.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Product version + build-hash pin | 1 | — |
| Data-category sensitivity table | 1 | — |
| End-to-end data flow annotated with controls | 1 | — |
| Named controls section backs every §2.5 claim | 2 | Most-missed property of system cards |
| §2.5 claims paired with falsification signals | 2 | A claim without a signal is a statement, not a control |
| Retention + DSR with SLA + drill cadence | 1 | — |
| Residual risks explicit, not elided | 1 | — |
| At least one NIST AI RMF function per control area | 1 | — |

Automatic failures:

- Any claim in §2.5 without a backing control in §2.4.
- "No training on customer data" asserted with no mechanical
  enforcement path.
- DSR process without an SLA.
- No build-hash / version pin.

## 5. Common mistakes

- **Marketing in a control document.** Phrases like
  "industry-leading" or "best-in-class" have no audit meaning and
  dilute the real claims. Keep the card to what can be shown.
- **Claims without controls.** "We never retain user content" is
  a claim; the backing control is the shipper + schema from
  [[exercise-04]]. The card must point to it.
- **Controls without falsification signals.** A control that can
  silently stop working is not a control. Each §2.5 row names
  what would prove it broken and how the operator would see that.
- **No build pin.** The system changes; the card must be tied to
  a version so a reviewer can audit *that* version.
- **Hiding the residuals.** Vendor-side logging, memorisation,
  and operator-privileged access are real. The card lists them
  because the alternative is a tenant discovering them during an
  incident.
- **Confusing the system card with a privacy policy.** The
  privacy policy is for end users; the system card is for
  reviewers. Different audiences, different fidelity.
- **DSR without a mechanism.** "We respond within 30 days" is
  not a mechanism; "we rotate the per-tenant HMAC key within
  7 days of a verified request and re-index within 30" is.
- **Omitting the subprocessor list.** Every third party that can
  see the data in §2.2 is a privacy surface. If it is not in the
  card, the tenant did not consent to it.

## 6. References

- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS.
  <https://atlas.mitre.org/> (`AML.T0024 Exfiltration via ML
  Inference API`, `AML.T0040 ML Model Inference API Access`,
  `AML.T0043 Craft Adversarial Data`, `AML.T0010 ML Supply Chain
  Compromise`).
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework> —
  Govern / Map / Measure / Manage functions used throughout the
  card's control section.
- Related solutions in this track: [[exercise-01]] per-tenant
  retrieval ACL; [[exercise-02]] negative-test harness;
  [[exercise-03]] prompt and completion logging policy;
  [[exercise-04]] redact-before-log pipeline; [[exercise-05]]
  agent-tool data scope and identity propagation. Residual-risk
  cross-links: [[mod-105]] (memorisation / extraction),
  [[mod-106]] (DP fine-tuning), [[mod-110]] (DSR and unlearning),
  [[mod-111]] (DPIA and privacy reviews).
