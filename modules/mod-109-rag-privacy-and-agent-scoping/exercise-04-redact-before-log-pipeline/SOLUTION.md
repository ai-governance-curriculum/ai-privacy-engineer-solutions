# Exercise 04 — Redact-Before-Log Pipeline — Reference Solution

## 1. Solution overview

This is an **implementation exercise**. The learner must build the
pipeline component that applies the logging policy from
[[exercise-03]]: it accepts a candidate log event, applies
field-level redaction / hashing / drop rules, and emits the sanitised
event to the log store. The pipeline runs *in-flight*, before any
persistent store sees the raw event.

Two forcing properties shape the design:

- **Fail-closed.** If the redactor cannot be applied (model
  unavailable, timeout, panic), the event does not reach the log
  store. A buffered local file with alerting is acceptable; a
  passthrough to the log store is not.
- **Deterministic, keyed identity hashing.** Joinable identifiers
  (`principal_id`, email, order ID) are HMAC-ed with a keyed function,
  not plain-SHA. The key is held in a KMS and is **rotatable per
  tenant** so a data-subject erasure is a key-destruction operation,
  not a line-by-line log delete.

The assessable artifacts:

1. A **redactor library** (Python reference below) exposing a
   `redact(event, schema) -> RedactedEvent` function.
2. A **schema registry** keyed by event name, declaring per-field
   actions: `raw | hash | redact_text | drop | sample_verbatim`.
3. A **shipper integration** that wraps the application's logger so
   that it is impossible to write an event to the store without
   passing through the redactor.
4. A **conformance test suite** that feeds known PII fixtures through
   each event schema and asserts the output contains no raw PII.
5. **Operational metrics** — counters for drops, redactions,
   verbatim samples, and redactor failures.

The reference implementation uses Microsoft Presidio for free-text
PII detection (consistent with [[mod-104]]) and a KMS-backed HMAC for
identity hashing. Either can be swapped; the redactor is designed so
engines are plug-ins and the schema is the contract.

## 2. Worked implementation

### 2.1 Project layout

```
exercise-04-redact-before-log-pipeline/
├── redactor/
│   ├── __init__.py
│   ├── pipeline.py       # redact() entry point
│   ├── actions.py        # per-action implementations
│   ├── schemas.py        # event-schema registry
│   └── kms.py            # HMAC key client (test fake + real KMS)
├── shipper/
│   ├── __init__.py
│   └── adapter.py        # logger wrapper + metrics
├── tests/
│   ├── fixtures/
│   │   ├── rag_query.json
│   │   ├── agent_tool_call.json
│   │   └── pii_corpus.json
│   ├── test_actions.py
│   ├── test_pipeline.py
│   └── test_conformance.py
└── pyproject.toml
```

### 2.2 Schema registry (`redactor/schemas.py`)

```python
# redactor/schemas.py
from __future__ import annotations

from dataclasses import dataclass
from typing import Literal

Action = Literal[
    "raw",             # pass-through
    "hash_keyed",      # HMAC under the tenant key
    "redact_text",     # free-text PII detector + placeholders
    "drop",            # remove from output entirely
    "sample_verbatim", # subject to the sampling rule; see exercise-03 §2.3
]


@dataclass(frozen=True)
class FieldRule:
    action: Action
    detectors: tuple[str, ...] = ()  # e.g., ("EMAIL", "PHONE", "ORDER_ID")


EventSchema = dict[str, FieldRule]

RAG_QUERY: EventSchema = {
    "request_id":       FieldRule("raw"),
    "tenant_id":        FieldRule("raw"),
    "principal_id":     FieldRule("hash_keyed"),
    "query":            FieldRule("drop"),  # fingerprint instead, below
    "query_fingerprint": FieldRule("hash_keyed"),
    "retrieved_doc_ids": FieldRule("hash_keyed"),
    "latency_ms":       FieldRule("raw"),
    "status":           FieldRule("raw"),
    "verbatim":         FieldRule("sample_verbatim"),  # 0.1% eligible
}

AGENT_TOOL_CALL: EventSchema = {
    "request_id":  FieldRule("raw"),
    "tenant_id":   FieldRule("raw"),
    "tool_name":   FieldRule("raw"),
    "tool_version": FieldRule("raw"),
    "arg_schema_hash": FieldRule("raw"),
    "args":        FieldRule("redact_text",
                             detectors=("EMAIL", "PHONE", "US_SSN",
                                        "CREDIT_CARD", "ORDER_ID",
                                        "IBAN", "IP_ADDRESS")),
    "caller_identity": FieldRule("hash_keyed"),
}

SCHEMAS: dict[str, EventSchema] = {
    "rag.query": RAG_QUERY,
    "agent.tool_call": AGENT_TOOL_CALL,
    # ... rag.retrieval, rag.completion, agent.tool_result,
    #     audit.policy_decision from exercise-03 §2.1
}
```

