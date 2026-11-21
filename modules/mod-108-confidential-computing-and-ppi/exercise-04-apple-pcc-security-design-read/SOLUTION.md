# Exercise 04 — Apple PCC Security Design Read — Reference Solution

## 1. Solution overview

This is a **design-read / analysis exercise**. The learner reads
Apple's published Private Cloud Compute (PCC) material — the Apple
Security Research blog post *Blueprint for Private Cloud Compute*,
Apple Platform Security coverage of PCC, and the PCC *Security
Guide* / technical overview published alongside the launch — and
produces a structured review of the design against the trust model
and threat vocabulary developed in
[[exercise-01-tee-trust-model-and-threat-map]].

The output is not a summary of Apple's post; it is a **privacy-
engineer's design critique** that a reviewer could take to a
leadership meeting. The deliverable has four parts:

1. **Pillar map** — five pillars PCC claims (stateless compute,
   enforceable guarantees, no-privileged-runtime-access, non-
   targetability, verifiable transparency) each tied to the
   mechanism that is supposed to deliver it.
2. **Trust-boundary diagram** mapping PCC onto the framework from
   [[exercise-01-tee-trust-model-and-threat-map]] — what is in
   Apple's TCB, what is outside, which parties the design has
   to trust, which it does not.
3. **Threat read** — PCC's effect on each threat class from
   [[exercise-01-tee-trust-model-and-threat-map]], with the
   residuals called out explicitly (side channels, model-inversion,
   compromised Apple insider, hostile physical attack, output
   logging).
4. **Comparison note** — what PCC does *that a generic CPU+GPU TEE
   architecture does not*, and what a generic TEE architecture
   does that PCC chose not to (e.g., customer-held keys,
   customer-run verifier).

The analysis must be grounded in Apple's own published documents.
Any claim that the learner cannot source from an Apple document or
a direct empirical measurement must be marked
`<!-- citation-pending: ... -->` rather than asserted.

## 2. Worked answer

### 2.1 PCC's five pillars (per Apple's blog post)

Apple's *Blueprint for Private Cloud Compute* post names five
goals; the table below maps each to the mechanism Apple uses to
deliver it. Keep the learner's write-up grounded in Apple's own
wording — do not invent mechanisms.

| Pillar (Apple) | Mechanism Apple describes |
|---|---|
| **Stateless computation on personal data** | Each request is processed in an ephemeral node that is destroyed after the request; nodes do not retain user data between requests and there is no persistent user identifier in the request path |
| **Enforceable guarantees** | PCC runs a custom, hardened OS on Apple silicon servers; code execution is restricted to signed, attested images; secrets are released only to attested nodes |
| **No privileged runtime access** | Even Apple operators do not have interactive SSH / debugger access to a running PCC node; there is no "break-glass" path to live customer data |
| **Non-targetability** | The architecture is designed so that an attacker (including an Apple insider) cannot steer a specific user's request to a compromised node; load-balancing is designed to prevent user-level routing |
| **Verifiable transparency** | PCC software images are published to a transparency log; researchers can inspect them; the device refuses to send data to a server whose image is not in the public log |

Each learner-authored row should additionally cite **where in
Apple's materials** the claim comes from (blog section, security
guide chapter, Platform Security page). Claims that are not in
the published materials — e.g., exact crypto parameters, specific
TCB component versions, specific attestation key hierarchies —
belong in `<!-- citation-pending: ... -->` comments until they can
be cited.

### 2.2 Trust boundary (PCC mapped onto exercise-01)

