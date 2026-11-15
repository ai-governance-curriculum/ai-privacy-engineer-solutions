# Exercise 01 — Presidio Pipeline and Custom Recognisers — Reference Solution

## 1. Solution overview

This is an **implementation exercise**. The learner is asked to stand up a working
PII detection + redaction pipeline on Microsoft Presidio, extend it with at
least one regex-plus-context custom recogniser and one model-based custom
recogniser, and demonstrate the pipeline on a labelled corpus with
precision/recall numbers computed from the learner's own test set.

The assessable output is a repository containing:

1. A `Dockerfile` (or equivalent lockfile) that pins Presidio Analyzer,
   Presidio Anonymizer, and the spaCy / transformers model versions.
2. A custom `EntityRecognizer` implementation using the Presidio Analyzer
   `EntityRecognizer` / `PatternRecognizer` extension points, registered with
   the `RecognizerRegistry`.
3. A YAML / JSON recogniser-registry configuration so additions can be
   reviewed in a PR without a Python patch.
4. A CLI or HTTP wrapper that accepts text and returns both analyzer results
   (`RecognizerResult[]`) and anonymised output.
5. A `tests/` directory with: (a) a gold-labelled fixture, (b) a metrics
   harness that computes precision, recall, F1 on a per-entity-type basis,
   and (c) regression tests that lock current behaviour.

The reference below walks through the design choices and provides a worked
custom recogniser, a registry config, a metrics script, and the
rubric used to grade a submission. Code snippets are written against the
Presidio public API as documented at
<https://microsoft.github.io/presidio/>.

## 2. Worked implementation

### 2.1 Pipeline architecture

Presidio splits detection from redaction. A production pipeline has four
stages, each of which the learner must be able to point at in their repo:

1. **NLP engine load.** `NlpEngineProvider` loads spaCy or transformers once
   per worker. Pick the model at config time, not request time.
2. **Analyze.** `AnalyzerEngine.analyze(text, language, entities=…,
   score_threshold=…)` returns `RecognizerResult` objects with
   `(entity_type, start, end, score, analysis_explanation)`.
3. **Anonymize.** `AnonymizerEngine.anonymize(text, analyzer_results,
   operators=…)` applies `replace`, `mask`, `hash`, `encrypt`, `redact`, or
   `custom` operators per entity type.
4. **Audit.** Every detection and every redaction is written to a structured
   audit log keyed by request ID, with entity type + offsets + score + the
   recogniser name that fired. This is the artifact a reviewer will inspect
   when asking "did we actually redact?"

> Separation matters because the same Analyzer is reused by (a) inference-time
> prompt scrubbing, (b) training-corpus preprocessing, and (c) stored-log
> redaction. Three callers, one recogniser set, three operator profiles.

### 2.2 Minimal working analyzer with two custom recognisers

```python
# pipeline.py
from presidio_analyzer import (
    AnalyzerEngine,
    PatternRecognizer,
    Pattern,
    RecognizerRegistry,
)
from presidio_analyzer.nlp_engine import NlpEngineProvider
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig


NLP_CONFIG = {
    "nlp_engine_name": "spacy",
    "models": [{"lang_code": "en", "model_name": "en_core_web_lg"}],
}


def build_internal_employee_id_recognizer() -> PatternRecognizer:
    """Regex + context recogniser for a fictional internal employee ID.

    Matches the literal format `EID-` followed by exactly 7 digits, boosted
    when the surrounding context mentions employment identifiers. The context
    list raises the confidence score when a nearby token matches; it does not
    *gate* the match (that is a choice — see rationale in section 2.4).
    """
    patterns = [
        Pattern(
            name="eid_pattern",
            regex=r"\bEID-\d{7}\b",
            score=0.7,
        ),
    ]
    return PatternRecognizer(
        supported_entity="INTERNAL_EMPLOYEE_ID",
        patterns=patterns,
        context=["employee", "staff", "badge", "payroll", "worker"],
        supported_language="en",
    )


def build_analyzer() -> AnalyzerEngine:
    nlp_engine = NlpEngineProvider(nlp_configuration=NLP_CONFIG).create_engine()
    registry = RecognizerRegistry()
    registry.load_predefined_recognizers(nlp_engine=nlp_engine, languages=["en"])
    registry.add_recognizer(build_internal_employee_id_recognizer())
    return AnalyzerEngine(nlp_engine=nlp_engine, registry=registry)


def analyze_and_redact(text: str) -> dict:
    analyzer = build_analyzer()
    anonymizer = AnonymizerEngine()
    results = analyzer.analyze(
        text=text,
        language="en",
        score_threshold=0.5,
    )
    operators = {
        "DEFAULT": OperatorConfig("replace", {"new_value": "<REDACTED>"}),
        "PHONE_NUMBER": OperatorConfig("mask", {
            "chars_to_mask": 7,
            "masking_char": "*",
            "from_end": True,
        }),
        "EMAIL_ADDRESS": OperatorConfig("hash", {"hash_type": "sha256"}),
        "INTERNAL_EMPLOYEE_ID": OperatorConfig("replace", {"new_value": "<EID>"}),
    }
    anonymized = anonymizer.anonymize(
        text=text,
        analyzer_results=results,
        operators=operators,
    )
    return {
        "detections": [r.to_dict() for r in results],
        "anonymized": anonymized.text,
    }
```

