# Exercise 06 — ML-BOM Provenance Slice — Reference Solution

## 1. Solution overview

This is an **implementation exercise**. An **ML-BOM** (machine-learning
bill of materials) is a machine-readable record of the components that
went into a model: training and evaluation datasets, pre-trained base
models, model artifacts, hyperparameters, software dependencies, and
the relationships between them. For a privacy engineer, the ML-BOM is
the join point between the datasheet (exercise 01), the consent /
basis store (exercise 02), the minimisation checkpoints (exercise 03),
the vault (exercise 04), and any synthetic-data substitute
(exercise 05). Without it, a request like "which models used this
dataset?" has no authoritative answer.

The exercise asks for a **provenance slice**: not a full SBOM for the
service, but the ML-specific slice that answers four questions:

1. Which datasets (and versions) trained this model?
2. Which model (and version) does this artifact derive from?
3. Who signed the artifact, with what key, and when?
4. What are the known constraints of each component (licence, lawful
   basis, prohibited uses, retention)?

The assessable output has three parts:

1. A **schema choice and justification** — which specification the
   slice uses, and why.
2. A **worked ML-BOM document** for a concrete model, pinned to the
   datasheet example from exercise 01.
3. An **emit-and-verify pipeline** — how the document is produced at
   build time and how a reviewer checks it is valid and complete.

The obvious choice is **CycloneDX** (OWASP, now also an ECMA standard)
with the ML-BOM profile. SPDX 3.0 also carries ML components via its
AI and Dataset profiles; either is acceptable if justified. An
answer that invents a bespoke JSON schema when a standard will do
fails because the point of a BOM is interoperability.

## 2. Worked answer

### 2.1 Specification choice

Use **CycloneDX 1.6+** with the ML-BOM `components` of type
`machine-learning-model` and `data`. Justification:

- CycloneDX ships an explicit machine-learning model type with slots
  for datasets, model parameters, considerations (intended use,
  limitations, ethical considerations, risks), and lineage.
- It composes cleanly with SBOM entries for software dependencies.
- Signing is defined via CycloneDX's JSON-LD / JWS extensions.
- It is an OWASP project and adopted by multiple tool chains, so the
  slice can be consumed by vendor and OSS scanners.

SPDX 3.0 is a valid alternative where the organisation already uses
SPDX 2.x for code — SPDX 3.0 added `software_AIPackage` and
`software_DatasetPackage` profiles. For this reference the worked
example uses CycloneDX; equivalent SPDX output would be noted in the
build pipeline.

### 2.2 Model under provenance — `assistant-ft-v3.1`

Reusing the exercise-01 scenario: the chatbot style-and-resolution
fine-tune `assistant-ft-v3.1`. The relevant lineage is:

- base model: `open-base-7b` (external, commercial licence).
- training corpus: `support-dialogues-eu-v3` (the subject of
  exercise 01).
- evaluation corpus: `support-eval-eu-v3`.
- dependency on the training container image
  `images/trainer:fs-20241002`.
- signed by the ML-platform signing identity
  `release@ml-platform.acme.example`.

