# Exercise 08 — NIST SP 800-226 Release Package Drill — Reference Solution

## 1. Solution overview

This is a **release-engineering drill**. The learner takes the output
of *one* of the earlier module exercises (DP-SGD model, PATE student,
SmartNoise synthetic data, or PipelineDP aggregate release) and
produces a **release package** that maps, line by line, to the
evaluation framework in **NIST Special Publication 800-226,
*Guidelines for Evaluating Differential Privacy Guarantees*** (final,
2025).

SP 800-226 is structured around a short set of evaluation criteria
that an assessor uses to tell whether a DP release is (a) defined
precisely enough to be reviewable at all, (b) implemented soundly, and
(c) operationally sustainable. The release package has one section per
criterion, each anchored to the specific section of 800-226 it answers.

Assessable output: a directory structure plus a `RELEASE.md`
file that a privacy reviewer can walk through in one sitting and mark
each criterion "accept," "accept with remark," or "reject with
remediation."

The reference below assumes the release under review is the DP-SGD
model from Exercise 02; the same structure applies to the other
release types with the mechanism-specific fields swapped.

## 2. Worked implementation

### 2.1 Release-package layout

```
release-dpsgd-classifier-2026-11-15/
├── RELEASE.md                        # the auditor's walkthrough; see §2.2
├── 01-definition/
│   ├── dp-claim.json                 # (ε, δ), neighbour, unit of privacy
│   └── variant-notes.md              # which DP variant used internally (RDP/PRV/zCDP)
├── 02-mechanism/
│   ├── mechanism.md                  # DP-SGD description with citations
│   └── parameters.json               # σ, C, q, T, epochs, batch structure
├── 03-accounting/
│   ├── accountant.json               # accountant name + library version
│   ├── transcript.txt                # training-time accountant log
│   └── reproduction.md               # canary CI pin + comparison with second accountant
├── 04-implementation/
│   ├── training-script.py
│   ├── validator.log                 # Opacus / TF Privacy validator output
│   ├── layer-compat.md               # BatchNorm→GroupNorm substitution notes
│   └── seeds.md                      # seed-handling and determinism policy
├── 05-threat-model/
│   ├── inside-dp-boundary.md         # what DP covers
│   └── outside-dp-boundary.md        # what DP does not cover (OWASP ML Top 10 / ATLAS map)
├── 06-utility/
│   ├── held-out-metrics.md           # accuracy and task-specific metrics
│   └── per-subgroup-metrics.md       # group-wise metrics for fairness review
├── 07-operations/
│   ├── drift-detection.md            # CI canary job
│   ├── incident-response.md          # DP-recall section of IR runbook
│   └── composition-register.md       # what else has shipped from this corpus
└── 08-data-governance/
    ├── ropa-link.md                  # link to the per-activity RoPA row (GDPR Art. 30)
    └── retention.md                  # training-copy retention & deletion tie-in
```

### 2.2 `RELEASE.md` — the auditor's walkthrough

Each section maps to a criterion from NIST SP 800-226 §4 (specifying)
or §5 (evaluating).

#### A. Privacy claim (SP 800-226 §4.1–4.3)

- Mechanism: DP-SGD (Abadi et al. 2016), library Opacus 1.x.
- Claim: **(ε = 8, δ = 1e-5)** under **add/remove one user** at
  **user-level** privacy.
- Scope: end-to-end over the training run and the released weights. No
  claim made for prompt-time behaviour beyond what DP training provides.
- δ ≪ 1/N check: N = 60 000; 1/N ≈ 1.67e-5; δ = 1e-5 < 1/N. ✅
- Variant internal to accounting: subsampled-Gaussian RDP converted to
  (ε, δ) via Opacus PRV accountant.

Anchors: 01-definition/dp-claim.json, 01-definition/variant-notes.md.

#### B. Mechanism precision (SP 800-226 §4.4)

- σ = `<noise_multiplier>`, C = 1.0, q = 256 / 60 000, T =
  `<steps>` steps across 10 epochs.
- Poisson subsampling: implemented by Opacus's `DPDataLoader`;
  verified via `data_loader.sample_rate`.