### 2.3 Model-based custom recogniser (transformers NER)

Use the Presidio `EntityRecognizer` base class to wrap any ML model that
produces spans. The example below uses a HuggingFace token-classification
pipeline for a domain-specific entity (e.g., `MEDICAL_DEVICE_SERIAL`). Swap
in whatever model the learner's domain requires — the integration surface is
the same.

```python
# ml_recognizer.py
from typing import List, Optional
from presidio_analyzer import EntityRecognizer, RecognizerResult
from presidio_analyzer.nlp_engine import NlpArtifacts


class HFSpanRecognizer(EntityRecognizer):
    def __init__(
        self,
        hf_pipeline,
        supported_entity: str,
        supported_language: str = "en",
        default_score: float = 0.85,
    ):
        super().__init__(
            supported_entities=[supported_entity],
            supported_language=supported_language,
            name=f"hf_{supported_entity.lower()}",
        )
        self._pipe = hf_pipeline
        self._entity = supported_entity
        self._default_score = default_score

    def load(self) -> None:
        # Pipeline already loaded by caller; nothing to do.
        return

    def analyze(
        self,
        text: str,
        entities: List[str],
        nlp_artifacts: Optional[NlpArtifacts] = None,
    ) -> List[RecognizerResult]:
        if self._entity not in entities:
            return []
        results: List[RecognizerResult] = []
        for span in self._pipe(text, aggregation_strategy="simple"):
            score = float(span.get("score", self._default_score))
            results.append(
                RecognizerResult(
                    entity_type=self._entity,
                    start=int(span["start"]),
                    end=int(span["end"]),
                    score=score,
                )
            )
        return results
```

Register it alongside the regex recogniser:

```python
registry.add_recognizer(HFSpanRecognizer(hf_pipeline=hf_pipe,
                                         supported_entity="MEDICAL_DEVICE_SERIAL"))
```

### 2.4 Rationale: non-obvious design choices

1. **Context terms raise score, they do not gate matches.** If context is a
   hard filter, a legitimate `EID-1234567` in a ticket that does not happen
   to say "employee" nearby silently fails. Better: let the regex match,
   then rely on `score_threshold` at the analyzer level so the reviewer
   tunes one knob, not two. (Presidio implements this: context terms
   contribute to the `ContextAwareEnhancer` boost; they are not a gate by
   default.)
2. **Hash, do not replace, for identifiers that are joined elsewhere.**
   Replacing an email with `<EMAIL>` kills downstream deduplication by
   email. Hashing with a keyed SHA-256 preserves join semantics while
   breaking reversibility. Choose the operator from the *downstream use*,
   not from the entity type alone.
3. **Keep the recogniser registry declarative.** Presidio accepts a YAML
   file of recognisers (see `RecognizerRegistry.add_recognizers_from_yaml`).
   Shipping a YAML file beats shipping Python because privacy reviewers can
   review it as data; the deny-list / allow-list review takes an hour
   instead of a day.
4. **Set `score_threshold` globally, not per call.** A per-call threshold
   invites callers to lower it for their use case and silently increase
   false negatives. Lock the threshold in config, and expose a
   `strict`/`relaxed` *profile* instead of a free-form float.
