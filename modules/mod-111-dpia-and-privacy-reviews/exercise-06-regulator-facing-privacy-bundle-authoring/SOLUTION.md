# Exercise 06 — Regulator-Facing Privacy Bundle Authoring — Reference Solution

## 1. Solution overview

This is a **design + authoring exercise**. The learner assembles
the **regulator-facing privacy bundle** for an AI system: the
packet that gets handed (or dropped into a supervisory authority
portal) when a regulator opens a prior-consultation, an
investigation, a complaint response, or a breach inquiry. The
bundle is not a DPIA; it *contains* the DPIA plus every piece of
corroborating evidence a regulator should need to confirm the
DPIA's claims without emailing engineering.

The assessable output is one directory of artefacts plus a cover
index. The learner must:

1. **Define the bundle's shape** — the mandatory manifest, the
   fixed section order, and the file-naming convention so an
   outside reader can navigate without instructions.
2. **Author the cover letter and executive summary** — the short
   document that tells the regulator what the system does, what
   decision they are being asked for (if any), and where to find
   each piece of evidence.
3. **Package the DPIA + evidence slices** from earlier exercises
   ([[exercise-01]] GDPR Art. 35 DPIA, [[exercise-02]] ICO shape,
   [[exercise-03]] NIST PRAM, [[exercise-04]] EDPB Opinion
   28/2024 + CNIL sheets, [[exercise-05]] HIPAA trigger map)
   into one self-contained package with pinned versions and
   cross-references.
4. **Attach the mechanical evidence** — processor register,
   transfer impact assessment, DSR audit sample (from
   [[mod-110]] exercise-07), model and system cards (from
   [[mod-109]] exercise-06), security-incident contact and prior
   notifications, retention schedule.
5. **Record provenance and integrity** — a signed manifest, a
   hash of each file, the version of every cited standard, and a
   named human accountable for the bundle.

The bundle maps to GDPR Art. 36 (prior consultation) and Art.
33/34 (breach notification) as its outer frame, Art. 35 (DPIA)
as the central document, and Art. 30 (records of processing) as
the evidentiary backbone. Non-EU regulators (ICO under UK GDPR,
CNIL under French variations, HHS/OCR under HIPAA, and US state
attorneys general under CPRA/VCDPA/etc.) read the same bundle
with different cover letters.

Why a *bundle* and not just a DPIA: a DPIA alone is an
assessment, not evidence. When a regulator asks "show that the
controls you claimed in §7 of the DPIA actually exist," the
bundle is what answers that without a second round of
discovery. The failure mode this exercise forecloses is the one
where DPIA claims and operational reality drift — and the
drift is discovered during an inquiry.

## 2. Worked answer / implementation

### 2.1 Bundle manifest (fixed shape)

The bundle is a directory with a pinned structure. The `MANIFEST.yaml`
is the source of truth; any file not listed in the manifest is
not in the bundle.

