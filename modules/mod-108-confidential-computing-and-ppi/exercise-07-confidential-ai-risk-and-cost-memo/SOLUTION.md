# Exercise 07 — Confidential AI Risk and Cost Memo — Reference Solution

## 1. Solution overview

This is a **design / memo exercise**. The learner synthesises
everything in the module into a short document a programme owner
could take to a steering committee: *for a given AI workload,
which privacy-preserving inference (PPI) pattern — TEE, HE, MPC,
or hybrid — is the right choice, what risks does it close, what
risks does it leave open, and what does it cost?*

Assessable artefacts:

1. **Workload brief** — 3-5 sentences describing the workload,
   data classification, counterparty structure, regulatory
   frame, and the single privacy question the memo answers.
2. **Pattern-selection matrix** — the four candidate patterns
   (TEE, HE, MPC, hybrid TEE+GPU-CC) scored against five
   dimensions: threat coverage, performance, maturity /
   operational burden, cost, flexibility.
3. **Recommendation** — one chosen pattern, two sentences of
   rationale, and a one-sentence rejected alternative per
   candidate.
4. **Residual-risk register** — the threats the chosen pattern
   does *not* close, with compensating controls and owners.
5. **Cost estimate** — capex (hardware class), opex (per-request
   latency / bandwidth / compute cost multiplier), and
   programme cost (hiring, vendor contracts, audit).
6. **NIST AI RMF hook** — one row per RMF function tying this
   memo into the organisation's risk register.

The exercise is **not** about picking a winner in the abstract.
It is about forcing the learner to connect the module's
mechanisms (TEE, HE, MPC) to a specific workload and to make
every tradeoff explicit.

## 2. Worked answer

### 2.1 Workload briefs (worked)

Three canonical shapes. The learner should pick (or be given) one;
the memo plugs the same template around whichever workload applies.

| Scenario | Data | Counterparties | Regulatory frame | The question the memo answers |
|---|---|---|---|---|
| **A. Hosted LLM inference for enterprise customers** | Customer prompts (contain PII / IP) + proprietary weights | Cloud provider operates the fleet; enterprise is a customer; no second data owner | GDPR Art. 28 processor obligations; sector-specific if regulated (HIPAA / PCI-DSS) | Can the cloud provider be contractually and technically kept out of the plaintext path? |
| **B. Cross-silo risk score between two banks** | Each bank's customer features; neither will share plaintext; a joint model computes a shared score | Two peer institutions; a neutral protocol operator | GDPR, banking secrecy, intra-group transfer rules | Can the two parties compute a joint score without either seeing the other's features? |
| **C. On-device health model with cloud fallback** | Medical readings; device is user-owned; cloud fallback is operated by a third party | User; on-device runtime; cloud operator | HIPAA (US) or GDPR Art. 9 (EU); strict purpose limitation | How is the cloud fallback kept non-targetable and transparent? |

The remaining sections work Scenario A as the running example;
the template applies to the others with minor edits.

### 2.2 Pattern-selection matrix (Scenario A worked)

Scoring: ✓✓ strong fit, ✓ partial, — not applicable, ✗ blocked.
Use the module exercises as ground truth, not vendor collateral.