5. **Measure on your own data, not on Presidio demo text.** The default
   recognisers are tuned for US-English generic PII; precision on domain
   data (finance disclosures, EHR notes, dev logs) is almost always worse
   than the demo suggests. The fixture under `tests/gold/` must be drawn
   from the actual target corpus, de-identified or synthetic.

### 2.5 Recogniser registry as data

```yaml
# recognizers.yaml
recognizers:
  - name: InternalEmployeeIDRecognizer
    supported_language: en
    supported_entity: INTERNAL_EMPLOYEE_ID
    patterns:
      - name: eid_pattern
        regex: '\bEID-\d{7}\b'
        score: 0.7
    context:
      - employee
      - staff
      - badge
      - payroll
      - worker

  - name: InternalCaseNumberRecognizer
    supported_language: en
    supported_entity: INTERNAL_CASE_NUMBER
    patterns:
      - name: case_pattern
        regex: '\bCASE-\d{4}-[A-Z]{2}\b'
        score: 0.6
    context:
      - case
      - matter
      - ticket
      - escalation
```

Load with:

```python
registry = RecognizerRegistry()
registry.load_predefined_recognizers(nlp_engine=nlp_engine, languages=["en"])
registry.add_recognizers_from_yaml("recognizers.yaml")
```

### 2.6 Metrics harness

```python
# tests/metrics.py
"""Span-level precision/recall against a gold JSONL fixture.

Each gold row: {"text": "...", "spans": [{"type": "EMAIL_ADDRESS",
"start": 12, "end": 29}, ...]}
"""
import json
from collections import defaultdict
from pipeline import build_analyzer


def spans_overlap(a, b) -> bool:
    return not (a["end"] <= b["start"] or b["end"] <= a["start"])


def score_file(path: str) -> dict:
    analyzer = build_analyzer()
    tp = defaultdict(int)
    fp = defaultdict(int)
    fn = defaultdict(int)
    with open(path) as f:
        for line in f:
            row = json.loads(line)
            preds = [
                {"type": r.entity_type, "start": r.start, "end": r.end}
                for r in analyzer.analyze(row["text"], language="en",
                                          score_threshold=0.5)
            ]
            gold = row["spans"]
            matched_pred = set()
            for g in gold:
                hit = False
                for i, p in enumerate(preds):
                    if (p["type"] == g["type"] and spans_overlap(p, g)
                            and i not in matched_pred):
                        tp[g["type"]] += 1
                        matched_pred.add(i)
                        hit = True
                        break
                if not hit:
                    fn[g["type"]] += 1
            for i, p in enumerate(preds):
                if i not in matched_pred:
                    fp[p["type"]] += 1
    report = {}
    for t in set(list(tp) + list(fp) + list(fn)):
        p = tp[t] / (tp[t] + fp[t]) if (tp[t] + fp[t]) else 0.0
        r = tp[t] / (tp[t] + fn[t]) if (tp[t] + fn[t]) else 0.0
        f1 = 2 * p * r / (p + r) if (p + r) else 0.0
        report[t] = {"precision": p, "recall": r, "f1": f1,
                      "tp": tp[t], "fp": fp[t], "fn": fn[t]}
    return report


if __name__ == "__main__":
    import sys
    print(json.dumps(score_file(sys.argv[1]), indent=2))
```

Report precision, recall, and F1 **per entity type**. Aggregate numbers
hide the entity type that tanks a release.

## 3. Validation steps

A learner (or reviewer) can validate a submission with the following,
without running against production data:

1. **Smoke test.** `python pipeline.py` (or an equivalent entry point) on a
   fixed string must detect at least `EMAIL_ADDRESS`, `PHONE_NUMBER`, and
   the two custom types, and return the anonymised text.
