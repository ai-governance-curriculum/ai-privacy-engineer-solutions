# Exercise 03 — Prompt and Completion Logging Policy — Reference Solution

## 1. Solution overview

This is a **design exercise**. The learner must write the policy that
governs what a RAG / agent system is allowed to log about prompts,
retrieved context, completions, tool calls, and tool results. Logging is
the single highest-leverage privacy surface in LLM applications: the
model does not retain state across turns, but the log pipeline does, and
every observability shortcut ("just log the full prompt until we figure
out what's slow") ends with a retention-shaped incident.

The forcing properties are three:

- **Prompts contain the retrieved chunks.** Whatever the RAG retriever
  pulled from the corpus is now in the prompt. Logging the prompt
  verbatim duplicates the corpus into the log store under a different
  owner, access control, and retention schedule than the source.
- **Completions can regurgitate training or RAG data.** Logging
  completions verbatim can create a secondary store of memorised data
  (training) or verbatim retrieved content (RAG).
- **Tool call arguments carry caller PII.** Function-call arguments and
  tool results routinely contain user identifiers, order IDs, email
  addresses, and anything else the agent passes between tools. These
  look innocuous as structured fields but become PII caches at scale.

The assessable output is a short policy document (`LOGGING_POLICY.md`)
with four parts:

1. **Taxonomy of log events** — what events the system emits, with a
   sensitivity label, retention class, and access role per event.
2. **Field-level rules** — for each event type, which fields are
   logged *raw*, which are *redacted*, which are *hashed*, which are
   *dropped*, and which are *sampled*.
3. **Retention and access** — how long each event class is kept,
   who can read it, and under what approval/break-glass path.
4. **DSR and incident hooks** — how the log store supports data
   subject requests (erasure, access) and incident forensics without
   requiring raw prompt/completion retention.

The policy is implemented by [[exercise-04]] (the redact-before-log
pipeline). This exercise produces the specification the pipeline must
implement.

## 2. Worked implementation

### 2.1 Taxonomy of log events

A workable baseline enumerates six event classes. The policy may add
more; it must not use fewer.

| Event | What it records | Example fields |
|---|---|---|
| `rag.query`           | A single user → RAG turn | request_id, tenant_id, principal_id_hash, query_fingerprint, retrieved_doc_id_hashes, latency_ms, status |
| `rag.retrieval`       | The retrieval step | request_id, engine, partition, filter_hash, k, hits_count, pre_ann_filter: true/false |
| `rag.completion`      | The LLM step | request_id, model_id, prompt_tokens, completion_tokens, finish_reason, refusal: bool |
| `agent.tool_call`     | Agent invoking a tool | request_id, tool_name, tool_version, arg_schema_hash, arg_hash_per_field, caller_identity |
| `agent.tool_result`   | Tool returning data | request_id, tool_name, result_schema_hash, result_size, redaction_applied: bool |
| `audit.policy_decision` | ACL / redaction / rate-limit decisions | request_id, control_id (C1–C8 from [[exercise-01]]), decision, reason_code |

Fields that look like prompt or completion text — `query`, `prompt`,
`response`, `tool_args`, `tool_result` — are **not** in the baseline
schema. They may be opted in per event class under an explicit
sampling rule (§2.3).

Each event carries three labels:

- **Sensitivity**: `public`, `internal`, `confidential`, `restricted`.
  Retrieved chunks and completions default to `restricted` because they
  may contain whatever the tenant ingested.
- **Retention class**: `debug` (7d), `operational` (30d),
  `security` (90d), `audit` (1y or per-regulation).
- **Role**: who is allowed to read this class of event.

### 2.2 Field-level rules

Default: structured metadata is logged, free-form text is not. Every
exception is written down.

| Field | Default action | Rationale |
|---|---|---|
| `request_id`, `tenant_id`, `model_id`, `tool_name`, `status`, `latency_ms`, `k`, `hits_count` | **Log raw** | Pure operational metadata; no user content |
| `principal_id`, `user_email`, `caller_account` | **Hash (keyed HMAC)** per [[exercise-04]] | Enables joining across logs and DSR lookup without plaintext PII |
| `query`, `prompt`, `completion`, `tool_args`, `tool_result` | **Drop by default** | These are the three privacy time-bombs |
| `query_fingerprint` | **Log hash** | Equal queries collide; supports caching/debug without raw text |
| `retrieved_doc_id` | **Hash** | Enough to join to the corpus by `doc_id_hash`; does not expose text |
| `tool_args` fields matching a PII detector | **Redact to placeholder** (`<EMAIL>`, `<PHONE>`, `<ORDER_ID>`) before storing | See [[mod-104]] |
| Error stack traces | **Scrub exception message**, keep type + file:line | Stack messages regularly contain raw user input |
| Debug-level trace of prompt/completion | **Allowed only in `debug` retention class, sampled, with tenant opt-in** | §2.3 |
| Refusal/safety classifier output | **Log category only**, not the triggering span | Avoid re-logging the content that was flagged |

### 2.3 Sampled verbatim capture — the one exception

Every real RAG system eventually needs some verbatim prompt/completion
data: for model eval, for prompt regressions, for incident reconstruction.
The policy allows this *as a sampled, consented, short-retention
exception*, not as a default behaviour.

Rules:

- **Sampling rate:** ≤ 0.1% of production turns, weighted toward
  refusals and errors, or an explicit per-user eval cohort.
- **Consent:** tenants whose turns are eligible for verbatim capture
  are listed in a tenant-level `allow_verbatim_eval` config and the
  tenant admin has acknowledged it in the admin UI. No tenant is in
  this list by default.
- **Retention:** `debug` class — 7 days. After 7 days, only the hashed
  / redacted baseline event remains.
- **Access:** eval + on-call engineers, with each read logged to
  `audit.policy_decision`. Not available to product, analytics, or
  support roles.
- **Redaction in-flight:** the sampled verbatim is still passed through
  the redactor from [[exercise-04]] before it reaches the log store; the
  pipeline does not keep a "true raw" pre-redaction copy.
- **Cross-tenant bar:** sampled verbatim capture is *off* for any
  tenant on a physical-isolation plan from [[exercise-01]].

Any policy that treats verbatim logging as a day-one default fails
this exercise.

### 2.4 Retention and access

| Event class | Retention | Who can read |
|---|---|---|
| `audit.*`              | 1 year (or per-regulation, whichever is longer) | Audit, legal, security IR |
| `security.*`           | 90 days | Security IR, on-call |
| `operational.*`        | 30 days | SRE, on-call |
| `debug.*` (incl. verbatim) | 7 days | Eval + on-call, each access logged |
| Raw prompt/completion anywhere outside `debug.*` | **Not permitted** | — |

Every access to `debug.*` is itself an event in `audit.*`. This is the
single most important access-control rule in the policy: if debug
verbatim is read, someone else can see that it was read.

### 2.5 DSR and incident hooks

The policy must support two workflows without needing to keep raw
prompts:

- **DSR erasure.** A data subject request erases the subject's
  identifiers from the corpus and from all events that reference them.
  Because `principal_id` is a **keyed HMAC** (not plain hash), erasure
  is implemented by rotating or deleting the HMAC key segment for that
  subject — making the hash non-recoverable without reading each log
  line. The policy must state which identifier(s) are used for DSR
  lookup and how the key rotation is coordinated with the retention
  schedule.
- **DSR access.** A subject-access request produces the list of events
  mentioning the subject. The policy must specify that this returns
  metadata (what was queried, when, by which tool) rather than raw
  completion text, unless the subject's own completions happen to be in
  the sampled verbatim window.
- **Incident forensics.** An incident timeline is built from the
  baseline event stream (`request_id`, `tenant_id`, control decisions,
  retrieved `doc_id_hash` set). Raw content is reconstructed only by
  pulling the corresponding sampled verbatim if present, with an audit
  entry for the read.

### 2.6 Worked example — the "a chunk leaked into the prompt" scenario

To make the policy concrete, trace a real incident through it:

1. Tenant A reports an intermittent cross-tenant chunk in a completion.
2. The on-call looks up the `request_id` and reads `rag.retrieval`,
   `rag.query`, and `audit.policy_decision` for that turn.
3. From `retrieved_doc_id_hashes`, they identify which chunk was
   returned. The hashes are stable across the event stream and the
   corpus, so they can cross-reference the originating document
   without reading any raw prompt.
4. If the turn happened to be in the sampled verbatim window, the
   on-call reads the one sampled record, and that access is logged
   to `audit.policy_decision` with the incident ticket ID.
5. Mitigation is applied; the harness from [[exercise-02]] is extended
   with a regression case derived from the chunk's shape (near-duplicate
   pattern), not its raw text.