- Per-sample clipping: confirmed by inspecting `grad_sample` tensors
  at the first few optimizer steps.

Anchors: 02-mechanism/mechanism.md, 02-mechanism/parameters.json.

#### C. Accounting precision (SP 800-226 §4.5, §5.3)

- Accountant: Opacus `PRVAccountant` (`<library_version>`).
- Transcript: full accountant history saved during training.
- Second-opinion recomputation: RDP accountant re-run on the same
  transcript; ε differs by `<x>`. Both values recorded.
- Composition register: this is the first DP release from this
  training corpus; no prior spend to compose with.

Anchors: 03-accounting/.

#### D. Implementation soundness (SP 800-226 §5.1–5.2)

- Library-validator output: zero errors.
- No BatchNorm; GroupNorm used in the architecture.
- Seed handling: training seed recorded; Opacus's internal secure PRNG
  for noise is not deterministic by design (secure source), which is
  documented in 04-implementation/seeds.md.
- Smoke test: with `noise_multiplier = 0` and all other parameters
  held, membership-inference AUC against the training set jumps from
  ≈ 0.52 to `<baseline>`, confirming that DP noise is doing the work
  attributed to it.

Anchors: 04-implementation/.

#### E. Threat model — inside and outside the DP boundary (SP 800-226 §4.6)

- Inside: training pipeline, intermediate gradients, model weights.
- Outside: inference-time prompt injection (OWASP ML Top 10 ML01),
  model extraction (ATLAS AML.T0043), training-data poisoning
  (OWASP ML02 / ATLAS AML.T0020), supply-chain attacks on the DP
  library itself (ATLAS AML.T0010).
- Note that DP does not protect correlated non-training users (SP
  800-226 §4.2); this is called out explicitly.

Anchors: 05-threat-model/.

#### F. Utility report (SP 800-226 §4.7)

- Held-out task metric + confidence interval.
- Per-subgroup metrics (at least for the sensitive features declared
  by the DPIA — see mod-111) with the group definitions anchored to
  the data schema, not to an ad hoc analysis.
- Comparison against a non-DP baseline with the same architecture; the
  utility gap *is* the privacy cost paid.

Anchors: 06-utility/.

#### G. Operations (SP 800-226 §5.4)

- Drift detection: a CI canary job runs DP-SGD on a fixed synthetic
  dataset with pinned hyperparameters; build fails if ε deviates from
  the pin by more than `<tol>`.
- Incident response: "DP recall" section of the IR runbook names the
  decision owner, the notification matrix, and the artifact
  (weights + manifest) to re-issue.
- Composition register: this release is entered in the per-corpus
  budget register; any future release from the same corpus must either
  re-run a joint accountant or re-review the budget ceiling.

Anchors: 07-operations/.

#### H. Data governance tie-in (GDPR Art. 30; mod-111)

- Links to the processing-activity row in the RoPA; the activity row
  names the lawful basis for training and the retention policy.
- Links to the retention record for the training corpus copy.

Anchors: 08-data-governance/.

### 2.3 Release-review checklist (one page, inside `RELEASE.md`)

```
[ ] Claim (A): neighbour, unit, (ε, δ), δ ≪ 1/N check
[ ] Mechanism (B): σ, C, q, T all stated; Poisson sampling verified
[ ] Accounting (C): named accountant + library version; second-opinion value
[ ] Implementation (D): validator clean; no BatchNorm; smoke test attached
[ ] Threat model (E): inside/outside DP boundary explicit; OWASP/ATLAS linked
[ ] Utility (F): held-out metric + subgroup metrics + non-DP baseline gap
[ ] Operations (G): drift canary, IR-recall path, composition register row
[ ] Governance (H): RoPA row, retention entry, DPIA cross-link
```

A release missing any one of these is not ready for review.

## 3. Validation steps

1. **Every section anchored.** Every section in `RELEASE.md` names at
   least one file under `01-…/08-…`. No section is "see above."
2. **Numbers match.** The (ε, δ), σ, C, q, T, N appear identically in
   `RELEASE.md`, `01-definition/dp-claim.json`,
   `02-mechanism/parameters.json`, and `03-accounting/accountant.json`.