A field that is not in the schema is dropped. This is the key
property: adding a field without declaring it does not reach the log
store. The conformance test (§2.5) rejects any unknown field as a
schema error, not a quiet drop.

### 2.3 Action implementations (`redactor/actions.py`)

```python
# redactor/actions.py
from __future__ import annotations

import hashlib
import hmac
import os
import random
from typing import Any

from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine

_ANALYZER = AnalyzerEngine()
_ANONYMIZER = AnonymizerEngine()

_VERBATIM_RATE = float(os.environ.get("REDACTOR_VERBATIM_RATE", "0.001"))


def hash_keyed(value: Any, *, tenant_id: str, kms) -> str:
    key = kms.get_hmac_key(tenant_id)  # rotatable per tenant; see §2.4
    msg = str(value).encode("utf-8")
    return hmac.new(key, msg, hashlib.sha256).hexdigest()[:32]


def redact_text(value: Any, *, detectors: tuple[str, ...]) -> str:
    text = str(value) if value is not None else ""
    if not text:
        return text
    findings = _ANALYZER.analyze(
        text=text, entities=list(detectors), language="en",
    )
    anonymised = _ANONYMIZER.anonymize(text=text, analyzer_results=findings)
    return anonymised.text


def should_capture_verbatim(tenant_opt_in: bool) -> bool:
    if not tenant_opt_in:
        return False
    return random.random() < _VERBATIM_RATE
```

Notes:

- `hash_keyed` uses the **tenant's** HMAC key. Erasure for a tenant is
  "destroy the tenant key"; erasure for a subject within a tenant uses a
  subject-sub-key variant of the same construction.
- `redact_text` goes through Presidio but is engine-agnostic; swap in
  a managed API or a custom recogniser set from [[mod-104]] without
  changing the pipeline.
- `should_capture_verbatim` enforces the two preconditions from
  [[exercise-03]] §2.3: tenant opt-in, then a probability gate. There
  is no override.

### 2.4 KMS client (`redactor/kms.py`)

```python
# redactor/kms.py
from __future__ import annotations

import os
from dataclasses import dataclass


class KMSError(Exception):
    pass


@dataclass
class KMSClient:
    """Abstracts KMS access.

    In tests, a static map is used. In production, this wraps a cloud
    KMS and caches keys per (tenant_id, version) with a short TTL."""
    _cache: dict[tuple[str, int], bytes]

    def get_hmac_key(self, tenant_id: str) -> bytes:
        version = self._current_version(tenant_id)
        key = self._cache.get((tenant_id, version))
        if key is not None:
            return key
        key = self._fetch(tenant_id, version)
        self._cache[(tenant_id, version)] = key
        return key

    def rotate(self, tenant_id: str) -> int:
        """Mint a new HMAC key for the tenant; returns new version.
        Hashes written with prior versions become unverifiable, which is
        the erasure guarantee."""
        raise NotImplementedError

    def _current_version(self, tenant_id: str) -> int:
        raise NotImplementedError

    def _fetch(self, tenant_id: str, version: int) -> bytes:
        raise NotImplementedError


class FakeKMS(KMSClient):
    """Deterministic test double."""
    def __init__(self, seeds: dict[str, bytes]):
        super().__init__(_cache={})
        self._seeds = seeds
        self._versions = {t: 1 for t in seeds}

    def _current_version(self, tenant_id: str) -> int:
        return self._versions[tenant_id]

    def _fetch(self, tenant_id: str, version: int) -> bytes:
        return self._seeds[tenant_id] + bytes([version])

    def rotate(self, tenant_id: str) -> int:
        self._versions[tenant_id] += 1
        self._cache.clear()
        return self._versions[tenant_id]
```