No step requires the system to hold raw prompts by default.

## 3. Validation steps

Reviewer walks the policy document through the following checks:

1. **Default-drop test.** Raw `query`, `prompt`, `completion`,
   `tool_args`, and `tool_result` fields are dropped by default. Any
   exception is a written sampling rule.
2. **Keyed-hash test.** `principal_id` and other joinable identifiers
   are **keyed HMAC**, not SHA of the raw value. Plain hashes are
   defeated by dictionary attacks on known identifier spaces (email
   addresses, order IDs).
3. **Retention test.** Every event class has an explicit retention,
   a role, and a role-access audit hook. Blank retention for any class
   is a finding.
4. **Verbatim-capture test.** The policy allows ≤ 0.1% sampling for
   eval, with tenant opt-in, 7-day retention, and per-read audit.
   Policies that allow 100% verbatim "for now" fail.
5. **Physical-isolation guard.** Tenants on physical-isolation plans
   are excluded from verbatim sampling.
6. **DSR interoperability.** The policy specifies how erasure is
   implemented over logs (keyed-HMAC rotation, not line-by-line
   grep-and-delete) and how the retention window relates to the DSR
   SLA.
7. **Incident reconstruction without raw text.** The policy's §2.6
   walk-through produces a usable incident timeline from metadata
   alone.
