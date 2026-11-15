# Exercise 06 — End-to-End AI PII Integration Contract — Reference Solution

## 1. Solution overview

This is a **design-and-implementation exercise**. The learner has stood up
a detection pipeline (exercise 01), chosen a vendor or hybrid model
(exercise 02), extended it across modalities (exercise 03), and chosen
an anonymisation strategy for structured data (exercise 04) under a
legal pathway (exercise 05). The outstanding question is how all of this
connects to the rest of the AI system — trainers, inference services,
RAG retrievers, log pipelines — so that the detection pipeline is the
single, auditable control point and not a decoration.

The assessable output is an **integration contract** that is both a
written specification and an executable artifact set:

1. A written contract (`CONTRACT.md`) between the PII detection /
   redaction service and each of its named callers (training ingest,
   inference prompt scrubber, structured-log redactor, RAG indexing,
   DSR / unlearning workflow). It specifies the request / response
   shape, SLOs, failure modes, taxonomy, and audit requirements.
2. An OpenAPI (or protobuf) schema for the service surface.
3. Reference client implementations (Python and one other language of
   choice) with retries, circuit breakers, and audit-log emission.
4. A conformance test suite that any caller can run against a running
   detection service.
5. A rollout plan with a shadow / canary / full-cut-over sequence and
   defined rollback signals.

The reference below focuses on the parts that most teams get wrong: the
fail-closed semantics, the taxonomy versioning, the audit contract, and
the way detection results feed DSR and training-data governance.

## 2. Worked implementation

### 2.1 Named callers

The contract enumerates callers by name. Each caller gets its own
operator profile (how to redact) and its own SLO.

| Caller | Purpose | Operator profile | SLO (p95) | Fail-closed behaviour |
|---|---|---|---|---|
| Training ingest | Scrub prompts/responses/notes before they reach the training corpus | Hash joinable identifiers; replace free-text PII with placeholder tokens; drop the row if scrubber fails | Minutes (batch) | Drop row, emit metric `training.ingest.pii_drop_total` |
| Inference prompt scrubber | Scrub user inputs before they hit the model | Replace with placeholder, keep offsets for upstream display | ≤ 50 ms added latency | Return "service unavailable" to the caller; do NOT forward raw input to the model |
| Structured-log redactor | Scrub logs before they are shipped to the central log store | Replace free-text PII with placeholders; hash joinable identifiers | ≤ 100 ms added to shipper | Buffer locally, alert; do NOT forward unredacted logs |
| RAG indexing | Scrub documents before they are embedded | Replace with placeholders; keep a side-table mapping doc-id → hash for recall diagnostics | Minutes (batch) | Reject the document, emit metric |
| DSR / unlearning | Identify which rows / documents / logs contain a named subject's identifiers, for erasure | Return locators (offsets, doc-ids) without redacting | ≤ 10 s per subject | Raise on timeout; the DSR workflow retries, it does not continue without a response |

The five callers share **one** detection service, **one** taxonomy, and
**one** audit log. They differ in operator and SLO. This separation is
the entire point of the contract; it prevents each caller from building
its own slightly-different redactor.

### 2.2 Service surface (OpenAPI sketch)

