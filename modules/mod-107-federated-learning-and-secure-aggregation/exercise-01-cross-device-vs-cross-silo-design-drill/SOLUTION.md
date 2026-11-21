# Exercise 01 — Cross-Device vs. Cross-Silo Design Drill — Reference Solution

## 1. Solution overview

This is a **design exercise**. The learner is given one or more training
scenarios and must decide whether each is a **cross-device** or
**cross-silo** federated learning (FL) setting, then draw the engineering
implications: participant count, participation reliability, stateful vs.
stateless clients, sampling regime, system identity, trust model, and
the privacy mechanisms that match. The deliverable is not just a label
but a short design memo a reviewer can use to allocate engineering and
privacy budget.

The assessable output has three parts:

1. **Classification table** — for each scenario, the label
   (cross-device / cross-silo / hybrid) with the three or four
   properties that forced the label.
2. **Design-implication matrix** — a per-scenario statement of what
   changes in the system because of the label: round length,
   participation rate, sampling, state, failure handling, privacy
   mechanism, and threat model.
3. **One rejected alternative per scenario** — a one-sentence
   explanation of the label a less careful reviewer would pick and why
   that label would be wrong.

The canonical distinction follows Kairouz et al., *Advances and Open
Problems in Federated Learning* (2021), Table 1, which is the standard
reference the broader FL literature cites. Governance language
(trust zones, threat enumeration, accountability) is kept consistent
with the NIST AI Risk Management Framework (AI RMF 1.0) so the design
memo can plug into a Govern/Map/Measure/Manage review without
re-labelling.

## 2. Worked answer

### 2.1 The distinction — what the two labels actually mean