The abstraction is deliberate: the redactor does not know whether
keys live in AWS KMS, GCP KMS, HashiCorp Vault, or a dedicated
HSM. The erasure guarantee is: once `rotate` is called (or the key
is destroyed), prior-version hashes are not linkable by anyone
who did not independently precompute them.

### 2.5 Pipeline entry point (`redactor/pipeline.py`)

```python
# redactor/pipeline.py
from __future__ import annotations

from dataclasses import dataclass
from typing import Any

from .actions import hash_keyed, redact_text, should_capture_verbatim
from .kms import KMSClient
from .schemas import SCHEMAS, FieldRule


class RedactionError(Exception):
    """Raised when fail-closed invariants would be violated."""


@dataclass(frozen=True)
class RedactedEvent:
    event: str
    tenant_id: str
    fields: dict[str, Any]
    dropped_fields: tuple[str, ...]
    verbatim_captured: bool


def redact(
    event: str,
    payload: dict[str, Any],
    *,
    kms: KMSClient,
    tenant_opt_in_verbatim: bool = False,
) -> RedactedEvent:
    schema = SCHEMAS.get(event)
    if schema is None:
        raise RedactionError(f"unknown event schema: {event}")

    tenant_id = payload.get("tenant_id")
    if not tenant_id:
        raise RedactionError("tenant_id is required on every event")

    out: dict[str, Any] = {}
    dropped: list[str] = []

    for field, rule in schema.items():
        if field not in payload:
            continue
        value = payload[field]
        out[field] = _apply(rule, value, tenant_id=tenant_id, kms=kms)
        if rule.action == "drop":
            dropped.append(field)
            del out[field]

    unknown = set(payload) - set(schema) - {"tenant_id"}
    if unknown:
        # Unknown fields are a schema violation, not a silent drop.
        # Fail-closed: the event does not reach the store until the
        # schema is updated.
        raise RedactionError(f"unknown fields for {event}: {sorted(unknown)}")

    verbatim_captured = False
    verbatim_rule = schema.get("verbatim")
    if verbatim_rule and verbatim_rule.action == "sample_verbatim":
        if should_capture_verbatim(tenant_opt_in_verbatim):
            verbatim_captured = True
            # The verbatim slice goes to `debug.*` with 7d retention;
            # it is redacted before being written. See exercise-03 §2.3.
            raw = payload.get("verbatim", {})
            out["verbatim"] = {
                k: redact_text(v, detectors=("EMAIL", "PHONE", "US_SSN",
                                             "CREDIT_CARD", "IBAN"))
                for k, v in raw.items()
            }

    return RedactedEvent(
        event=event,
        tenant_id=tenant_id,
        fields=out,
        dropped_fields=tuple(dropped),
        verbatim_captured=verbatim_captured,
    )


def _apply(rule: FieldRule, value: Any, *, tenant_id: str, kms: KMSClient):
    if rule.action == "raw":
        return value
    if rule.action == "hash_keyed":
        if isinstance(value, list):
            return [hash_keyed(v, tenant_id=tenant_id, kms=kms) for v in value]
        return hash_keyed(value, tenant_id=tenant_id, kms=kms)
    if rule.action == "redact_text":
        if isinstance(value, dict):
            return {k: redact_text(v, detectors=rule.detectors)
                    for k, v in value.items()}
        return redact_text(value, detectors=rule.detectors)
    if rule.action == "drop":
        return None
    if rule.action == "sample_verbatim":
        # Verbatim handled at the top-level above so sampling is applied
        # once per event, not per field.
        return value
    raise RedactionError(f"unknown action: {rule.action}")
```

The pipeline treats three situations as fail-closed:

- Unknown event name → raise.
- Unknown field on a known event → raise.
- Missing `tenant_id` → raise.

The shipper (§2.6) translates `RedactionError` into "buffer locally,
alert, do not forward."

### 2.6 Shipper integration (`shipper/adapter.py`)