```yaml
# pii-service.openapi.yaml
openapi: 3.1.0
info:
  title: PII Detection & Redaction Service
  version: "1.0.0"
paths:
  /v1/detect:
    post:
      summary: Detect PII in a single asset without redaction
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/DetectRequest'
      responses:
        '200': { content: { application/json: { schema: { $ref: '#/components/schemas/DetectResponse' } } } }
        '400': { description: Malformed request }
        '415': { description: Unsupported media_type }
        '503': { description: Degraded — upstream model unavailable }
  /v1/redact:
    post:
      summary: Detect and redact in one call
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RedactRequest'
      responses:
        '200': { content: { application/json: { schema: { $ref: '#/components/schemas/RedactResponse' } } } }
  /v1/locate:
    post:
      summary: Locate occurrences of specific subject identifiers in an asset (DSR)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LocateRequest'
      responses:
        '200': { content: { application/json: { schema: { $ref: '#/components/schemas/LocateResponse' } } } }
components:
  schemas:
    DetectRequest:
      type: object
      required: [asset_id, media_type, payload, caller, taxonomy_version]
      properties:
        asset_id:            { type: string }
        media_type:          { type: string, enum: [text/plain, image/png, image/jpeg, application/pdf, audio/wav] }
        payload:             { type: string, description: "base64 for non-text; raw text for text/plain" }
        caller:              { type: string, description: "named caller from the contract" }
        taxonomy_version:    { type: string, description: "semver of the taxonomy the caller expects" }
        operator_profile:    { type: string, description: "named profile (training|inference|logs|rag|dsr)" }
        trace_id:            { type: string }
    DetectResponse:
      type: object
      required: [asset_id, taxonomy_version, findings, audit]
      properties:
        asset_id:            { type: string }
        taxonomy_version:    { type: string }
        detector_versions:   { type: object, additionalProperties: { type: string } }
        findings:            { type: array, items: { $ref: '#/components/schemas/Finding' } }
        audit:
          type: object
          required: [request_id, decided_at]
          properties:
            request_id:   { type: string }
            decided_at:   { type: string, format: date-time }
    Finding:
      type: object
      required: [entity_type, score, recognizer]
      properties:
        entity_type: { type: string }
        score:       { type: number, minimum: 0, maximum: 1 }
        recognizer:  { type: string }
        text:        { type: object, properties: { start: { type: integer }, end: { type: integer } } }
        image:       { type: object, properties: { x: { type: integer }, y: { type: integer }, w: { type: integer }, h: { type: integer }, page: { type: integer } } }
        audio:       { type: object, properties: { start_ms: { type: integer }, end_ms: { type: integer } } }
    RedactRequest:
      allOf:
        - $ref: '#/components/schemas/DetectRequest'
    RedactResponse:
      allOf:
        - $ref: '#/components/schemas/DetectResponse'
        - type: object
          required: [redacted_payload]
          properties:
            redacted_payload: { type: string, description: "same encoding as request payload" }
    LocateRequest:
      type: object
      required: [subject_identifiers, asset_id, media_type, payload]
      properties:
        subject_identifiers:
          type: array
          items:
            type: object
            required: [kind, value]
            properties:
              kind:  { type: string, enum: [email, phone, ssn, mrn, employee_id, name, custom] }
              value: { type: string }
    LocateResponse:
      type: object
      required: [asset_id, matches]
      properties:
        asset_id:  { type: string }
        matches:   { type: array, items: { $ref: '#/components/schemas/Finding' } }
```

Non-obvious shape choices:

- **`caller` and `operator_profile` are required.** This routes to the
  right operator set, applies the right SLO, and tags the audit log.
  Missing `caller` returns 400.
- **`taxonomy_version` is a request field.** If the service upgrades the
  taxonomy under the caller, the caller's downstream code can break
  (new entity type, renamed type). The server returns 409 Conflict if
  it cannot serve the requested version and provides the current one.
- **`detector_versions` is in the response.** The audit log records
  which detector (model + hash) decided each finding. When a finding is
  later disputed, the exact decision environment is recoverable.
- **Separate `/v1/locate` for DSR.** Locate does not redact — it returns
  *where* the named subject identifiers appear, so the DSR workflow can
  delete the containing records. Overloading `/v1/detect` with
  "redact by subject" invites ambiguity.

### 2.3 Fail-closed semantics (the single most important clause)

The contract states, verbatim, for each caller:

> **If the detection service returns anything other than a 200 within the
> SLO, the caller MUST NOT forward the raw payload to the downstream
> system.** The specific fallback is `operator_profile`-dependent (drop,
> buffer, 503 to the user, reject document), but *no caller is permitted
> to default to "pass raw text through."*

This clause is the entire reason an engineer can point at the detection
service and say "no unredacted PII reaches training / logs / RAG." The
fallback path must be as well-tested as the happy path; an untested
fail-closed path is a fail-open path in disguise.

### 2.4 Audit contract

Every detection and every redaction emits one structured audit record
to a durable, tamper-evident store (append-only log). Minimum fields:

- `request_id` — ULID / UUIDv7
- `trace_id` — propagated from the caller
- `caller` — one of the named callers
- `operator_profile` — the profile applied
- `asset_id` — the caller's identifier
- `media_type`
- `taxonomy_version`
- `detector_versions` — map from component to version/hash
- `findings_summary` — count by entity type, min/median/max score, NOT
  the excerpt
- `outcome` — `detected_only`, `redacted`, `failed_open_blocked`,
  `failed_closed_dropped`
- `decided_at` — ISO 8601 timestamp

The audit log **does not** contain the raw payload or the full excerpt
list. It is the index you query when you need to prove "did this asset
pass through the scrubber?" Not "what did the scrubber see?"

### 2.5 Taxonomy versioning

The taxonomy is a separately versioned artifact (`taxonomy.yaml`) with
semver:

- MAJOR bump: entity type removed or renamed, or detection behaviour
  changes meaning (e.g., `EMAIL_ADDRESS` becomes `CONTACT_EMAIL` with a
  broader definition).
- MINOR bump: new entity type added.
- PATCH bump: recogniser weights / thresholds changed; taxonomy entries
  unchanged.

The service exposes `GET /v1/taxonomy` with the current and a history of
supported versions, and `POST /v1/detect` accepts a `taxonomy_version` to
pin against. Callers upgrade at their own cadence, with a documented
support window (typical: n and n-1 MAJOR versions; n and n-2 MINOR).