8. **Cross-reference check.** The policy references [[exercise-01]]
   for ACL controls, [[exercise-04]] for the redactor,
   [[mod-104]] for PII detection, and [[mod-110]] for DSR / unlearning.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Event taxonomy with ≥ 6 classes, each labelled | 1 | Sensitivity, retention class, role |
| Default-drop rule on raw prompts/completions/tool_args | 2 | Most commonly broken |
| Field-level matrix with raw / hash / redact / drop per field | 1 | — |
| Keyed HMAC on joinable identifiers (not plain SHA) | 1 | — |
| Sampled-verbatim exception with rate, consent, retention, and access controls | 2 | — |
| Retention table per event class | 1 | — |
| DSR hook — erasure strategy over logs | 1 | Key-rotation rather than record-level delete scales |
| Incident walkthrough reconstructs timeline from metadata alone | 1 | — |

Automatic failures:

- Default-on verbatim logging of prompts or completions.
- Plain SHA (unkeyed) hash on PII identifiers.
- No retention on any event class.
- Verbatim sampling enabled for physical-isolation tenants.

## 5. Common mistakes

- **"We'll redact before log-ship but keep raw in debug."** Debug
  storage *is* a log store, with its own ACL, retention, and backup
  schedule. If debug has raw prompts, raw prompts are retained.
- **Plain SHA on email addresses.** Email space is small and known;
  SHA(email) is trivially reversible with a dictionary. Use a keyed
  HMAC and treat the key as an erasure control.
- **"Log everything, redact later."** The redactor runs in-flight for a
  reason: once raw content is in the log store, every backup, every
  BI export, every shared viewer is a new exposure. Redact before log.
- **Confusing refusal metrics with content logging.** You can measure
  refusal rate without logging the refused content. The content is
  usually what triggered the refusal — exactly what you do not want to
  store.
- **Retention on calendars, not classes.** A policy that says "logs
  retained 90 days" without saying *which* logs invites every team to
  pick the longest number. Retention is per event class.
- **Treating tool-call arguments as "metadata."** `tool_args` carries
  user IDs, order IDs, SKUs, email addresses. It is user content; the
  redactor applies to it.
- **No DSR story for logs.** If you cannot erase a subject from the
  log store within the DSR SLA, your DSR story is a fiction. Keyed-HMAC
  identifiers plus key rotation is the cleanest way; see [[mod-110]].

## 6. References

- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS.
  <https://atlas.mitre.org/> (`AML.T0024 Exfiltration via ML Inference
  API`, `AML.T0020 Data from Local System`).
- NIST AI Risk Management Framework 1.0 — Govern (policies, roles,
  accountability) and Manage (incident response, continual
  improvement).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- Related solutions in this track: [[exercise-01]] per-tenant ACL;
  [[exercise-02]] negative-test harness; [[exercise-04]]
  redact-before-log pipeline; [[exercise-05]] agent-tool identity
  propagation; [[exercise-06]] RAG system card privacy slice; and the
  PII-detection stack in [[mod-104]] and DSR / unlearning workflow in
  [[mod-110]].