```
privacy-bundle/<system>/<yyyy-mm-dd>/
├── 00-cover/
│   ├── cover-letter.pdf                   # §2.2
│   ├── executive-summary.pdf              # §2.3
│   └── MANIFEST.yaml                      # §2.4
├── 01-system/
│   ├── system-card.pdf                    # pinned build hash
│   ├── model-card.pdf                     # pinned model IDs
│   └── architecture-and-data-flow.pdf     # DFD from [[exercise-01]] §2.3
├── 02-dpia/
│   ├── dpia-article-35.pdf                # from [[exercise-01]]
│   ├── dpia-ico-shape.pdf                 # from [[exercise-02]]
│   ├── nist-pram-worksheet.pdf            # from [[exercise-03]]
│   └── risk-register.csv                  # one row per identified risk
├── 03-legal-basis/
│   ├── legal-basis-assessment.pdf         # Art. 6 + Art. 9 analysis
│   ├── consent-mechanism-evidence.pdf     # if consent is relied on
│   └── lia-legitimate-interest.pdf        # if LI is relied on
├── 04-transfers/
│   ├── transfer-impact-assessment.pdf     # EDPB TIA, per destination
│   ├── sccs-executed.pdf                  # 2021/914 modules, signed
│   └── adequacy-and-supplementary.pdf     # measures per EDPB 01/2020
├── 05-vendors/
│   ├── processor-register.csv             # Art. 28 processors
│   ├── dpa-index.csv                      # one row per signed DPA
│   └── subprocessor-flow-down.csv         # Art. 28(4) evidence
├── 06-rights/
│   ├── dsr-runbook.pdf                    # from [[mod-110]] exercise-01
│   ├── dsr-audit-sample.jsonl             # from [[mod-110]] exercise-07
│   └── art-22-automated-decision-notice.pdf
├── 07-security/
│   ├── security-controls-summary.pdf      # Art. 32 controls
│   ├── incident-response-plan.pdf         # Art. 33 workflow
│   └── prior-breach-notifications.csv     # Art. 33 history (may be empty)
├── 08-sector/                             # present only when sector law bites
│   ├── hipaa-trigger-map.pdf              # from [[exercise-05]]
│   └── other-sector-notes.pdf             # COPPA, GLBA, FERPA, etc.
├── 09-ai-act/                             # present when EU AI Act applies
│   ├── risk-classification.pdf            # unacceptable/high/limited/minimal
│   ├── article-10-data-governance.pdf     # for high-risk
│   └── fria-fundamental-rights-impact.pdf # Art. 27 FRIA where required
├── 10-governance/
│   ├── nist-ai-rmf-crosswalk.pdf          # Govern/Map/Measure/Manage mapping
│   ├── iso-42001-slice.pdf                # from [[exercise-07]]
│   └── edpb-opinion-28-2024-position.pdf  # from [[exercise-04]]
└── signatures/
    ├── MANIFEST.yaml.sig                  # detached signature over manifest
    └── per-file.sha256                    # one hash per file in manifest
```

A reviewer opens `00-cover/cover-letter.pdf`, reads
`00-cover/executive-summary.pdf` next, then uses
`00-cover/MANIFEST.yaml` to navigate directly to whatever they
need. The numeric prefixes are mandatory and must match the
manifest order.

### 2.2 Cover letter (template)

The cover letter is short (≤ 1 page) and contains exactly these
elements, in this order:

1. **Addressee.** Named supervisory authority + case reference
   (e.g., "ICO case ref: ENQ-123456" or "CNIL, service des
   contrôles") *or* the controller's DPO contact as the
   addressee of record when the bundle is filed proactively
   (prior consultation under Art. 36).
2. **Sender.** Controller legal name, registered seat, DPO name
   and contact (Art. 37–39), named author of this bundle, and
   the executive signatory.
3. **One-sentence purpose.** E.g., "This bundle accompanies our
   prior-consultation request under Article 36 GDPR regarding
   the deployment of **`<system>`**, an AI system processing
   **`<category>`** data for **`<purpose>`**."
4. **Legal position.** The controller's view of (a) the lawful
   basis under Art. 6, (b) any Art. 9 condition relied on, (c)
   the applicability of Art. 22 (automated decision-making),
   and (d) whether the EU AI Act is engaged and at what risk
   tier.
5. **What the regulator is being asked for.** Explicit:
   acknowledgement; approval of a mitigation plan; response to
   an inquiry; breach notification under Art. 33. If no
   decision is sought (purely proactive), say so.
6. **Map to the bundle.** A two-column table: "You will find …
   in §`NN`". This is a courtesy index on top of the manifest;
   it is not optional.
7. **Signatory block.** Controller's accountable executive,
   named, dated, signed. The signature is wet or qualified
   electronic (eIDAS QES in EU contexts) — never an unverified
   image.

What the cover letter does **not** contain: arguments,
marketing, apologies, speculative mitigations, or anything not
directly supported by evidence in the bundle. A cover letter
with legal argument but no citations is a liability, not an
asset.

### 2.3 Executive summary (template)

The executive summary is at most four pages. It is written for a
regulator who has 20 minutes before deciding whether to escalate.
Fixed sections:

- **System in plain language** (≤ 200 words): what it does, who
  uses it, what data it touches, what decisions (if any) it
  automates.
- **Lawful basis and necessity** (≤ 200 words): the specific
  Art. 6 condition and, where relevant, the specific Art. 9
  derogation; the alternatives considered and rejected.
- **The highest residual risks** (table, ≤ 5 rows): risk,
  probability, severity, mitigation, residual rating, owner.
  This is a slice of the risk register (§2.5) — the regulator
  looks here first.
- **Mitigations and commitments** (bulleted): the engineering
  controls and policy commitments that bring residual risk to
  acceptable, each pointing to the file in the bundle that
  evidences it (e.g., "end-to-end data minimisation — see
  `01-system/architecture-and-data-flow.pdf` §4 and
  `02-dpia/dpia-article-35.pdf` §7.2").
- **Prior consultation trigger** (≤ 100 words): why the
  controller believes the processing meets (or does not meet)
  the Art. 36(1) threshold (likely to result in a high risk
  absent mitigation). This is written whether the bundle is
  filed proactively or in response to a regulator request.
- **Changes since last filing** (if applicable): a diff against
  the prior bundle version. Explicitly named: controls added,
  controls removed, scope changes, incident history.
- **Contacts and commitments on turnaround.** DPO, legal, the
  named author of this bundle, and the executive signatory —
  with response SLAs on the controller's side.

### 2.4 Manifest (machine-readable)

The manifest is the source of truth; `MANIFEST.yaml` lives at
`00-cover/MANIFEST.yaml`. Every artefact carries its own
checksum; the manifest aggregates and signs them.

```yaml
schema_version: "privacy-bundle/1.0"
bundle_id: "pb_01HK3V6YXM9N4Z7G5F2TQR1BCD"            # ULID
system:
  name: "patient-triage-assistant"
  version: "2.4.7"
  build_hash: "sha256:3b7e…ff20"
controller:
  legal_name: "Example AI Ltd."
  seat: "Dublin, IE"
  dpo:
    name: "Dr. A. O'Neill"
    contact: "dpo@example.com"
author:
  name: "J. Ferguson"
  role: "Privacy engineer"
  email: "jferguson@example.com"
signatory:
  name: "M. Costa"
  role: "Chief Privacy Officer"
  signed_at: "2026-11-27T14:03:00Z"
filing:
  type: "prior-consultation"                          # | breach-notification | inquiry-response | proactive
  authority: "DPC (IE)"
  case_ref: null
standards_pinned:
  gdpr: "Regulation (EU) 2016/679"
  ai_act: "Regulation (EU) 2024/1689"
  edpb_opinion_28_2024: "EDPB Opinion 28/2024 (17 Dec 2024)"
  iso_42001: "ISO/IEC 42001:2023"
  iso_27701: "ISO/IEC 27701:2019"
  nist_ai_rmf: "NIST AI RMF 1.0 (Jan 2023)"
  nist_pram: "NIST PRAM worksheets (current)"
  cnil_ai_sheets: "CNIL AI how-to sheets (2024 edition)"
  hipaa: "45 CFR §§ 160, 162, 164 (as of filing date)"
contents:
  - path: "00-cover/cover-letter.pdf"
    sha256: "…"
    section: "cover"
  - path: "00-cover/executive-summary.pdf"
    sha256: "…"
    section: "cover"
  - path: "02-dpia/dpia-article-35.pdf"
    sha256: "…"
    section: "dpia"
    supersedes: "pb_01HJYAB…/02-dpia/dpia-article-35.pdf"
  # … one entry per file …
integrity:
  manifest_hash: "sha256:…"
  signature_file: "signatures/MANIFEST.yaml.sig"
  signer_key_id: "kms://privacy-keys/signer-2026-q4"
```

Every file that appears in the directory must appear in the
manifest and every manifest row must point at a file, with one
carve-out: the integrity envelope itself — `00-cover/MANIFEST.yaml`,
`signatures/MANIFEST.yaml.sig`, and `signatures/per-file.sha256`
— is **not** listed under `contents:` and is not covered by
`per-file.sha256`. Listing them would be circular: a manifest row
for `MANIFEST.yaml` must contain the hash of the manifest it
lives in, and a row for `MANIFEST.yaml.sig` must be signed into
the document it signs. Instead, the manifest's own integrity is
anchored by `integrity.manifest_hash` (computed over the manifest
with `manifest_hash` and `signature_file` elided — a canonical
"hash-with-holes") and by the detached signature at
`signatures/MANIFEST.yaml.sig` over the finalised manifest bytes;
`signatures/per-file.sha256` is itself a derivative of `contents:`
and is verified by recomputation, not by a self-hash. A mismatch
between `contents:` and the bundle directory (ignoring the three
envelope files) is a bundle-level failure.

### 2.5 Risk register shape

The risk register at `02-dpia/risk-register.csv` is the piece of
the DPIA a regulator reads most carefully. Columns:

| Column | Example | Notes |
|---|---|---|
| `risk_id` | `R-014` | Stable id across bundle versions |
| `title` | "Re-identification of training-set patients via model memorisation" | Short |
| `description` | "If the model memorises rare records, an API user can extract…" | One paragraph |
| `affected_rights` | "Art. 5(1)(c), 5(1)(f), 9" | GDPR articles; add sector cites where relevant |
| `likelihood_before` | `3` (1–5) | Pre-mitigation |
| `severity_before` | `4` (1–5) | Pre-mitigation |
| `mitigations` | "DP-SGD training (ε=4, δ=1e-6); canary eval; prompt-redaction shipper" | Each mitigation points to a file |
| `mitigation_refs` | "`06-rights/dsr-audit-sample.jsonl`;`02-dpia/dpia-article-35.pdf#7.2`" | Hyperlink into bundle |
| `likelihood_after` | `1` | Post-mitigation |
| `severity_after` | `3` | Severity rarely drops to 1 |
| `residual_rating` | `low` | `low` / `medium` / `high` — defined in DPIA §1 |
| `owner` | "ML platform lead" | Role, not person |
| `review_date` | `2027-05-27` | Must be ≤ 6 months out for `high` |
| `status` | `accepted` | `accepted`/`open`/`in-mitigation`/`transferred` |

Any `residual_rating = high` row is the trigger for Art. 36
prior consultation, and must be paired with a specific mitigation
plan and a committed re-review date. "High, accepted" without a
plan is a bundle-level failure.

### 2.6 Transfer impact assessment

The TIA at `04-transfers/transfer-impact-assessment.pdf` is per
destination country, per processor, per category. The CSV lives
at `04-transfers/transfers.csv`:

| Column | Notes |
|---|---|
| `processor` | Name of entity |
| `role` | processor / joint-controller / sub-processor |
| `destination_country` | ISO 3166-1 alpha-2 |
| `categories` | Which rows of §2.2 of the system card |
| `transfer_mechanism` | adequacy-decision / SCCs-2021/914-module-N / BCRs / Art. 49 derogation |
| `supplementary_measures` | "encryption in transit + TEE at rest + split-key"; pointer to `04-transfers/sccs-executed.pdf` |
| `third-country-law-review` | pointer to the memo — EDPB Recommendations 01/2020 §22 |
| `onward_transfer` | yes/no + mechanism |
| `last_reviewed` | date |

The EDPB Recommendations 01/2020 step-by-step — know, identify,
assess, supplement, procedural steps, re-evaluate — is the
structure of the backing memo. A TIA that stops at "SCCs are
executed" is incomplete; the EDPB's position (confirmed post-
Schrems II) is that SCCs are the floor, supplementary measures
make them sufficient.

### 2.7 AI Act slice (when applicable)

If the EU AI Act applies (Regulation (EU) 2024/1689, in force
2024-08-01 with staggered application dates), the bundle adds
`09-ai-act/`. The risk-tier classification file states:

- Whether the system is **prohibited** (Art. 5). If yes, the
  bundle is a cessation notice, not a filing.
- Whether the system is **high-risk** under Art. 6 + Annex III
  (e.g., biometric categorisation, employment, essential
  services, migration, administration of justice). High-risk
  triggers Art. 9 (risk management), Art. 10 (data governance),
  Art. 11 (technical documentation), Art. 13 (transparency),
  Art. 14 (human oversight), Art. 15 (accuracy / robustness /
  cybersecurity).
- Whether Art. 27 **FRIA** is required (deployers that are
  public bodies or entities providing public services, plus
  Annex III systems for creditworthiness and life/health
  insurance pricing). The FRIA is a separate document; its
  availability is recorded here.
- Whether general-purpose AI model obligations (Art. 51–55)
  attach, and whether the model is designated as GPAI with
  systemic risk.

A regulator opening the bundle should see the tier conclusion
and the Art. 10 data-governance slice without searching.

### 2.8 Breach-notification variant

When the bundle is a breach notification under Art. 33 (or
Art. 34 — subject notification — accompanying it), the shape
shifts slightly:

- The cover letter purpose line becomes "This bundle discharges
  the controller's Art. 33 notification obligation for an
  incident first detected at **`<timestamp>`**."
- `07-security/prior-breach-notifications.csv` becomes the
  focus; the specific incident gets its own subdirectory
  `07-security/incidents/<incident-id>/` with the detection
  timeline, scope assessment, affected categories,
  containment steps, remediation plan, and Art. 34 analysis.
- The DPIA is still present because the regulator asks "was
  this incident contemplated?" — the risk register must either
  show the matching risk row or admit it was missed.
- The 72-hour clock under Art. 33(1) is tracked explicitly; a
  bundle filed late carries the delay reason and the mitigation
  for recurrence.

### 2.9 Non-EU adaptations (brief)

The bundle structure is deliberately regulator-neutral. Swap
the cover letter and §03 contents for the relevant regime:

- **UK ICO.** Section 03 becomes UK-GDPR legal basis; the TIA
  cites the UK ICO International Transfer Agreement (IDTA) or
  the UK Addendum to EU SCCs instead of 2021/914.
- **US HIPAA.** The bundle's §08 HIPAA trigger map
  ([[exercise-05]]) is primary; the DPIA is retitled "Privacy
  Impact Assessment" and references the HIPAA Privacy Rule
  (45 CFR Part 164 Subpart E) and Security Rule (Subpart C).
  Breach notification follows 45 CFR §§ 164.400–414.
- **US state (CCPA/CPRA, CTDPA, VCDPA, …).** The DPIA shape
  becomes a "privacy risk assessment" per the statute (CPRA
  Regulations §7152 for California, VCDPA §59.1-580 for
  Virginia). The transfer section becomes the "sale/share"
  opt-out-and-contract analysis.
- **CNIL / French specificities.** French supervisory authority
  expectations are documented in CNIL's AI how-to sheets; the
  EDPB Opinion 28/2024 position ([[exercise-04]]) accompanies
  any filing involving training on web-collected data or
  legitimate-interest claims.

### 2.10 Integrity envelope

The bundle is tamper-evident:

- Each file has a SHA-256 in `signatures/per-file.sha256`.
- `MANIFEST.yaml` aggregates those hashes and adds the manifest
  hash.
- `signatures/MANIFEST.yaml.sig` is a detached signature over
  `MANIFEST.yaml` with a KMS-held key. The key id is in the
  manifest.
- The bundle is immutable after signature; any revision is a new
  bundle with its own `bundle_id`, an explicit `supersedes`
  entry pointing at the old bundle, and a diff in the executive
  summary.

Why this matters: a regulator receiving a modified PDF of the
DPIA has no way to verify it reflects the controller's position
on a given date. The signature + manifest hash lets the
regulator confirm the artefact the controller filed is the
artefact in hand.

## 3. Validation steps

A reviewer runs this checklist before the bundle is filed.

1. **Manifest completeness.** Every file in the directory is in
   `MANIFEST.yaml`; every manifest row resolves to a file.
2. **Hash check.** For every row, the recorded SHA-256 matches
   the file on disk.
3. **Signature check.** `signatures/MANIFEST.yaml.sig` verifies
   against the referenced KMS key.
4. **Version pinning.** Every cited standard (GDPR, AI Act,
   ICO, EDPB, CNIL, NIST AI RMF, NIST PRAM, ISO 42001, ISO
   27701, HIPAA, state statutes) has a specific version /
   publication date in `standards_pinned`.
5. **Cover letter ≤ 1 page, executive summary ≤ 4 pages.**
   Longer cover documents dilute the signal.
6. **Risk-register integrity.** No `residual_rating = high`
   without a mitigation plan and a ≤ 6-month review date. No
   row without an owner role (not a person). No row whose
   `mitigation_refs` resolves to nothing in the bundle.
7. **TIA completeness.** Every processor in
   `05-vendors/processor-register.csv` whose destination
   country lacks adequacy has a TIA row and an SCC (or valid
   alternative) in `04-transfers/`.
8. **DPIA threshold documented.** Even if the controller
   concludes a DPIA was not strictly required, the DPIA file is
   present with the reasoning — regulators ask.
9. **Art. 22 analysis.** If any system output is used for
   decisions "producing legal effects or similarly significant
   effects," the Art. 22 analysis is in `03-legal-basis/`
   (allowed basis + safeguards + human-review mechanism).
10. **AI Act tier.** If the controller is EU-based or placing a
    system on the EU market, `09-ai-act/risk-classification.pdf`
    is present and conclusive (even if the conclusion is
    "minimal risk — no further AI-Act obligations attach").
11. **Sector-law coverage.** If any category in §2.2 of the
    system card is sector-regulated (health, education, child,
    financial), the corresponding `08-sector/` artefact is
    present.
12. **Governance crosswalk.** `10-governance/nist-ai-rmf-crosswalk.pdf`
    maps every control claim to a Govern / Map / Measure /
    Manage function; `10-governance/iso-42001-slice.pdf`
    (from [[exercise-07]]) is attached.
13. **Prior-filing diff present.** If this is not the first
    filing, the executive summary carries a "changes since last
    filing" section pointing at the diff.
14. **Named human accountable.** `signatory` in the manifest is
    an executive; `author` is the privacy engineer; `dpo` is
    the formal controller-side contact. All three are present
    and non-identical.
15. **Clock integrity for breach bundles.** If the filing is
    Art. 33, the detection timestamp, the 72-hour deadline, and
    the actual filing timestamp are in the manifest; any delay
    has a reason and a remediation.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Bundle has fixed shape, manifest, and signature | 2 | §2.1, §2.4, §2.10 — a bundle without an integrity envelope is not a regulator-grade artefact |
| Cover letter + executive summary follow the templates | 1 | §2.2, §2.3 |
| DPIA + evidence slices from [[exercise-01..05]] are pinned and cross-referenced | 2 | §2.1 §02–§08 |
| Risk register columns present and residual-high rows have mitigation plan + ≤ 6-month review | 1 | §2.5 |
| Processor register + TIA per destination with EDPB 01/2020 structure | 1 | §2.6 |
| AI Act slice (tier + Art. 10 + Art. 27 FRIA where applicable) | 1 | §2.7 |
| Governance crosswalk attached ([[exercise-07]] + NIST AI RMF) | 1 | §2.8 wider governance section |
| Named human accountability: signatory, author, DPO | 1 | §2.2 + manifest |

Automatic failures:

- No manifest, or any file not in the manifest.
- No signature on the manifest.
- Any `residual_rating = high` without a mitigation plan and a
  review date.
- Any processor outside the adequacy list without a TIA.
- Cover letter containing legal argument without evidence in
  the bundle backing it.
- Filing that mixes two system versions (bundle must be pinned
  to one `system.version` + `build_hash`).

## 5. Common mistakes

- **Treating the bundle as a stapled DPIA.** The DPIA is
  central, but the mechanical evidence (processor register,
  TIA, DSR audit, security controls summary) is what converts
  the DPIA from a claim into a dossier. Regulators ask for the
  evidence; its absence is read as its absence in production.
- **Legal argument in the cover letter without citations.** A
  controller arguing legitimate interest in the cover letter
  without pointing to an LIA in `03-legal-basis/` has written
  advocacy instead of filing. Regulators read it as
  overreach.
- **Pinning nothing.** A bundle that cites "GDPR" without a
  version, "the EDPB" without an opinion number, and "NIST"
  without a document id is ungradeable. Every citation names
  the document and the date.
- **Residual-high without a plan.** A risk rated `high`
  post-mitigation requires either a mitigation plan (status
  `in-mitigation`) or explicit acceptance at the executive
  level with a review date. "High, accepted, next review: TBC"
  is not a position.
- **Missing the Art. 22 analysis when it bites.** If any model
  output ever drives a decision producing legal effects or
  similarly significant effects, Art. 22 analysis is mandatory
  — even when the controller's position is that Art. 22(1)
  does not apply. Silence is read as oversight.
- **One TIA for all destinations.** The EDPB 01/2020 structure
  is per destination, per processor, per category. A single
  "multi-region TIA" collapses the analysis and misses the
  third-country-law differences that make or break the SCC
  reliance.
- **Confusing the DPIA with the FRIA.** Under the EU AI Act, a
  FRIA (Art. 27) is additional to a DPIA, not a replacement,
  and only certain deployers must run one. A bundle that files
  a DPIA as a FRIA under the AI Act has not discharged the AI
  Act obligation.
- **Filing as "PII-processing is proportionate" without
  showing alternatives.** Necessity under Art. 6(1)(f) and
  proportionality under Art. 5(1)(c) are demonstrated by
  enumerating the data elements considered and dropped, not
  only by listing the ones retained.
- **Treating the bundle as one-shot.** The bundle has a
  version, a signatory, and a review cadence. Filing a bundle
  that was authored two years ago and never revisited is a
  compliance failure in most regimes.
- **Shipping raw PII inside the bundle.** The bundle is an
  evidence package, not a data dump. DSR audit samples
  ([[mod-110]] exercise-07) use pseudonymised identifiers; the
  bundle carries hashes and references, not direct
  identifiers.
- **Omitting the breach history.** `07-security/prior-breach-notifications.csv`
  can be empty, but it must be present. A missing file is
  read as a missing control.
- **Mixing two system versions.** Every artefact in the bundle
  pins to one `system.version`. If the system changed during
  authoring, the author re-pins and re-runs the controls
  summary — the bundle does not describe a moving target.
- **Not signing.** A bundle without a manifest signature is
  structurally the same as a PDF emailed from an unverified
  inbox. The signature is the "we commit to this as filed"
  statement.

## 6. References

Primary standards (versions pinned in `MANIFEST.yaml.standards_pinned`):

- Regulation (EU) 2016/679 — General Data Protection Regulation
  (Articles 5, 6, 9, 22, 28, 30, 32, 33, 34, 35, 36, 37–39, 44–49).
  <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- Regulation (EU) 2024/1689 — Artificial Intelligence Act
  (Articles 5, 6, 9, 10, 11, 13, 14, 15, 27, 51–55; Annex III).
  <https://eur-lex.europa.eu/eli/reg/2024/1689/oj>
- Commission Implementing Decision (EU) 2021/914 — Standard
  Contractual Clauses for transfers to third countries.
  <https://eur-lex.europa.eu/eli/dec_impl/2021/914/oj>
- EDPB Recommendations 01/2020 on measures that supplement
  transfer tools (adopted 2021-06-18).
  <https://edpb.europa.eu/system/files/2021-06/edpb_recommendations_202001vo.2.0_supplementarymeasurestransferstools_en.pdf>
- EDPB Opinion 28/2024 on certain data protection aspects of
  the processing of personal data in the context of AI models
  (17 December 2024). <https://www.edpb.europa.eu/our-work-tools/our-documents/opinion-board-art-64/opinion-282024-certain-data-protection-aspects_en>
- EDPB Guidelines 04/2019 (prior consultation) and 9/2022 (data
  breach notification).
- ICO Guide to the UK GDPR — Data Protection Impact
  Assessments. <https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/data-protection-impact-assessments/>
- CNIL — How to carry out a DPIA + AI how-to sheets (current
  edition at filing date).
  <https://www.cnil.fr/en/ai-how-to-sheets>
- 45 CFR Parts 160, 162, 164 — HIPAA Privacy, Security, and
  Breach Notification Rules.
  <https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C>
- NIST Privacy Risk Assessment Methodology (PRAM) worksheets.
  <https://www.nist.gov/privacy-framework/privacy-risk-assessment-methodology-pram>
- NIST AI Risk Management Framework 1.0 (Govern / Map /
  Measure / Manage). <https://www.nist.gov/itl/ai-risk-management-framework>
- ISO/IEC 42001:2023 — Artificial intelligence management
  system. <https://www.iso.org/standard/81230.html>
- ISO/IEC 27701:2019 — Extension to ISO/IEC 27001 and ISO/IEC
  27002 for privacy information management.
  <https://www.iso.org/standard/71670.html>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS. <https://atlas.mitre.org/>

Related solutions in this track: [[exercise-01]] (Art. 35
DPIA), [[exercise-02]] (ICO DPIA shape), [[exercise-03]] (NIST
PRAM), [[exercise-04]] (EDPB Opinion 28/2024 + CNIL sheets),
[[exercise-05]] (HIPAA trigger map), [[exercise-07]] (ISO 42001
AIMS privacy slice). Cross-references: [[mod-109]] exercise-06
(system card), [[mod-110]] exercise-01 (DSR runbook) and
exercise-07 (DSR audit trail), [[mod-112]] (privacy program and
communication).