### 2.6 DSR / unlearning integration

The `/v1/locate` endpoint powers the subject-resolution step of the DSR
workflow (see mod-110). The learner's contract must explicitly address:

- **Input identifiers come from the subject-identifier registry**, not
  from the user. A DSR worker does not accept a free-text name; it
  resolves the request to the canonical identifiers (`email`, `phone`,
  `mrn`, `employee_id`, and any custom identifiers maintained in the
  privacy register) and passes those.
- **Locate returns findings keyed to asset locators**, which the DSR
  fanout uses to delete the specific row, blur the specific image
  region, or mark the specific doc chunk for re-embedding.
- **Negative result is not proof of absence.** `/v1/locate` is bounded
  by what the detector can see. The DSR workflow still needs the
  deterministic join on direct identifiers in structured stores; it
  uses `/v1/locate` only for the stores that do not have a canonical
  index on the subject identifier (free-text logs, document stores,
  vector indexes).

### 2.7 Reference Python client

```python
# pii_client.py
import json, time, uuid
from typing import Optional
import httpx


class PIIServiceError(Exception):
    pass


class PIIClient:
    def __init__(self, base_url: str, caller: str, operator_profile: str,
                 taxonomy_version: str, timeout_s: float = 5.0):
        self._c = httpx.Client(base_url=base_url, timeout=timeout_s)
        self._caller = caller
        self._profile = operator_profile
        self._taxonomy = taxonomy_version

    def redact(self, asset_id: str, media_type: str, payload: str,
               trace_id: Optional[str] = None) -> dict:
        body = {
            "asset_id": asset_id,
            "media_type": media_type,
            "payload": payload,
            "caller": self._caller,
            "operator_profile": self._profile,
            "taxonomy_version": self._taxonomy,
            "trace_id": trace_id or str(uuid.uuid4()),
        }
        for attempt in range(3):
            try:
                r = self._c.post("/v1/redact", json=body)
            except httpx.TimeoutException:
                if attempt == 2:
                    raise PIIServiceError("timeout after retries")
                time.sleep(0.2 * (attempt + 1))
                continue
            if r.status_code == 200:
                return r.json()
            if r.status_code == 409:
                raise PIIServiceError(
                    f"taxonomy mismatch: requested {self._taxonomy}, "
                    f"server supports {r.json().get('taxonomy_version')}")
            if r.status_code >= 500 and attempt < 2:
                time.sleep(0.2 * (attempt + 1))
                continue
            raise PIIServiceError(f"status {r.status_code}: {r.text}")
        raise PIIServiceError("retries exhausted")
```

Points for the review:

- The client enforces `caller`, `operator_profile`, and
  `taxonomy_version` at construction time. A caller cannot forget them.
- The client retries idempotent 5xx / timeout, not 4xx. A 400 is a bug
  in the caller; retrying hides it.
- The client raises on taxonomy mismatch rather than falling back to
  the server's version silently. Silent fall-back is how major taxonomy
  renames corrupt the audit trail.
- There is no "ignore errors and return the raw payload." The caller's
  fail-closed code path owns that decision, not the client.

### 2.8 Conformance test suite (caller-side)

Any caller runs the same suite against a running service before being
onboarded:

1. **200 happy path per media_type** returns a response that validates
   against the OpenAPI schema.
2. **Fail-closed test.** Point the client at a sink that returns 503.
   Assert the caller's fallback path is taken; assert the raw payload
   never reaches the downstream system.
3. **Taxonomy mismatch test.** Pin a nonexistent taxonomy version.
   Assert a 409 is raised and no caller fallback assumes the server
   version silently.
4. **Caller tagging test.** Make N calls with the caller's identifier.
   Query the audit log. Assert the count matches and every record is
   tagged with this caller.
5. **DSR idempotency test.** Call `/v1/locate` twice with the same
   subject identifiers on the same asset. Assert the results are
   identical and that two audit records exist, both tagged with
   `caller=dsr`.
6. **Latency budget test.** p95 added latency is below the caller's SLO
   on a warm pool.

### 2.9 Rollout plan

1. **Shadow.** The service is deployed and receives a mirror of
   production traffic with no downstream effect. Compare detection
   coverage to the pre-existing scrubber (or baseline); investigate any
   regression.
2. **Canary.** One caller (typically the structured-log redactor, which
   has the clearest fail-closed path) is moved behind the service for
   a small percentage of traffic. Monitor the audit log and the
   operator metrics.
3. **Ramp.** Increase the percentage; then move the next caller behind
   the service.
4. **Cut-over.** All callers go through the service. The old scrubbers
   are kept running for a week in "verify-only" mode to catch
   regressions, then removed.
5. **Rollback signals.** p95 SLO breach beyond 5 minutes, elevation of
   `failed_closed_dropped` above the agreed budget, or a conformance-
   test failure in production triggers rollback to the previous
   scrubber.