```python
# shipper/adapter.py
from __future__ import annotations

import json
import logging
from typing import Any

from prometheus_client import Counter

from redactor.pipeline import redact, RedactionError
from redactor.kms import KMSClient

LOG = logging.getLogger("rag.shipper")

_REDACT_OK   = Counter("rag_log_redact_ok_total", "Events redacted and shipped",
                       ["event"])
_REDACT_FAIL = Counter("rag_log_redact_fail_total", "Events rejected pre-ship",
                       ["event", "reason"])
_DROP_FIELD  = Counter("rag_log_field_drop_total", "Fields dropped by rule",
                       ["event", "field"])


class RedactingShipper:
    """Wraps the application logger so no caller can bypass redaction."""

    def __init__(self, sink, kms: KMSClient):
        self._sink = sink
        self._kms = kms

    def emit(self, event: str, payload: dict[str, Any], *,
             tenant_opt_in_verbatim: bool = False) -> None:
        try:
            r = redact(event, payload, kms=self._kms,
                       tenant_opt_in_verbatim=tenant_opt_in_verbatim)
        except RedactionError as e:
            _REDACT_FAIL.labels(event=event, reason=type(e).__name__).inc()
            # Fail-closed: buffer locally, alert, DO NOT forward.
            LOG.error("redact_fail event=%s err=%s", event, e)
            self._buffer_locally(event, payload, str(e))
            return
        _REDACT_OK.labels(event=event).inc()
        for f in r.dropped_fields:
            _DROP_FIELD.labels(event=event, field=f).inc()
        self._sink.write(json.dumps({
            "event": r.event,
            "tenant_id": r.tenant_id,
            **r.fields,
            "_verbatim": r.verbatim_captured,
        }))

    def _buffer_locally(self, event: str, payload: dict[str, Any],
                        reason: str) -> None:
        # Buffer to an on-host file under tight ACL; an operator triages.
        # NEVER retry-ship the raw payload to the central store.
        ...
```

Application code never writes to the sink directly. The build enforces
this with a module-import rule / linter (e.g., forbid direct
`logger.info(...)` imports of the central store client; force use of
`RedactingShipper.emit`).

### 2.7 Conformance test (`tests/test_conformance.py`)

```python
# tests/test_conformance.py
import json
import re

import pytest

from redactor.pipeline import redact
from redactor.kms import FakeKMS

PII_FIXTURES = json.load(open("tests/fixtures/pii_corpus.json"))

KMS = FakeKMS(seeds={"tenant-a": b"a" * 32, "tenant-b": b"b" * 32})


@pytest.mark.parametrize("fixture", PII_FIXTURES)
def test_no_raw_pii_leaks(fixture):
    event = fixture["event"]
    payload = fixture["payload"]
    forbidden = fixture["forbidden_substrings"]
    r = redact(event, payload, kms=KMS)
    rendered = json.dumps({"event": r.event, **r.fields})
    for needle in forbidden:
        assert needle not in rendered, f"raw PII leaked: {needle!r}"


def test_unknown_field_fails_closed():
    with pytest.raises(Exception):
        redact("rag.query",
               {"tenant_id": "tenant-a", "request_id": "r1",
                "mystery_blob": "surprise!"},
               kms=KMS)


def test_missing_tenant_fails_closed():
    with pytest.raises(Exception):
        redact("rag.query", {"request_id": "r1"}, kms=KMS)


def test_verbatim_requires_opt_in():
    r = redact("rag.query",
               {"tenant_id": "tenant-a", "request_id": "r1",
                "verbatim": {"prompt": "my email is bob@example.com"}},
               kms=KMS,
               tenant_opt_in_verbatim=False)
    assert not r.verbatim_captured
    assert "verbatim" not in r.fields


def test_hash_is_tenant_scoped():
    r_a = redact("rag.query",
                 {"tenant_id": "tenant-a", "principal_id": "alice"},
                 kms=KMS)
    r_b = redact("rag.query",
                 {"tenant_id": "tenant-b", "principal_id": "alice"},
                 kms=KMS)
    assert r_a.fields["principal_id"] != r_b.fields["principal_id"], \
        "same subject across tenants must not share a joinable hash"
```

### 2.8 Operational metrics

The pipeline exports at minimum:

- `rag_log_redact_ok_total{event}` — events shipped.
- `rag_log_redact_fail_total{event, reason}` — events rejected
  (the alertable metric; non-zero means a schema/invariant regression).
- `rag_log_field_drop_total{event, field}` — fields dropped by
  rule. Useful for spotting "why is the operator asking for this
  field in support tickets" patterns.
- `rag_log_verbatim_captured_total{event}` — count of sampled
  verbatim events. Dashboarded so the sampling rate is never
  silently raised.
- `rag_log_redactor_latency_ms` — histogram; redactor is in the
  hot path.