```
┌─── outside Apple ─────────────────────────────┐
│  User's non-Apple threat surface (home ISP,   │
│  Wi-Fi, BGP, third-party apps on device)      │
│                                               │
│  ┌── outside the per-request TCB ──────────┐  │
│  │  Apple network fabric / load balancer   │  │
│  │  Apple datacenter operators (admin)     │  │
│  │  CI / build / release infrastructure    │  │
│  │  Transparency-log operator (Apple)      │  │
│  │                                         │  │
│  │  ┌── per-request PCC TCB ─────────────┐ │  │
│  │  │ Apple-silicon server (SEP + SoC)   │ │  │
│  │  │ Custom hardened OS image           │ │  │
│  │  │ Model runtime + model weights      │ │  │
│  │  │ Ephemeral per-request state        │ │  │
│  │  │ Attestation keys in Secure Enclave │ │  │
│  │  └────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────┘  │
└───────────────────────────────────────────────┘

User's client device (iPhone / iPad / Mac):
  verifies the PCC node's attestation, checks the
  image hash is in the public transparency log,
  and only then sends the request.
```

Key engineering points:

- The **user's device** is the attestation verifier for PCC.
  The user does not outsource "is this a trustworthy PCC node"
  to Apple's cloud control plane; the device itself checks the
  attestation and the transparency-log inclusion proof. This is
  the structural property that lets Apple keep the datacenter
  operator *out* of the per-request TCB.
- **No customer-held keys.** Unlike BYO-KMS designs in other
  confidential-compute offerings, PCC's model keys and node keys
  are Apple's. The user is not "co-signer" of a decryption
  decision; they are a *verifier* of an attested policy.
- **Transparency log as a structural control.** The design
  substitutes "any image that lands on a PCC node is auditable
  by anyone" for "the data owner audits each image personally."
  This is a different trust pattern from CPU TEE deployments
  where the data owner runs the attestation verifier and gates
  key release; the device plays that role here.
- **Non-targetability** is a system property (load balancing +
  blinded routing) rather than a cryptographic one. The learner's
  read must flag that this is a design assumption with specific
  threat model limits — e.g., it depends on an attacker not being
  able to fingerprint PCC nodes from outside.

### 2.3 Threat read (PCC vs. the exercise-01 threat map)

For each asset from [[exercise-01-tee-trust-model-and-threat-map]],
what PCC does and what remains residual. Keep the second column
grounded in Apple's published claims; the third column is the
privacy engineer's judgement call.

| Asset | PCC mechanism (per Apple docs) | Residual |
|---|---|---|
| User request payload | Encrypted end-to-end from device to PCC node under keys bound by attestation | Attack on device itself (device is the trusted verifier); side-channel attacks on PCC node |
| Model weights | Loaded into the node from Apple-signed images; node OS prevents dumping | Backdoor in the model is not addressed by PCC; MITRE ATLAS `AML.T0010 ML Supply Chain Compromise` |
| Server-side state | Stateless pillar — no per-user state retained between requests | Depends on strict enforcement; a node that caches aggressively for performance would silently weaken this |
| Insider access | "No privileged runtime access" pillar — no interactive debugger / SSH into a running node | Apple's build pipeline is still privileged; a compromised release engineer could ship a malicious image *provided it survived transparency log review* |
| Request targeting | Non-targetability pillar — load balancer designed to prevent per-user steering | Timing / usage-pattern fingerprinting outside the PCC edge is not inside PCC's threat model |
| Output leakage / model-inversion / MIA (`AML.T0024`) | PCC does not address these; it's a confidentiality-of-transport-and-execution system | Output DP, rate limiting, policy filters [[mod-105-memorisation-extraction-and-mia]] |
| Rollback | Each request runs on an ephemeral node, so classical TEE rollback does not apply | Misbehaviour in the build pipeline (shipping an older signed image) is bounded by the transparency log, not by cryptographic monotonicity |
| Side channels on Apple silicon | Not addressed by any public PCC document in detail | Document as explicitly out of scope for the current PCC generation |

### 2.4 Comparison with a generic CPU+GPU TEE deployment