## 3. Validation steps

1. **Schema validation.** The OpenAPI document parses with a validator
   (`openapi-spec-validator`) and every response from the reference
   service validates against the schema.
2. **Caller enumeration.** The contract lists each named caller with
   operator profile, SLO, and fail-closed behaviour.
3. **Fail-closed proof.** For each caller, there is a test that forces a
   5xx / timeout and asserts no raw payload reaches the downstream
   system. "By inspection" is not acceptable.
4. **Audit completeness.** A sampled percentage of requests produce an
   audit record that includes `caller`, `operator_profile`,
   `taxonomy_version`, `detector_versions`, `outcome`.
5. **Taxonomy pinning.** The service rejects a request that pins an
   unsupported taxonomy version with 409 and does not auto-upgrade.
6. **DSR integration.** A seeded-subject locate on a mixed corpus
   returns the expected locators for every modality that contains the
   subject; downstream deletion covers each returned locator.
7. **Rollout dry-run.** The shadow → canary → cut-over sequence is
   rehearsed in staging with at least one injected failure per stage,
   and the rollback signal fires within the SLO.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Named callers enumerated with operator profile, SLO, and fail-closed behaviour | 2 | "All callers behave the same" fails |
| Schema (OpenAPI / protobuf) committed and validated | 1 | Pseudocode schemas fail |
| Fail-closed behaviour is a tested code path, not prose | 2 | Must include a test per caller |
| Audit contract includes `caller`, `operator_profile`, `taxonomy_version`, `detector_versions`, `outcome` | 1 | Missing `detector_versions` is a common gap |
| Taxonomy versioned (semver) with request-side pinning and 409 on mismatch | 1 | Silent upgrade fails |
| DSR / unlearning integration via a separate `locate` endpoint | 1 | Overloaded detect fails |
| Reference client enforces caller, profile, taxonomy at construction time | 1 | "Caller fills in optional field" is a design smell |
| Rollout plan with shadow / canary / cut-over and named rollback signals | 1 | Big-bang rollout fails |

Automatic failures:
- Any caller permitted to default to "pass raw text through" on
  detection failure.
- No `caller` field on the request or no `caller` in the audit log —
  the control point is unobservable.
- A single "detect-and-redact" endpoint overloaded to serve DSR
  (locate) semantics, with no separation of the operator intent.
- Taxonomy embedded in the service with no version history, no pinning,
  and no deprecation window.

## 5. Common mistakes

- **One integration pattern for all callers.** Training ingest and
  inference prompt scrubbing have different SLOs, different fail-closed
  behaviours, and different operator profiles. A one-size-fits-all
  contract either breaks latency for inference or breaks durability for
  training.
- **Fail-open by omission.** The caller's happy path works, the
  exception handler logs and continues. This is fail-open; the
  downstream system receives unredacted PII. Every fail-closed path must
  have a dedicated test.
- **Audit log contains the excerpt.** The audit log is a decision index,
  not a copy of the data. Excerpt-in-audit expands the PII blast radius
  and complicates DSR (now you have to erase the audit log too).
- **Taxonomy mutations without a version bump.** Renaming
  `EMAIL_ADDRESS` to `EMAIL` breaks every downstream consumer's
  per-type handling. Semver the taxonomy; expose pinning on the
  request.
- **Forgetting DSR.** Many detection services expose only `detect` and
  `redact`. Locating a subject's data across stores at erasure time is
  a different operation with different inputs (named identifiers, not a
  blob of text). Build it as its own endpoint.
- **Mixing detection versions silently.** If the service runs
  model-v3.2 but the audit says the server is "model-v3," the dispute
  is unresolvable later. Record exact versions.
- **Shipping to production without a rollback signal.** "We will watch
  the dashboard" is not a rollback signal; it is a wish. Name the
  specific metric, threshold, and duration that trigger rollback.
- **Using the detection pipeline as the only control.** Detection is
  probabilistic. Combine it with deterministic controls — direct-
  identifier suppression, access control, retention, egress control —
  per the broader program (mod-111, mod-112).

## 6. References

- OpenAPI Specification 3.1. <https://spec.openapis.org/oas/latest.html>
- NIST AI Risk Management Framework (AI RMF 1.0) — specifically the
  "MAP," "MEASURE," and "MANAGE" functions, which frame the audit,
  SLO, and incident response clauses of the contract.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — relevant to data-leak risks
  the contract is designed to constrain, and to the dependency risk of
  the detector as part of the AI supply chain.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — adversary tactics the fail-closed clause is intended
  to defeat (e.g., exfiltration via prompt echo, training-data
  extraction). <https://atlas.mitre.org/>
- Microsoft Presidio — the detection engine assumed by the earlier
  exercises in this module.
  <https://microsoft.github.io/presidio/>