Alerts:

- Any non-zero rate on `rag_log_redact_fail_total` pages the on-call
  (schema violations must not be silent).
- `rag_log_verbatim_captured_total` rate above 2× the configured
  nominal rate alerts.

## 3. Validation steps

Reviewer runs:

1. `pytest tests/test_conformance.py -q`. All tests pass. The suite
   must include at least one fixture per event class with ≥ 5 PII
   substrings in `forbidden_substrings`.
2. `grep -R "logger.info\|logger.warning\|logger.error" src/` for direct
   logger calls on the central sink from application code. Any direct
   call is a finding; everything must flow through `RedactingShipper`.
3. **Fault injection.** Kill the KMS client briefly and confirm the
   shipper buffers locally and does not forward the event. Restart
   KMS and confirm no retroactive ship of raw payloads.
4. **Schema-change test.** Add a new field to the application code
   without updating the schema; confirm `RedactionError` fires and the
   event does not reach the sink.
5. **Key-rotation test.** Call `KMS.rotate(tenant_id)` and confirm
   that a previously logged `principal_id` hash is no longer
   recoverable from the new key (operational analogue of DSR
   erasure).
6. **Metric sanity.** Confirm counters increment on the expected
   paths and that `rag_log_redact_fail_total` has an alerting rule.
7. **Verbatim audit.** Verify that sampled verbatim records are
   written with 7-day retention metadata and that their access is
   logged to `audit.policy_decision` (hooks into [[exercise-03]]).

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Event-schema registry with per-field actions (`raw | hash_keyed | redact_text | drop | sample_verbatim`) | 2 | Drives the whole pipeline |
| Unknown field → fail-closed | 1 | Most-missed invariant |
| Missing `tenant_id` → fail-closed | 1 | — |
| Keyed HMAC per-tenant, with a rotation path | 2 | Plain SHA fails |
| Free-text redaction integrated (Presidio or equivalent) | 1 | — |
| Verbatim sampling respects rate + tenant opt-in | 1 | — |
| Shipper wraps logger so bypass is syntactically impossible | 1 | Lint rule or module structure counts |
| Metrics + alert on redactor failure | 1 | — |

Automatic failures:

- Any application code path logs to the central sink without the
  redactor.
- Plain SHA on identifiers.
- On KMS unavailability, the pipeline forwards the raw payload.
- Silent drop of unknown fields (should fail-closed).
- Verbatim capture without tenant opt-in.

## 5. Common mistakes

- **Treating the redactor as middleware instead of a gate.**
  Middleware is skippable; a gate is not. Make the shipper the only
  path to the sink.
- **Unkeyed hashes.** `sha256("bob@example.com")` is the same value
  across every system that ever hashed that email. The cross-join
  risk is enormous. Always HMAC with a rotatable key.
- **One global HMAC key.** Rotating a global key revokes everyone's
  linkability at once. Scope keys per tenant (and optionally per
  subject class) so DSR rotation is surgical.
- **Logging the redaction result + the raw.** Debug logs that record
  both the pre- and post-redaction strings defeat the whole exercise.
  The redactor returns one object.
- **No metric on redactor failures.** A silent fallback that forwards
  raw when the redactor is slow is worse than no redactor at all —
  it has the appearance of a control without the function of one.
- **Letting Presidio decide the field list.** The schema decides which
  fields are candidates for free-text redaction. Scanning every field
  as free text is slow and sometimes mangles structured identifiers
  like order IDs that you specifically want to hash.
- **Allowing direct `logger.*` calls to the central sink.** Enforce
  via lint, module layout, or build rule. Social conventions are not
  controls.

## 6. References

- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS.
  <https://atlas.mitre.org/> (`AML.T0024 Exfiltration via ML Inference
  API`).
- NIST AI Risk Management Framework 1.0 — Measure (controls testing)
  and Manage (incident response).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- Microsoft Presidio documentation.
  <https://microsoft.github.io/presidio/>
- Related solutions in this track: [[exercise-01]] per-tenant retrieval
  ACL; [[exercise-02]] negative-test harness; [[exercise-03]] logging
  policy; [[exercise-05]] agent-tool identity propagation;
  [[exercise-06]] RAG system card. The PII-detection engine choices are
  developed in [[mod-104]]; the DSR / unlearning use of key rotation is
  developed in [[mod-110]].