### 2.3 CycloneDX ML-BOM slice (abbreviated)

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.6",
  "serialNumber": "urn:uuid:9b0d3a3e-9c2e-4d9e-8f30-7a2c5c3f41a6",
  "version": 1,
  "metadata": {
    "timestamp": "2024-10-03T09:12:00Z",
    "tools": [
      { "vendor": "Acme", "name": "mlbom-emit", "version": "0.4.1" }
    ],
    "component": {
      "bom-ref": "model:assistant-ft-v3.1",
      "type": "machine-learning-model",
      "name": "assistant-ft-v3",
      "version": "3.1.0",
      "supplier": { "name": "Acme Retail SaaS B.V." },
      "modelCard": {
        "modelParameters": {
          "approach": { "type": "supervised" },
          "task": "text-generation-fine-tune",
          "architectureFamily": "transformer-decoder",
          "modelArchitecture": "open-base-7b",
          "datasets": [
            { "ref": "dataset:support-dialogues-eu-v3@3.0.0" },
            { "ref": "dataset:support-eval-eu-v3@3.0.0" }
          ],
          "inputs":  [ { "format": "text/plain" } ],
          "outputs": [ { "format": "text/plain" } ]
        },
        "quantitativeAnalysis": {
          "performanceMetrics": [
            { "type": "exact-match",  "value": "0.41",  "slice": "support-eval-eu-v3 / tier-A" },
            { "type": "bleu",         "value": "0.33",  "slice": "support-eval-eu-v3 / tier-A" }
          ]
        },
        "considerations": {
          "users": ["EU/UK English retail support agents"],
          "useCases": ["agent-assistant style-and-resolution"],
          "technicalLimitations": [
            "Trained on EU/UK English only; performance drops outside this scope.",
            "Style-and-resolution target; must not be used for user-impacting decisions."
          ],
          "performanceTradeoffs": [
            "Latency 180ms median at 4x A10 serving."
          ],
          "ethicalConsiderations": [
            {
              "name": "memorisation-risk",
              "description": "Consent-filtered training mitigates but does not eliminate verbatim recall of prior tickets; canary-style audits are run at release."
            }
          ],
          "fairnessAssessments": [
            {
              "groupAtRisk": "non-native English speakers",
              "benefits":    "resolution speed",
              "harms":       "style polish may mis-translate idiom"
            }
          ],
          "environmentalConsiderations": {
            "properties": [
              { "name": "training_energy_kwh", "value": "<!-- pending-measurement: measured kwh for trainer:fs-20241002 run -->" }
            ]
          }
        }
      },
      "licenses": [
        { "license": { "id": "LicenseRef-Acme-Internal-Only" } }
      ]
    }
  },
  "components": [
    {
      "bom-ref": "dataset:support-dialogues-eu-v3@3.0.0",
      "type": "data",
      "name": "support-dialogues-eu-v3",
      "version": "3.0.0",
      "supplier": { "name": "Acme Retail SaaS B.V." },
      "data": [{
        "type": "dataset",
        "name": "support-dialogues-eu-v3",
        "classification": "personal-data; pseudonymised; sensitive=false",
        "contents": { "attachment": { "contentType": "text/markdown", "content": "See datasheet at datasets/support-dialogues-eu-v3/DATASHEET.md" } },
        "governance": {
          "owners": [{ "organization": { "name": "ml-platform-eu" } }],
          "stewards": [{ "organization": { "name": "privacy-engineering" } }]
        }
      }],
      "properties": [
        { "name": "lawful_basis",      "value": "GDPR Art. 6(1)(a) consent for purpose improve-assistant-quality" },
        { "name": "activity_id",       "value": "assistant-training-eu" },
        { "name": "retention_days",    "value": "730" },
        { "name": "jurisdiction",      "value": "EU" },
        { "name": "pseudonymisation",  "value": "account_id -> account_pid via pseudonymisation service" },
        { "name": "freeze_date",       "value": "2024-09-01" },
        { "name": "preprocess_image_digest", "value": "sha256:f5a91e2..." },
        { "name": "DSR_filter_commit", "value": "sha256:7c03b1a..." }
      ]
    },
    {
      "bom-ref": "dataset:support-eval-eu-v3@3.0.0",
      "type": "data",
      "name": "support-eval-eu-v3",
      "version": "3.0.0",
      "supplier": { "name": "Acme Retail SaaS B.V." },
      "properties": [
        { "name": "lawful_basis",   "value": "GDPR Art. 6(1)(a) consent for purpose improve-assistant-quality" },
        { "name": "activity_id",    "value": "assistant-eval-eu" },
        { "name": "retention_days", "value": "730" },
        { "name": "jurisdiction",   "value": "EU" }
      ]
    },
    {
      "bom-ref": "base-model:open-base-7b@1.2.0",
      "type": "machine-learning-model",
      "name": "open-base-7b",
      "version": "1.2.0",
      "supplier": { "name": "<!-- pending-vendor-review: confirm legal name of base-model publisher at time of use -->" },
      "licenses": [
        { "license": { "id": "<!-- pending-vendor-review: pin the exact licence id (e.g., Apache-2.0 or a custom non-commercial licence) -->" } }
      ]
    },
    {
      "bom-ref": "container:images/trainer:fs-20241002",
      "type": "container",
      "name": "images/trainer",
      "version": "fs-20241002",
      "hashes": [
        { "alg": "SHA-256", "content": "<trainer image digest>" }
      ]
    }
  ],
  "dependencies": [
    {
      "ref": "model:assistant-ft-v3.1",
      "dependsOn": [
        "dataset:support-dialogues-eu-v3@3.0.0",
        "dataset:support-eval-eu-v3@3.0.0",
        "base-model:open-base-7b@1.2.0",
        "container:images/trainer:fs-20241002"
      ]
    }
  ],
  "signature": {
    "algorithm": "ES256",
    "publicKey": "{...jwk...}",
    "value": "{...jws...}"
  }
}
```

Notes on the document:

- `datasets` link the model to the data component via `bom-ref`. One
  pointer, not a copy of the dataset metadata — the dataset's own
  properties are the authoritative source.
- `properties` on the dataset components carry the privacy-engineering
  facts a reviewer actually needs: lawful basis, activity id,
  retention, jurisdiction, pseudonymisation boundary, and the
  `DSR_filter_commit` (the commit of the consent-state snapshot used
  to filter the corpus — the join point to exercise 02).
- `freeze_date` and `preprocess_image_digest` tie the data component
  back to the datasheet from exercise 01.
- Claims that cannot be confirmed from internal records at emit time
  are written as `<!-- pending-review: ... -->` comments (e.g.,
  `pending-measurement`, `pending-vendor-review`). This is
  deliberate — the ML-BOM is accurate about what is known and honest
  about what is pending.
- `signature` is required for release-grade BOMs. Internal pre-release
  BOMs may be unsigned, in which case the pipeline marks them
  `metadata.properties.unsigned=true`.

### 2.4 Emit pipeline implementation

Where the ML-BOM comes from:

- The **training runner** emits a candidate ML-BOM when it finishes a
  training job, reading its own job spec plus:
  - the `MODEL_CARD.md` front matter,
  - the dataset manifests (`datasets/*/manifest.yaml`),
  - the pinned trainer image digest,
  - the base-model artifact id, and
  - the Git commit of the training pipeline.
- A **post-processor** merges downstream information:
  - evaluation metrics from the eval job output,
  - the latest DSR-filter commit used for the corpus,
  - the vault classes and purposes touched during training (none, in
    this scenario).
- The **signer** signs the final document with the ML-platform release
  key. The signing identity is the one that can be revoked
  independently of the training runner (so a compromised runner
  cannot silently ship a signed BOM).
- The signed BOM is written to the release artifact registry at
  `sboms/assistant-ft-v3-mlbom.json` and attached as a provenance
  predicate to the model artifact (e.g., via an in-toto attestation).

Trivial emit sketch:

```python
# tools/mlbom/emit.py
def emit(job_spec: TrainingJobSpec) -> dict:
    datasets = [load_dataset_manifest(d) for d in job_spec.datasets]
    base = load_base_model(job_spec.base_model_ref)
    bom = {
      "bomFormat": "CycloneDX", "specVersion": "1.6",
      "serialNumber": new_urn_uuid(),
      "version": 1,
      "metadata": metadata_block(job_spec),
      "components": components_block(datasets, base, job_spec),
      "dependencies": dependencies_block(job_spec, datasets, base),
    }
    attach_metrics(bom, job_spec.eval_output)
    attach_governance(bom, datasets)      # lawful basis, retention, ...
    return bom
```

### 2.5 Verify pipeline

A separate `tools/mlbom/verify` script runs on every PR that touches
a `sboms/*-mlbom.json` file, and on every release. Checks:

1. **Schema validity.** JSON parses; conforms to the CycloneDX 1.6
   JSON schema; `bom-ref` identifiers are unique and resolvable.
2. **Signature.** `signature.value` verifies against the published
   public key for the release identity; the key chain has not been
   revoked.
3. **Lineage closure.** Every `dependsOn` target exists in the
   `components` array; nothing points into the void.
4. **Dataset property minimum.** Every `type: data` component has
   `lawful_basis`, `activity_id`, `retention_days`, and `jurisdiction`
   properties. Missing properties fail.
5. **Base-model licence.** `license.id` on any
   `machine-learning-model` component is populated, or marked
   a `pending-*` placeholder with a tracking ticket.
6. **Datasheet cross-check.** For each dataset `bom-ref`, pull the
   referenced datasheet and confirm the version / freeze-date match.
7. **Model-card cross-check.** `metadata.component.modelCard.datasets`
   matches the model card's declared training datasets.
8. **No `pending-*` placeholders on release.** On a release build, any
   `pending-*` placeholder fails the verify.
9. **No vault classes leaked.** No `properties[]` carries a raw vault
   class value when the ML-BOM is published externally (should be a
   reference, never a secret).
10. **Replay safety.** The verify tool is deterministic; given the
    same inputs, it reports the same result.

### 2.6 How this slice is used downstream

- A **DSR coordinator** queries ML-BOMs by `dataset:*` references to
  answer "which production models used this dataset?" when a mass
  consent withdrawal or a complaint lands.
- **Procurement / vendor review** checks base-model licensing and
  the signing chain when a third-party model is adopted.
- **Incident response** reads the ML-BOM to scope affected artifacts
  within minutes of a data-source incident.
- **Auditors** use the ML-BOM to cross-check the RoPA, the datasheet,
  and the model card; the ML-BOM is the single machine-readable artifact
  that ties them together.

## 3. Validation steps

1. **Parse test.** The ML-BOM parses as valid CycloneDX 1.6 with
   `cyclonedx validate` (or an equivalent tool).
2. **Signature test.** The signature verifies against the release
   public key; a tampered byte fails verification.
3. **Lineage test.** Rename a dataset in the `components` array.
   Verify fails with a specific dangling-reference message.
4. **Minimum-properties test.** Drop `lawful_basis` from a dataset
   component. Verify fails.
5. **Datasheet cross-check.** Bump the dataset version in the ML-BOM
   without updating the datasheet. Verify fails.
6. **Release-gate test.** Submit an ML-BOM containing a
   `pending-*` placeholder on a release build. Verify fails.
7. **Query test.** Given a set of ML-BOMs, run a query such as
   "all models depending on `dataset:support-dialogues-eu-v3`" and
   confirm the expected models are returned.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Uses a standard spec (CycloneDX or SPDX) with justification | 1 | Bespoke schemas fail |
| Declares the model as `machine-learning-model` and each dataset as a `data` component | 1 | Mixed-into-one fails |
| Dataset components carry lawful basis, activity id, retention, jurisdiction, pseudonymisation boundary | 2 | Missing any one fails this item |
| `freeze_date` / preprocess image digest / DSR-filter commit tied back to the datasheet (exercise 01) and the basis store (exercise 02) | 2 | "Trust me" fails |
| `dependencies` block closes over datasets, base model, trainer image | 1 | Dangling refs fail |
| Signed by a release identity distinct from the training runner | 1 | Shared identity fails |
| Verify pipeline exists and includes signature, schema, lineage, property minimum, release gate | 1 | Missing release gate fails |
| Model-card and ML-BOM cross-check present | 1 | Silent divergence fails |

Automatic failures:

- Inventing a new schema when CycloneDX or SPDX would carry the data.
- Signing the ML-BOM with the same credentials that produced the
  training artifact.
- Omitting the lawful-basis property on a dataset component.
- Shipping an ML-BOM full of `pending-*` placeholders without a
  ticketed resolution plan.

## 5. Common mistakes

- **Treating the ML-BOM as documentation.** The ML-BOM is a
  machine-readable artifact that consuming tools act on; prose
  alongside it is secondary. "The BOM is in the README" fails.
- **Copying dataset metadata into the BOM instead of referencing it.**
  The datasheet is the authoritative source; the BOM references, and
  a reviewer can trace by `bom-ref`.
- **Signing with the runner key.** The runner can be compromised; the
  release signing key must live in an isolated identity so that
  compromise of the training job does not produce legitimately signed
  outputs.
- **Pending placeholders on release.** A `pending-*` marker is a legitimate
  pre-release state; it is a release blocker.
- **Missing `freeze_date`.** Without it, you cannot reproduce the
  corpus or answer "was this subject in training?"
- **Confusing model licence with data licence.** They are different
  components and different clauses. Base-model licences often
  constrain commercial use; dataset licences often constrain
  redistribution.
- **One ML-BOM per project instead of per artifact.** Each model
  artifact is its own component with its own version and its own
  lineage. Collapsing many artifacts into one BOM breaks the "which
  model used this data?" query.
- **Not emitting on evaluation.** Evaluation datasets are personal
  data too; emit lineage for eval, not just training.

## 6. References

- OWASP CycloneDX specification — in particular the Machine Learning
  Bill of Materials (ML-BOM) guidance. <https://cyclonedx.org/>
- OWASP CycloneDX Authoritative Guide to SBOM / ML-BOM. <https://cyclonedx.org/guides/>
- SPDX 3.0 specification, AI and Dataset profiles. <https://spdx.dev/learn/overview/>
- NTIA SBOM minimum elements, 2021 (pre-CycloneDX ML-BOM adoption but
  sets the baseline for BOM expectations). <https://www.ntia.gov/report/2021/minimum-elements-software-bill-materials-sbom>
- SLSA — Supply-chain Levels for Software Artifacts, for how signing
  and attestation attach to BOMs. <https://slsa.dev/>
- in-toto attestations framework, for binding provenance predicates
  (including ML-BOMs) to artifacts. <https://in-toto.io/>
- Regulation (EU) 2016/679 (General Data Protection Regulation),
  Arts. 5, 6, 25, 30, 32. <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- Regulation (EU) 2024/1689 — Artificial Intelligence Act, in
  particular Articles 10 (data governance) and 11 + Annex IV
  (technical documentation) — the ML-BOM is one of the mechanisms
  that satisfies Annex IV items. <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- NIST AI Risk Management Framework (AI RMF 1.0), Govern/Map
  functions and the companion AI RMF Playbook. <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST AI 600-1 — Generative AI Profile, content-provenance and
  model-provenance subcategories.
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
- OWASP Machine Learning Security Top 10 — ML06 AI Supply Chain
  Attacks (an ML-BOM is a primary defence). <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — ML Supply Chain Compromise technique family. <https://atlas.mitre.org/>
- VeriSwarm Trust Center — practitioner example of ML-BOM / dataset
  lineage disclosure. <https://veriswarm.ai/trust>