| Dimension | Cross-device FL | Cross-silo FL |
|---|---|---|
| Typical client count | 10⁶–10⁹ (phones, cars, IoT) | 2–100 (hospitals, banks, business units) |
| Client participation | Unreliable, opportunistic (charging + Wi-Fi, idle) | Scheduled, highly reliable |
| Addressability | Clients are **anonymous and ephemeral**; a given client is seen rarely, if ever, again | Clients are **named, stable institutions** |
| State | Stateless clients; the server cannot rely on client-local state across rounds | Stateful clients; rounds can resume where they left off |
| Sampling | Server samples a tiny fraction (e.g., 100 of 10⁶) each round | Typically **all** silos participate every round |
| Data per client | Small (a single user's phone); heterogeneous | Large (a whole institution); heterogeneous |
| Network | Edge cellular / consumer Wi-Fi; high dropout | Site-to-site / VPN; low dropout |
| Aggregation | Secure aggregation (Bonawitz) is almost required | Secure aggregation optional; TEE or MPC are realistic |
| Primary privacy claim | **User-level** differential privacy aggregated across clients | Record-level or dataset-level agreements; DP is layered on if cross-institution contamination must be bounded |
| Trust model | Clients are mutually distrustful and untrusted; server is honest-but-curious or worse | Participants are contractually bound; adversary is usually an outside attacker or a single rogue silo |
| Governance anchor | Consent + on-device rotation + per-release DP | Data-sharing agreements, DPIAs per silo, joint controller analysis |

The label is **not** chosen by participant count alone. "Three hospitals
federating patient data" is cross-silo even if one hospital is tiny;
"ten enterprise phones in a pilot" is cross-device because the
*operational contract* (unreliable, anonymous, stateless) is the
cross-device contract.

### 2.2 Classification table (worked scenarios)

The exercise typically presents three or four scenarios. The six below
cover the forcing properties the learner should name. If the exercise
uses different scenarios, the same reasoning applies — the label is
driven by *trust, addressability, and reliability*, not by headcount.

| # | Scenario | Label | Forcing properties |
|---|---|---|---|
| A | Next-word prediction on 50 M Android keyboards; opt-in; model updates only when the phone is idle, charging, and on unmetered Wi-Fi | Cross-device | Stateless, ephemeral clients; server samples ≪ 1 % per round; user-level DP + secure aggregation expected |
| B | Fraud model trained across 5 banks in one jurisdiction; weekly synchronous rounds; each bank operates its own training node inside its VPC | Cross-silo | Stable, named, contractually bound participants; full-participation rounds; no end-user devices in scope |
| C | 40 hospitals federating a chest-X-ray classifier under a BAA; each hospital is sampled per round due to network cost | Cross-silo | Institutions are named parties; sampling is for cost, not anonymity; HIPAA record-level agreements layered over FL |
| D | Smart-meter forecasting across 2 M residential meters via a utility's concentrator | Cross-device | Massive stateless fleet; opportunistic uplink; sub-1 % participation per round |
| E | Autonomy perception model trained across 300 test vehicles belonging to one OEM, each with a consistent uplink while parked overnight | Hybrid (cross-silo-ish) | Named, stateful fleet owned by one party; design can lean cross-silo because clients are reliable, but DP accounting may still want user-level (per-vehicle) framing |
| F | On-device LLM fine-tune of a consumer app, 1 M users, 2 h daily uplink windows | Cross-device | Anonymous consumer users; privacy claim is user-level; secure aggregation + DP are table stakes |

### 2.3 Design-implication matrix

For every scenario above, the table forces the engineering choices
below. These are the lines that should appear in the learner's memo.

| Design dimension | Cross-device answer (A, D, F) | Cross-silo answer (B, C) | Hybrid answer (E) |
|---|---|---|---|
| Round orchestration | Server pulls from a pool of currently-eligible clients; round closes on timeout, not on 100 % participation | Scheduled rounds, named silos, barrier synchronisation | Scheduled fleet rounds; fall-back to "k-of-n reporting" for silo semantics |
| Sampling | Poisson over the eligible pool; q ≪ 1 % | Typically all silos; weight by data size if heterogeneous | Poisson over vehicles; q selected per DP accounting |
| Client state | Treat as stateless; the server stores any state | Stateful, per-silo; local resumption across rounds | Fleet-local state is acceptable; privacy claim ignores it |
| Failure handling | Clients drop silently; secure aggregation must tolerate O(√n) dropouts (Bonawitz § 4) | Retry / backoff per silo; failed silos excluded with an explicit decision record | Mix: silo-style retry per vehicle, fleet-level dropout budget |
| Identity | Ephemeral client IDs rotated per round; devices not re-identifiable across rounds | Long-lived mTLS identities; audit logged | Fleet identities; rotation only if the privacy claim requires it |
| Secure aggregation | **Required** — Bonawitz-style (see [[exercise-04]]) because the server should never see a single-client update | Optional; MPC/TEE across 2–100 silos is tractable; may be mandated by the data-sharing agreement | Required if the privacy claim is per-vehicle DP |
| DP unit | **User-level DP** (one phone = one user) with amplification via subsampling | Record-level DP or no DP (contractual boundary carries the privacy claim); dataset-level DP if cross-silo leakage must be bounded | Per-vehicle user-level DP |
| DP mechanism | DP-FedAvg (McMahan et al. 2018): clip per-client update, add Gaussian noise *at the aggregator* | DP-FedAvg with a much smaller noise budget, or no DP; audit for silo-level leakage | DP-FedAvg with per-vehicle clipping |
| Threat model (MITRE ATLAS mapping) | Primary: *AML.T0024 Exfiltration via ML Inference API* at the server side; *AML.T0020 Data from Local System* if a client is compromised | Primary: compromised silo (insider / supply chain); *AML.T0010 ML Supply Chain Compromise* via silo code | Fleet-wide compromise; OEM internal threat |
| Governance mapping (NIST AI RMF) | Govern/Map: user-consent artefacts, on-device policy; Measure: DP accountant output; Manage: rotation + revocation story | Govern: DPIA per jurisdiction, joint-controller agreement; Map: silo-level data-flow diagram; Measure: convergence + leakage drill | Both: fleet DPIA + per-vehicle consent envelope |

### 2.4 Rejected-alternative sentences

Each scenario memo includes one "why not the other label" sentence:

- **A** — Not cross-silo: 50 M consumer phones have no joint controller
  agreement and no reliable uplink; you cannot plan a synchronous
  round across them.
- **B** — Not cross-device: five named banks sign a joint-controller
  agreement, participate every round, and expose stable identities;
  anonymising them discards the contractual trust that is the whole
  privacy model.
- **C** — Not cross-device: hospitals are legal entities under BAAs;
  stripping that framing collapses the HIPAA chain of accountability
  without buying any of the cross-device privacy benefits.
- **D** — Not cross-silo: smart meters are consumer-grade,
  ephemeral-ID, sub-1 %-participation endpoints; the utility is a
  processor over millions of users, not a joint-controller partner.
- **E** — Not pure cross-device: one OEM owns and operates every
  vehicle, so the trust model is single-party; the cross-device label
  would over-engineer secure aggregation and under-document the OEM's
  controller status.
- **F** — Not cross-silo: consumer app users are not institutional
  partners; the privacy story is user-level DP, not a
  joint-controller agreement.

## 3. Implementation

This exercise has no runnable code — the "implementation" is the
design-memo artefact the learner commits to the repository. The
recommended shape is one Markdown file per scenario (or a single
file with one section per scenario) with the following structure,
so a reviewer can map each paragraph back to §2 and §5:

```
scenario-<id>-<short-slug>.md
├── 1. Scenario restatement (2–3 sentences, in the learner's own words)
├── 2. Classification
│     ├── Label: cross-device | cross-silo | hybrid
│     └── Forcing properties (≥ 3 from §2.1: trust, addressability,
│         participation, state, sampling)
├── 3. Design-implication memo (one row per dimension in §2.3)
│     ├── Round orchestration
│     ├── Sampling
│     ├── Client state
│     ├── Failure handling
│     ├── Identity
│     ├── Secure aggregation (explicit yes/no + why)
│     ├── DP unit (user / record / dataset / none)
│     ├── DP mechanism
│     ├── Threat model (≥ 1 MITRE ATLAS technique)
│     └── Governance mapping (NIST AI RMF function)
└── 4. Rejected alternative (one sentence; see §2.4)
```

Concrete steps to produce each scenario file:

1. **Restate the scenario.** Paraphrase the brief in 2–3 sentences
   so the classification cannot hide behind the original wording.
2. **Run the three-property test from §2.1 before writing the label.**
   List the forcing properties first; the label falls out of them.
   If fewer than three properties point the same way, the scenario
   is probably hybrid — flag that explicitly instead of forcing a
   label.
3. **Fill the design-implication matrix row by row.** Use §2.3 as a
   template; do not skip rows, even if the answer is "not
   applicable — stateless clients." A written "not applicable" is a
   decision; a blank row is an omission.
4. **Make the DP unit explicit.** Write one of
   `user-level | record-level | dataset-level | none` and note
   whether the mechanism (clipping granularity, noise site) matches.
   If the mechanism does not match the unit, rewrite one of them
   before continuing.
5. **Make the secure-aggregation decision explicit.** For every
   cross-device scenario, state "secure aggregation: required
   (Bonawitz-style)" or "secure aggregation: deferred because
   <reason>." A missing decision is the most common §6 failure.
6. **Attach at least one MITRE ATLAS technique** (e.g.,
   `AML.T0024 Exfiltration via ML Inference API`,
   `AML.T0010 ML Supply Chain Compromise`) and the NIST AI RMF
   function it maps to (Govern / Map / Measure / Manage).
7. **Write the rejected alternative last.** One sentence stating
   which label a less careful reviewer would pick and the specific
   property that makes that label wrong — not just the chosen label
   with "not" in front of it.
8. **Self-check against §5 before committing.** If any scoring row
   would score zero, fix the memo before submitting; do not rely on
   the reviewer to find the gap.

For hybrid scenarios, produce two sub-memos in the same file —
one for the cross-device-leaning dimensions (typically the privacy
accounting) and one for the cross-silo-leaning dimensions
(typically the round orchestration and identity model) — so the
split is explicit rather than implied.

## 4. Validation steps

A reviewer (or the learner self-checking) walks the memo through these
checks:

1. **Three-property test.** For each label, the memo names at least
   three forcing properties from §2.1 (trust, addressability, state,
   participation, sampling). Any scenario labelled on headcount alone
   fails.
2. **Privacy-mechanism coherence.** The chosen DP unit (user-level,
   record-level, dataset-level) is consistent with the chosen round
   topology. User-level DP on a cross-silo system without per-user
   clipping is incoherent; the memo must either rewrite the mechanism
   or revise the unit.
3. **Secure-aggregation consistency.** For every cross-device entry,
   the memo either includes secure aggregation or explains in one
   sentence why it is deferred. ("The server is trusted to see
   per-client updates" is an *explicit* decision, not a default.)
4. **Threat-model cross-check.** Every scenario names at least one
   MITRE ATLAS technique that applies in the chosen topology, and the
   NIST AI RMF function it maps to (Govern / Map / Measure / Manage).
5. **Rejected-alternative test.** For every classification, the
   one-sentence counterfactual is present and does not just restate the
   chosen label negated.
6. **Hybrid honesty.** Any scenario flagged "hybrid" is accompanied by
   which *half* of the design leans which way. "Hybrid" without a
   concrete split is treated as an unanswered question.

## 5. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Classification table lists at least three forcing properties per scenario | 2 | Trust, addressability, participation, state, sampling |
| Each scenario labels its DP unit explicitly (user / record / dataset / none) | 2 | Must match the round topology |
| Secure-aggregation decision is explicit for every scenario | 1 | "Yes/no + why" |
| Threat model names at least one MITRE ATLAS technique per scenario | 1 | Pulled from the ATLAS matrix |
| NIST AI RMF function named per scenario | 1 | Govern/Map/Measure/Manage |
| Rejected-alternative sentence present and substantive | 2 | Not the chosen label with "not" in front |
| Hybrid scenarios split explicitly into "cross-device-ish" vs. "cross-silo-ish" dimensions | 1 | — |

Automatic failures:

- Classifying on participant count alone.
- Claiming user-level DP on a system that does not clip per user.
- Omitting secure aggregation from a cross-device design *without* a
  written exception.
- Treating cross-silo as "small cross-device"; the trust model is
  categorically different.

## 6. Common mistakes

- **Counting clients, not contracts.** Headcount is a *symptom* of the
  topology, not the forcing property. Two hundred smart speakers from
  one vendor can be cross-silo if the vendor runs them under a single
  controller; ten million users under a consumer product are
  cross-device even if only a few thousand ever actually participate
  per round.
- **Collapsing the privacy claim.** Cross-device FL's privacy story is
  **user-level DP + secure aggregation + consent**; cross-silo's story
  is **contractual + DPIA per silo + record-level controls**. Papers
  sometimes report "DP-FL with ε = 1" without saying which. The
  learner's memo must not do the same.
- **Assuming secure aggregation is a drop-in for cross-silo.**
  Bonawitz-style secure aggregation is designed for many small,
  unreliable clients; applying it across three hospitals is
  over-engineering. For 2–100 parties, MPC or a TEE at the aggregator
  is usually a better match.
- **Forgetting the sampling-amplification story.** Cross-device DP
  depends on q ≪ 1 for amplification by subsampling (Abadi et al.
  2016, Mironov et al. 2019). Cross-silo often runs all silos every
  round, so the amplification is not available — but q = 1 is still a
  valid DP claim; the accountant is just tighter in one case and
  looser in the other.
- **Over-attributing to the server.** Both topologies assume the server
  is at least honest-but-curious. The *difference* is whether a single
  compromised participant leaks a single user (cross-device) or a
  whole dataset (cross-silo). The threat model must say which.
- **Treating "hybrid" as a non-answer.** Real systems are often
  hybrid. A good memo *splits* the hybrid: "fleet-level participation
  is cross-silo; per-vehicle privacy accounting is cross-device."
- **Using OWASP ML Top 10 as the only threat model.** OWASP ML Top 10
  covers ML attack classes at a high level and is useful for
  prioritisation, but for FL-specific threats you still need ATLAS
  techniques (ATLAS has FL-specific techniques around
  model-poisoning and inference-time leakage) and the NIST AI RMF
  categorisation.

## 7. References

- Kairouz, P.; McMahan, H. B.; et al. *Advances and Open Problems in
  Federated Learning.* Foundations and Trends in Machine Learning
  14(1–2), 2021. <https://arxiv.org/abs/1912.04977> — Table 1 is the
  canonical cross-device vs. cross-silo contrast used here.
- McMahan, H. B.; Moore, E.; Ramage, D.; Hampson, S.; y Arcas, B. A.
  *Communication-Efficient Learning of Deep Networks from Decentralized
  Data.* AISTATS 2017. <https://arxiv.org/abs/1602.05629> — the FedAvg
  paper that defines the cross-device training loop.
- McMahan, H. B.; Ramage, D.; Talwar, K.; Zhang, L. *Learning
  Differentially Private Recurrent Language Models.* ICLR 2018.
  <https://arxiv.org/abs/1710.06963> — the DP-FedAvg algorithm and the
  user-level DP framing.
- Bonawitz, K.; Eichner, H.; Grieskamp, W.; et al. *Towards Federated
  Learning at Scale: System Design.* MLSys 2019.
  <https://arxiv.org/abs/1902.01046> — the production-side cross-device
  system that most of the engineering choices in §2.3 come from.
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS — Adversarial Threat Landscape for AI Systems.
  <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