3. **Anchor to NIST SP 800-226.** Each section header cites the
   SP 800-226 subsection it answers. A section with no anchor is
   probably ad hoc.
4. **Threat model is not optimistic.** 05-threat-model names threats
   the release does not defend against; at minimum one OWASP ML Top 10
   item and one MITRE ATLAS technique.
5. **Operations section is executable.** The canary CI job referenced
   exists; the IR recall path names an owner.
6. **Governance tie-in is live.** The RoPA row and retention record
   referenced are real documents, not placeholders. If using TBD, the
   release is a draft and must be labelled as such.
7. **Re-review rehearsal.** Hand the package to a colleague who was
   not in the training work. They should be able to complete the
   release-review checklist from the package alone, without asking
   the training engineer any questions.

## 4. Rubric

Score out of 16 — two points per section A–H — with the following
overlay. Below 12/16 → rework.

| Section | Required for full marks |
|---|---|
| A. Claim | Neighbour, unit, (ε, δ), δ ≪ 1/N check all explicit and sourced |
| B. Mechanism | σ, C, q, T all stated; sampling type verified by inspection |
| C. Accounting | Named accountant + library version; a second accountant's value for comparison |
| D. Implementation | Validator clean; smoke test; no DP-incompatible layers |
| E. Threat model | Inside + outside boundary; at least one OWASP ML Top 10 and one ATLAS item named |
| F. Utility | Held-out metric + subgroup metrics + non-DP baseline gap |
| G. Operations | Canary CI pin + IR recall path + composition register row |
| H. Governance | Live RoPA row + retention entry + DPIA cross-link |

Automatic failures:

- Any ε claim without δ.
- Any missing section A–H.
- An operations section that names no owner for the DP recall path.
- A threat model that says "DP covers this" without saying what it
  does not cover.
- Numbers that disagree between `RELEASE.md` and the JSON files.

## 5. Common mistakes

- **Treating the manifest as the release.** A manifest is one file in
  the package. The package is also the transcript, the validator log,
  the smoke test, the canary pin, and the governance links.
- **Reporting ε without the accountant's library version.** The ε is
  tied to the implementation; a bump in library version changes the
  number and invalidates downstream review decisions.
- **Omitting the composition register.** DP releases compose across
  time against the same corpus; a release with no register entry
  silently inflates the real per-user budget.
- **Treating threat modelling as a list of generic ML risks.** The
  boundary section is specific: inside-the-DP-boundary vs.
  outside-the-DP-boundary. Generic risks that fall on the "outside"
  side still belong, but labelled as such.
- **Letting the subgroup utility report slide.** DP-SGD's per-sample
  clipping can shift error onto under-represented groups. The utility
  section is where this must be surfaced; a DPIA (mod-111) uses this
  report downstream.
- **Reusing a prior release's RELEASE.md with search-and-replace.**
  The reviewer's job is to inspect specifics; a near-identical
  RELEASE.md for a different training run is a strong signal of
  template-driven drift.
- **Missing the δ ≪ 1/N check.** 800-226 §3.1 is explicit. A release
  that silently publishes δ ≥ 1/N invites the next audit cycle to
  reject the claim.

## 6. References

- NIST Special Publication 800-226, *Guidelines for Evaluating
  Differential Privacy Guarantees* (final, 2025). This exercise's
  package layout is a point-for-point response to the specification
  and evaluation sections of this document.
  <https://csrc.nist.gov/pubs/sp/800/226/final>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- Abadi, M. et al. *Deep Learning with Differential Privacy.* ACM CCS 2016.
  <https://arxiv.org/abs/1607.00133>
- Opacus privacy accountants documentation.
  <https://opacus.ai/docs/faq>
- TensorFlow Privacy documentation.
  <https://github.com/tensorflow/privacy>
- OWASP Machine Learning Security Top 10 — used in §E to anchor
  non-DP residual risks (model extraction, prompt injection, data
  poisoning).
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — adversarial ML technique taxonomy used in §E.
  <https://atlas.mitre.org/>
- Regulation (EU) 2016/679 (GDPR), Art. 30 (RoPA).
  <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