2. **Gold-set metrics.** `python tests/metrics.py tests/gold/corpus.jsonl`
   must print per-entity precision/recall/F1. The submission must include
   an explicit target (e.g., "F1 ≥ 0.85 for `EMAIL_ADDRESS`, `PHONE_NUMBER`,
   `CREDIT_CARD`; F1 ≥ 0.75 for custom types") and must meet it on its own
   fixture.
3. **Recogniser isolation test.** Disable all predefined recognisers and
   confirm the custom recognisers still fire. Then re-enable predefined and
   confirm no regression in the custom F1 (predefined recognisers can and
   do overlap).
4. **Operator contract test.** For every entity type listed in `operators`,
   assert that the output substring at the detected offset does not equal
   the input substring. This catches the common bug where a new entity
   type is added to detection but not to anonymisation.
5. **Threshold sensitivity.** Run the metrics script at `score_threshold`
   of 0.3, 0.5, 0.7. The submission must document which threshold it uses
   in production and why. A pipeline that silently drops matches between
   0.3 and 0.5 without noting the trade-off fails.
6. **Load benchmark.** Measure p50 / p95 latency on 1000 representative
   inputs with a warm worker. The spaCy engine must be loaded once at
   startup, not per request; the test proves this by comparing cold-start
   latency to steady-state latency.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Pipeline runs end-to-end from text to anonymised text with audit log | 2 | Audit log must include `recognizer_name`, `entity_type`, `start`, `end`, `score` |
| At least one regex-plus-context custom recogniser registered | 1 | Via `PatternRecognizer` or YAML |
| At least one model-based custom recogniser registered | 1 | Subclass of `EntityRecognizer` wrapping a model |
| Recogniser registry is declarative (YAML) and loaded at startup | 1 | Not hard-coded in Python |
| Metrics harness computes per-entity precision/recall/F1 on a gold fixture | 2 | Aggregate-only scores do not pass |
| Operators are chosen per downstream use, not per entity type | 1 | `hash` for joinable identifiers, `mask` for user-facing, `replace` for free-text |
| Threshold is set in config with documented rationale | 1 | A magic number in code fails |
| spaCy / transformers model loaded once per worker (not per request) | 1 | Proved by cold-vs-warm benchmark |

Automatic failures:
- Analyzer invoked once per recogniser (indicates misunderstanding of the
  `RecognizerRegistry` pattern).
- Gold fixture is identical to Presidio's demo text (not the learner's own
  data).
- Any entity type in the gold set reports F1 = 0 and is not acknowledged.
- The custom recogniser's `supported_entity` string is already a Presidio
  built-in name (e.g., reusing `PERSON` for a custom type), which corrupts
  the audit trail.

## 5. Common mistakes

- **Treating Presidio as a single `analyze(text)` call.** It is a registry
  of recognisers. Every custom recogniser is a *subclass* that registers
  with the registry; it does not patch the analyzer.
- **Returning the analyzer's full `RecognizerResult` to callers.** The
  `analysis_explanation` field contains the input text positions and
  confidence by rule; leaking it to a public API leaks the detector
  internals. Serialise a minimal projection.
- **Using `replace` for an email that is a join key.** Downstream consumers
  that joined on email cannot join on `<EMAIL>`. Use `hash` with a
  pipeline-scoped key so the same email in two records produces the same
  hash.
- **Catching recogniser exceptions and swallowing them.** A recogniser that
  crashes on a malformed input is a false negative in disguise. Fail the
  request and alert; do not silently return an empty detection list.
- **Forgetting language scoping.** Every recogniser declares
  `supported_language`. A US-English `SSN` recogniser registered as
  `supported_language="en"` will not fire on text analysed with
  `language="en-GB"`; this manifests as "the detector works locally and
  returns nothing in staging."
- **Running the NLP engine inside the request path.** `NlpEngineProvider`
  takes seconds. Load once per worker at process startup.
- **Benchmarking on cherry-picked positive examples.** Precision on a
  positives-only set is uninformative. Score on a mixed corpus whose
  non-PII portion is at least as large as the PII portion.

## 6. References

- Microsoft Presidio documentation (project site and API reference).
  <https://microsoft.github.io/presidio/>
- Microsoft Presidio Analyzer: Customizing Presidio with custom recognizers.
  <https://microsoft.github.io/presidio/analyzer/customizing_analyzer/>
- Microsoft Presidio Anonymizer: Operators.
  <https://microsoft.github.io/presidio/anonymizer/>
- Microsoft Presidio: Recognizer registry YAML format.
  <https://microsoft.github.io/presidio/analyzer/recognizer_registry/>
- NIST AI Risk Management Framework (AI RMF 1.0). Used as the governance
  vocabulary for "manage" and "measure" functions a detection pipeline
  supports. <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — relevant to ML05 "Model theft"
  and the broader supply-chain context for the NER model used by the
  model-based recogniser.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