| Dimension | TEE (SEV-SNP / TDX) [[exercise-02-amd-sev-snp-attestation-walkthrough]] | TEE + GPU CC mode [[exercise-03-nvidia-h100-confidential-inference]] | HE (CKKS / TenSEAL) [[exercise-05-tenseal-he-inference-on-a-small-model]] | MPC (CrypTen) [[exercise-06-crypten-mpc-inference-walkthrough]] |
|---|---|---|---|---|
| Covers "operator sees plaintext" | ✓✓ | ✓✓ | ✓✓ | ✓✓ (if both sides are honest) |
| Covers weights-at-rest | ✓✓ (keys released only after attestation) | ✓✓ | — (server has plaintext weights) | ✓✓ (secret-shared) |
| Covers output leakage / MIA / model-inversion (`AML.T0024`) | ✗ | ✗ | ✗ | ✗ |
| Covers model poisoning (`AML.T0010`) | ✗ | ✗ | ✗ | ✗ |
| Works for GPT-class LLM inference | ✓ (CPU only is impractical) | ✓✓ (H100 CC mode is the practical path) | ✗ (CKKS doesn't scale to that depth) | ✗ (round complexity explodes) |
| Works for small classifier inference | ✓✓ | ✓✓ | ✓ | ✓ |
| Latency vs. plaintext | ~1× + attestation handshake | PCIe-encryption tax; measure per workload (see note below) | 10–100× slower; depends on depth | Communication-bound; ≥ 1 round per non-linearity |
| Bandwidth vs. plaintext | ~1× | ~1× | 10–1000× (ciphertext size) | multi-round, 10–100× bytes |
| Hardware root of trust required | Yes (AMD/Intel) | Yes (AMD/Intel **and** NVIDIA) | No | No |
| Maturity | Production-grade from major cloud providers | Newer; H100 CC single-tenant only | Research-to-pilot | Research-grade |
| Operational burden | Attestation verifier + measurement allow-list + rotation | As TEE + GPU VBIOS + driver allow-list | Parameter management; HE standard rotation | Two production nodes + protocol operator |
| Fits a one-party cloud model | ✓✓ | ✓✓ | ✓✓ | ✗ (needs two real parties) |
| Fits a two-party cross-silo model | ✓ | ✓ | ✓ (client-server pattern) | ✓✓ |

Do not quote a specific "X % overhead" figure unless you can cite
either (a) a learner-run benchmark (as in exercises 5 and 6) or
(b) a published vendor benchmark whose workload matches yours.
<!-- citation-pending: cite the specific NVIDIA H100 CC-mode overhead benchmark that matches the learner's inference shape before promising a number in the memo -->

### 2.3 Recommendation (Scenario A worked)

**Choose TEE + GPU CC-mode** (AMD SEV-SNP or Intel TDX host, H100
in Confidential Computing mode) as the primary control.

Rationale (two sentences):

> The workload is GPT-class inference under a GDPR processor
> relationship, where the operator must be verifiably out of the
> plaintext path but ordinary PyTorch models and GPU kernels must
> keep working. Pure HE and MPC are structurally mismatched at
> this model size; a TEE + GPU CC pair closes the operator-in-the-
> middle threat and chains to published hardware attestations we
> can verify, at a bandwidth/latency cost small enough for
> batched inference.

Rejected alternatives (one sentence each):

- **TEE without GPU CC** — the H100 can carry activations and
  weights in plaintext over PCIe and in HBM; without CC mode the
  operator is still in the TCB at the GPU boundary.
- **HE (CKKS)** — multiplicative depth of a modern transformer is
  not addressable with current CKKS parameters at interactive
  latency.
- **MPC (CrypTen)** — there is only one data party (the customer)
  and one model party (the provider), but MPC's round complexity
  and semi-honest security model don't fit a hosted-LLM
  serving loop.

### 2.4 Residual-risk register

Risks the chosen pattern does *not* close. These must be covered
by other controls; this is the "what still has to be true" list.

| # | Residual risk | Why the pattern doesn't close it | Compensating control | Owner |
|---|---|---|---|---|
| R1 | Output leakage, MIA, model-inversion (`AML.T0024`) | TEE/GPU CC protects data in use, not model behaviour | Output DP + rate limiting + policy filters [[mod-105-memorisation-extraction-and-mia]] | ML privacy team |
| R2 | Model poisoning upstream (`AML.T0010`) | The enclave happily runs a backdoored model | Signed training provenance (SLSA / in-toto), reproducible builds | Model ops |
| R3 | Compromised attestation verifier operator | Treating NRAS / cloud verifier as always-trusted expands TCB | Local verification; log verdict hashes; cross-check [[exercise-03-nvidia-h100-confidential-inference]] | Security engineering |
| R4 | Side channels (Spectre-family, PCIe timing, power) | Documented out-of-scope for current TEE and GPU CC implementations | Microcode floor, SMT off per vendor guidance, dedicated hosts for high-sensitivity workloads | Platform engineering |
| R5 | Vulnerability in attested guest image | Attestation proves *which* image, not that it's bug-free | CVE scanning, measurement rotation, minimal image | Platform engineering |
| R6 | Rollback of sealed state | TEE attestation is per-boot/per-session, not monotonic | Monotonic counters, sealed state with versioned KEKs | Platform engineering |
| R7 | DP budget exhausting over time (if output DP is used) | Not addressed by any CC primitive | DP accountant with budget alerting, re-training schedule | ML privacy team |
| R8 | Legal / jurisdictional transfer under GDPR Art. 46 | TEE doesn't change who the data controller is | Data-processing agreement, SCCs, DPIA | Legal / Privacy office |
| R9 | Supply chain for NVIDIA/AMD/Intel firmware | TCB includes vendor firmware | Pin ARK fingerprint, pin NVIDIA cert chain, monitor vendor security advisories | Security engineering |
| R10 | Over-reliance on vendor marketing for threat model | Vendor docs sometimes conflate "secure" with "scope of this release" | Keep [[exercise-01-tee-trust-model-and-threat-map]] on file; refresh when vendor docs update | Security engineering |

### 2.5 Cost estimate

Costs come in three buckets. Numbers are placeholders in this
reference; the learner's memo must either plug in cloud-pricing
pages or mark the gap with `<!-- citation-pending: ... -->`.

- **Capex-ish (compute class / instance type).** CC-mode hosts
  are priced at a premium vs. standard VMs on all major clouds;
  the exact premium depends on provider and region. The memo
  should quote the official price page and the SKU, not a
  "ballpark."
  <!-- citation-pending: current cloud-provider list price for SEV-SNP/TDX hosts with H100 CC-mode in the target region -->
- **Opex (performance cost).** Two sub-costs:
  (a) **Attestation handshake per cold start / per KEK rotation.**
      This is a one-time or periodic cost (seconds, typically).
  (b) **PCIe encryption tax during steady-state inference.**
      Published by NVIDIA for H100; depends on workload shape.
      Measure on the actual workload before committing to a
      specific number
      <!-- citation-pending: NVIDIA-published H100 CC-mode overhead figures for the specific inference-batch-size regime the memo targets -->.
- **Programme cost.**
  - One-time: hiring / training for attestation verifier
    ownership; vendor-contract review for the attestation
    service; third-party audit of the verifier code.
  - Recurring: measurement allow-list maintenance (every patch
    cycle); microcode-floor tracking; post-CVE rotation.

A short cost table the memo can carry forward:

| Line item | Direction | Magnitude | Cadence |
|---|---|---|---|
| CC-enabled host SKU premium | ↑ | vendor-published | continuous |
| PCIe encryption tax | ↓ throughput | workload-specific | continuous |
| Attestation + KEK rotation | ↓ per-cold-start | seconds | per cold start / per rotation |
| Verifier ownership (team time) | ↑ | 0.2-0.5 FTE | continuous |
| Measurement allow-list maintenance | ↑ | ad-hoc | every patch cycle |
| Vendor-security-advisory response | ↑ | ad-hoc | per CVE |
| DPIA / legal review | ↑ | one-time + review cadence | annual |

### 2.6 NIST AI RMF hook

| Function | Artefact in this memo |
|---|---|
| Govern | Who owns the decision to adopt TEE+GPU CC? Which role approves measurement allow-list rotations? What's the exit plan if a vendor deprecates the chosen stack? |
| Map | §2.1 workload brief + §2.2 pattern matrix + the trust-boundary diagram from [[exercise-01-tee-trust-model-and-threat-map]] |
| Measure | Attestation failure rate, measurement-drift rate, output-DP budget consumed, latency SLO vs. baseline, cost per 1k inferences |
| Manage | Incident playbook: failing attestation → quarantine node; vendor CVE → rotate allow-list; DP budget low → retrain |

## 3. Implementation (memo deliverable shape)

```
exercise-07-confidential-ai-risk-and-cost-memo/
├── 1_workload_brief.md          # §2.1
├── 2_pattern_matrix.md          # §2.2
├── 3_recommendation.md          # §2.3
├── 4_residual_risk_register.md  # §2.4
├── 5_cost_estimate.md           # §2.5
└── 6_rmf_hook.md                # §2.6
```

Production steps:

1. **Write the workload brief first and fix it before anything
   else.** The pattern recommendation is a function of the
   workload. If the brief changes, the memo changes.
2. **Fill the pattern matrix using the module exercises as source
   of truth.** For each row, write one or two sentences and
   point back to the exercise. Do not paraphrase vendor pages.
3. **Write the recommendation before the cost estimate.** The
   cost estimate is scoped to the chosen pattern; estimating
   costs for all four at the detail level needed to compare is
   a six-month project.
4. **Residual-risk register with owners.** A risk without an
   owner is not accepted; it is pending.
5. **Cost estimate with sources.** Every number must either be a
   quoted vendor price-page SKU, a learner measurement, or a
   `<!-- citation-pending: ... -->` marker. No ranges without a
   source.
6. **RMF hook last.** This is the artefact that makes the memo
   useful to the programme office.

## 4. Validation steps

1. **Workload brief drives the matrix.** If you swap the brief
   (e.g., from "hosted LLM" to "cross-silo risk score"), at
   least two scoring rows must change. If they don't, the memo
   is not grounded in the workload.
2. **Every scoring row cites an exercise.** The matrix is a
   summary; the source of truth is exercises 01-06.
3. **Residual-risk list covers output leakage, model poisoning,
   side channels, supply chain, legal, DP budget.** Missing any
   of these is a signature failure — the memo would claim CC
   solves them.
4. **Recommendation includes rejected-alternative sentences** for
   every pattern that wasn't picked. "We chose TEE" without
   saying "not HE because <specific reason>" is incomplete.
5. **Cost estimate has sources.** Anything without a citation or
   a learner measurement is marked `<!-- citation-pending: ... -->`
   and does not go into the exec summary.
6. **RMF hook is actionable.** Each row names an owner or a
   trigger, not just a function name.
7. **Memo is review-length (≤ 2 pages when printed).** Longer
   memos don't get read.

## 5. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Workload brief states data, counterparties, regulatory frame, single question answered | 1 | §2.1 |
| Pattern matrix scores all four patterns against ≥ 5 dimensions | 2 | §2.2; must cite exercises |
| Recommendation + rejected-alternative sentences for every non-chosen pattern | 2 | §2.3 |
| Residual-risk register covers output leakage, poisoning, side channels, supply chain, legal, DP budget, with owners | 2 | §2.4 |
| Cost estimate has sources or `<!-- citation-pending: ... -->` markers; no uncited numbers | 1 | §2.5 |
| NIST AI RMF Govern/Map/Measure/Manage row each has an artefact + owner | 1 | §2.6 |
| Memo length ≤ 2 printed pages; reads like a decision doc | 1 | — |

Automatic failures:

- Claiming any pattern closes output leakage, MIA, or model
  inversion.
- Quoting overhead numbers without a source or a learner
  measurement.
- Residual-risk list without owners.
- Pattern choice that structurally doesn't fit the workload
  (e.g., MPC with one data party, HE for GPT-class inference).
- Memo longer than two pages or missing a recommendation.

## 6. Common mistakes

- **Treating the pattern choice as a vendor choice.** The four
  patterns live at different points in the design space; the
  right question is "what trust pattern do we need?" not "which
  vendor do we like?"
- **Conflating confidentiality with privacy.** TEE / HE / MPC
  are confidentiality-of-execution tools. Output leakage,
  membership inference, and model inversion are privacy risks
  that survive all of them and must be handled separately
  ([[mod-105-memorisation-extraction-and-mia]],
  [[mod-106-differential-privacy-engineering]]).
- **Importing vendor marketing as a residual-risk claim.** If a
  vendor post says "fully protects X," put that claim in the
  memo only as a *quote*, with the Apple / NVIDIA / AMD /
  Intel document footnoted. Do not restate it as a design
  property.
- **Writing a cost estimate without sources.** Cost estimates
  age quickly; untraceable ones get challenged in the first
  review and lose the memo's credibility. Cite price-page
  SKUs or mark the gap.
- **Over-claiming on MPC for a hosted LLM.** MPC's semi-honest
  guarantee + round complexity do not fit a GPT-class interactive
  serving loop. State this directly.
- **Under-claiming on PCC-style patterns.** For
  consumer-privacy-facing workloads, PCC-style designs
  ([[exercise-04-apple-pcc-security-design-read]]) carry
  design features (non-targetability, transparency log) that no
  CPU+GPU TEE deployment provides by default. The memo should
  acknowledge this when it is relevant.
- **Forgetting the exit plan.** Vendor stacks get retired. The
  memo should include at least a sentence on what happens if the
  chosen pattern is deprecated — otherwise the Govern row is
  incomplete.
- **Treating "hybrid" as a non-answer.** Real systems almost
  always combine patterns (TEE for the serving host, DP for the
  outputs, signed training for the pipeline). A memo that names
  the chosen *combination*, not just "TEE," is stronger.

## 7. References

- Confidential Computing Consortium. *Common Terminology for
  Confidential Computing* and *Confidential Computing — A
  Technical Analysis.* <https://confidentialcomputing.io/>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- Within this module:
  [[exercise-01-tee-trust-model-and-threat-map]],
  [[exercise-02-amd-sev-snp-attestation-walkthrough]],
  [[exercise-03-nvidia-h100-confidential-inference]],
  [[exercise-04-apple-pcc-security-design-read]],
  [[exercise-05-tenseal-he-inference-on-a-small-model]],
  [[exercise-06-crypten-mpc-inference-walkthrough]].