| Dimension | Generic CPU TEE + GPU CC-mode (ex. 02 + 03) | PCC |
|---|---|---|
| Who runs the verifier? | Data owner (or data owner's trusted appliance) | The user's own Apple device |
| Who holds the data-release key? | Data owner (via KMS wrapped under attested session pubkey) | Apple (data-release decision is encoded in the attestation + signed image) |
| Who defines the allow-list of images? | Data owner | Apple, with the transparency log as the external check |
| What stops "run an older image"? | Measurement allow-list + monotonic counters | Transparency log + the device refusing to send to an image not in the log |
| What stops a targeted attack on one user? | Nothing in-band (requires separate controls) | The "non-targetability" pillar is a first-class design goal |
| Multi-tenancy posture | Possible, if co-tenancy is engineered | Node is logically single-request; no co-tenancy |
| Side-channel coverage | Varies by vendor (AMD SEV-SNP, Intel TDX, NVIDIA CC mode each publish different scope) | Not publicly detailed |
| Ecosystem openness | Open — any workload can use it | Closed — Apple's workloads only |

### 2.5 NIST AI RMF mapping for the learner's review

| RMF function | PCC-specific artefact |
|---|---|
| Govern | Who inside the acquiring org decides whether PCC's "Apple is the key holder, user device is the verifier" trust pattern is acceptable for the workload? What's the policy for off-device inference eligibility? |
| Map | The learner's pillar map (§2.1) + trust-boundary diagram (§2.2) |
| Measure | What signals would detect PCC drift from its pillars? E.g., transparency-log monitoring, client-side attestation failure rates, model-output privacy monitoring (independent of PCC) |
| Manage | Fallback when PCC is unavailable or an attestation fails — on-device degraded mode, user-visible messaging, logging policy |

## 3. Implementation (deliverable shape)

```
exercise-04-apple-pcc-security-design-read/
├── 1_pillar_map.md           # §2.1 filled out with citations to Apple docs
├── 2_trust_boundary.md       # §2.2 diagram, your version
├── 3_threat_read.md          # §2.3 table, your version
├── 4_comparison.md           # §2.4 table vs. generic TEE
└── 5_rmf_mapping.md          # §2.5 for the organization that is adopting PCC
```

Production steps:

1. **Read the primary source material in full.** Apple's blog
   post, the PCC security overview on Apple's Platform Security
   page, and any linked technical guide. Take notes with page /
   section anchors — every claim in the write-up must cite one of
   these sources or be marked `<!-- citation-pending: ... -->`.
2. **Write the pillar map first.** One row per pillar, one
   sentence of mechanism, one link to where Apple makes the claim.
   If a pillar's mechanism isn't explicit in the published docs,
   say so — don't invent.
3. **Draw the trust boundary in your own words.** Then compare to
   [[exercise-01-tee-trust-model-and-threat-map]]. Highlight the
   two structural choices that distinguish PCC from a generic TEE
   deployment: user-device-as-verifier and transparency-log-as-
   control.
4. **Walk the threat map** from [[exercise-01-tee-trust-model-and-threat-map]]
   and mark, per asset, PCC's mechanism and what remains residual.
   Do not claim PCC solves output leakage, model-inversion, or
   MIA — it is a confidentiality-of-execution system, not a
   privacy-of-outputs system.
5. **Write the comparison.** The point is not to score; it is to
   make the trust-pattern differences legible to a decision maker
   who has never built on either PCC or a generic TEE stack.
6. **Write the RMF mapping last.** This is the artefact that
   turns the review into something a program office can act on.

## 4. Validation steps

1. **Citation audit.** Every mechanism claim in §2.1 and §2.3
   cites an Apple document (blog post, Platform Security section,
   or published PCC guide). Claims without a citation are either
   marked `<!-- citation-pending: ... -->` or removed.
2. **Trust-boundary completeness.** Does the diagram include the
   user's device as the verifier, the transparency log as an
   external check, the Apple build pipeline as privileged-but-
   bounded, and the data-centre operator as deliberately
   excluded?
3. **Residuals are honest.** Does the threat read list
   model-inversion, MIA, output leakage, side channels, and build-
   pipeline compromise as *not fully solved by PCC*?
4. **Non-targetability caveat.** Does the memo note that non-
   targetability is a *system* property that depends on specific
   load-balancing / routing assumptions, not a cryptographic
   guarantee?
5. **Comparison is directional, not preference.** Does §2.4
   state trade-offs rather than declare PCC "better" or "worse"
   than a generic TEE stack?
6. **RMF mapping is actionable.** Could a program officer point
   at a row in §2.5 and assign it to someone?

## 5. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Pillar map correctly names all five pillars with cited mechanisms | 2 | §2.1 |
| Trust-boundary diagram places user device as verifier and excludes datacenter operator | 2 | §2.2 |
| Threat read covers ≥ 6 assets from [[exercise-01-tee-trust-model-and-threat-map]] with residuals | 2 | §2.3 |
| Comparison with generic TEE calls out: verifier identity, key holder, allow-list owner, targeting resistance | 2 | §2.4 |
| RMF mapping has concrete actions per function | 1 | §2.5 |
| Every mechanism claim is cited or marked `<!-- citation-pending: ... -->` | 1 | — |

Automatic failures:

- Summarising Apple's post without citations.
- Claiming PCC solves model-inversion / MIA / output leakage.
- Treating PCC's "non-targetability" as a cryptographic
  guarantee.
- Omitting the transparency log from the trust model.
- Asserting specific crypto parameters / TCB versions / attestation
  key hierarchies that are not in Apple's public documents.

## 6. Common mistakes

- **Reading PCC as "Apple's version of a TEE."** It is a system
  built *on* Apple-silicon security primitives, but the
  interesting design decisions (user device as verifier,
  transparency log, non-targetability) are *not* available in a
  generic AMD / Intel / NVIDIA TEE deployment and are what the
  review should focus on.
- **Confusing the transparency log with reproducible builds.**
  The log says: "if a PCC image is deployed, it is publicly
  listed and open to inspection." It does not by itself prove the
  image is bug-free — it prevents *secret* compromise, not
  compromise. Researchers still have to look at the images.
- **Over-trusting non-targetability.** Non-targetability is a
  system property with specific threat-model assumptions
  (adversary cannot steer load, cannot fingerprint nodes from
  outside). It is not cryptographically enforced. State it as a
  design assumption.
- **Ignoring the model's own behaviour.** PCC does not change
  anything about the model's propensity to memorise its training
  data, to leak via outputs, or to be susceptible to MIA. Those
  live in [[mod-105-memorisation-extraction-and-mia]].
- **Quoting specifics that aren't in the published docs.**
  Do not invent attestation key hierarchies, specific TCB
  component versions, or "x % overhead" numbers for PCC. If a
  specific is needed, mark it `<!-- citation-pending: ... -->`.
- **Treating the user's device as "trusted enough."** PCC
  explicitly makes the device the verifier. That shifts part of
  the TCB *onto the device*. If the device is compromised, the
  "server side is trustworthy" claim is moot. Call this out.

## 7. References

Primary (Apple):

- Apple Security Research. *Blueprint for Private Cloud Compute.*
  <https://security.apple.com/blog/private-cloud-compute/>
- Apple. *Private Cloud Compute (PCC) — Security Guide / technical
  overview.* <https://security.apple.com/documentation/private-cloud-compute/>
- Apple. *Apple Platform Security — Private Cloud Compute* (within
  the Platform Security PDF / portal).
  <https://support.apple.com/guide/security/welcome/web>

Threat vocabulary:

- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS — Adversarial Threat Landscape for AI Systems.
  <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>

Context from this module:

- [[exercise-01-tee-trust-model-and-threat-map]] — the trust-model
  template this review is written against.
- [[exercise-02-amd-sev-snp-attestation-walkthrough]] and
  [[exercise-03-nvidia-h100-confidential-inference]] — the
  "generic TEE stack" referenced in §2.4.
