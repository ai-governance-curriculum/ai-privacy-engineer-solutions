# Exercise 01 — TEE Trust Model and Threat Map — Reference Solution

## 1. Solution overview

This is a **design exercise**. The learner is asked to draw the trust
model for a Trusted Execution Environment (TEE)-backed AI inference
workload and map the threats that remain after "put the model in an
enclave" is on the architecture diagram. The output is not a vendor
comparison; it is a security memo a reviewer can hand to an auditor
or a red-teamer and have them agree on *what the TEE is and is not
buying*.

The assessable output has three parts:

1. **Trust-boundary diagram** — one picture (or ASCII table) that
   lists every component that touches plaintext data or plaintext
   model weights and marks it as **inside** or **outside** the
   Trusted Computing Base (TCB).
2. **Threat map** — a table that enumerates the adversaries,
   assets, and attack techniques that survive even when the TEE
   works as advertised, plus the mitigations.
3. **Residual-risk register** — a short list of the risks the
   design explicitly accepts (side channels, rollback, bugs in the
   enclave's own code, operator compromise of the attestation
   service), with the business rationale per entry.

The canonical references are the Confidential Computing Consortium's
"Common Terminology for Confidential Computing" white paper and
"Confidential Computing — A Technical Analysis," plus the vendor
reference documents for the hardware assumed by the scenario. The
threat vocabulary is kept consistent with MITRE ATLAS (for the
AI-specific attack techniques) and the NIST AI RMF (for the
governance wrapper).

## 2. Worked answer

### 2.1 What a TEE actually promises

A TEE is a hardware-enforced execution environment that provides
three things, and *only* these three:

1. **Confidentiality** of data and code loaded into the enclave
   against software outside the enclave (hypervisor, host OS,
   other VMs, cloud operator).
2. **Integrity** of code and data inside the enclave — tampering
   is either prevented or detected and aborts execution.
3. **Attestation** — a signed quote from the hardware root of
   trust that lets a remote party verify *what code is running* and
   *that it is running on genuine, patched hardware* before
   provisioning any secret.

Everything else is a *claim the system makes on top of the TEE*, not
a property of the TEE itself. In particular, a TEE does **not** by
itself provide:

- Protection against bugs in the code *inside* the enclave.
- Protection against side channels (cache, timing, power, EM, Spectre-
  class speculative execution) unless the vendor explicitly includes
  them in the threat model and provides mitigations.
- Protection against a malicious model developer (the model itself
  is trusted code as far as the enclave is concerned).
- Protection against "roll-back" of state when the enclave is
  restarted — unless the design adds a monotonic counter / sealed
  state mechanism.
- Protection of outputs *after* they leave the enclave.
- Protection against a compromised attestation verifier — the
  verifier is part of the TCB whether the design admits it or not.

### 2.2 Trust-boundary diagram (worked)

The scenario assumed below is the typical starter workload: a
hosted LLM inference endpoint where the model weights and the user
prompt must both be confidential from the cloud operator. The
pattern transfers to training and to on-device confidential
inference with minor substitutions.

```
┌────────────────────────── outside TCB ──────────────────────────┐
│  Cloud operator / hypervisor / host OS / other tenants          │
│  Network fabric / top-of-rack switch / NIC firmware             │
│  Storage service / object store                                 │
│  Orchestrator (k8s control plane, scheduler)                    │
│  Monitoring / metrics / log pipeline (unless scrubbed)          │
│  Attestation verifier *operator* (process — see §2.5)           │
│                                                                 │
│  ┌──────────────────── inside TCB ─────────────────────────────┐│
│  │  CPU TEE (SEV-SNP VM / TDX TD / SGX enclave)                ││
│  │   ├── Guest kernel + initramfs (if VM-based TEE)            ││
│  │   ├── Inference runtime (Python, PyTorch, CUDA stub, ...)   ││
│  │   ├── Model weights (loaded via sealed channel)             ││
│  │   ├── Per-request plaintext prompt + plaintext response     ││
│  │   └── Enclave-held keys (model KEK, attestation signing)    ││
│  │                                                             ││
│  │  Optional: GPU TEE (NVIDIA H100 CC mode) bound to the CPU   ││
│  │   ├── GPU VBIOS + driver inside CPU TEE                     ││
│  │   ├── Encrypted bounce buffers over PCIe                    ││
│  │   └── GPU attestation report chained to CPU report          ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  Root-of-trust hardware (CPU, GPU, TPM, PSP/SEAM/PCH)           │
│  Attestation *service* protocol endpoint (operator untrusted,   │
│     but signed quotes are self-authenticating — see §2.5)       │
└─────────────────────────────────────────────────────────────────┘
```

Key points a reviewer should be able to read off the diagram:

- The **hypervisor, host OS, orchestrator, and cloud operator are
  outside the TCB.** This is the whole point of using a TEE.
- The **guest OS and inference runtime are inside the TCB** for
  VM-based TEEs (SEV-SNP, TDX). That is a large TCB — kernel +
  userland + Python + model code — and must be acknowledged.
- The **network, storage, and orchestration path remain outside
  the TCB.** Secrets that cross them must be encrypted *under keys
  held only in the enclave*, released only after attestation.
- The **GPU, if used, is only in the TCB when the GPU itself runs
  in Confidential Computing mode** and its attestation is chained
  to the CPU attestation. A stock GPU accelerator bolted onto a
  SEV-SNP VM is **not** in the TCB.
- The **attestation verifier operator** (who runs the verifier
  service, deploys its code, holds its DB) is outside the TCB; the
  signed quote itself is self-authenticating because it chains to
  the hardware vendor's root CA. The design must treat the verifier
  *implementation* as part of the TCB but the verifier *operator*
  as not — a distinction many designs blur.

### 2.3 Threat map (worked)

| Asset | Adversary | Technique | Residual after TEE | Mitigation |
|---|---|---|---|---|
| Model weights (IP + training-data proxy) | Cloud operator with root on the host | Dump hypervisor memory, snapshot VM | Blocked by memory encryption + integrity (SEV-SNP VMPL / TDX ephemeral key) | Hardware does the work; verify by refusing to decrypt weights until attestation succeeds |
| Model weights | Insider with physical access | Cold-boot DRAM extraction | Blocked for SEV-SNP (encryption key never leaves PSP), SGX, TDX; **not blocked** for TEEs without memory encryption | Choose a TEE whose threat model includes physical attack; acknowledge if not |
| User prompt / per-request PII | Operator, other tenants | Hypervisor-level memory read, `/proc`/eBPF snooping | Blocked by TEE memory encryption + integrity | As above |
| User prompt | Co-tenant VM on same host | Spectre v2 / Foreshadow / MDS variants | **Only partially blocked** — depends on microcode + mitigations | Require patched microcode as a condition of attestation; enable SMT=off where the vendor recommends |
| Model output | Any party downstream of the enclave | Logging, screenshots, model-inversion / MIA on outputs | **Not addressed by TEE at all** | Output DP / rate limiting / policy filters [[mod-105-memorisation]]; MITRE ATLAS `AML.T0024 Exfiltration via ML Inference API` |
| Attestation freshness | Operator replaying an old quote to impersonate a patched enclave | Replay of stale `report_data` / nonce | Blocked **only if** the verifier binds a fresh nonce to the quote | Mandate a per-session nonce; reject quotes without one |
| TCB measurement | Operator ships an old vulnerable enclave image | Serve a build with known CVE | Blocked only if the verifier checks measurement against an allow-list of known-good builds | Maintain a signed image-measurement allow-list; rotate on CVE |
| Rollback / state freshness | Operator restarts enclave to replay a previous good state (e.g., old model policy) | Snapshot + restore, cold restart | **Not addressed by basic TEE** | Monotonic counter (TPM NVRAM, vendor-provided) + sealed state + rollback-detection logic in the enclave |
| Keys at rest | Operator exfiltrates sealed blobs | Copy sealed files | Blocked — sealing binds to measurement; cannot be unsealed outside that enclave | Keep sealed blobs; refuse to reuse unsealing material |
| Model poisoning / backdoor | Model developer (upstream) | Train in a backdoor before shipping to the enclave | **Not addressed by TEE** — the enclave happily runs a poisoned model | Supply-chain controls, reproducible training, MITRE ATLAS `AML.T0010 ML Supply Chain Compromise` |
| Side-channel leakage | Co-tenant on same socket | Cache/branch-predictor timing (Spectre, Prime+Probe) | Partially mitigated; **explicitly out of scope** for many TEEs | Document scope; add constant-time kernels where feasible; consider dedicated hosts |
| Membership inference | External API user | MIA on outputs [[mod-105-memorisation-extraction-and-mia]] | **Not addressed by TEE** | DP during training; output policy; monitor query patterns |
| Compromised verifier **implementation** | Supplier of the attestation verifier | Malicious verifier accepts tampered quotes | TCB expands to include verifier code | Treat verifier as TCB; pin to vendor-signed releases; log quote hashes |

### 2.4 NIST AI RMF mapping

| RMF function | Artefact for this exercise |
|---|---|
| Govern | Who signs off on the TCB contents? Which operator role is allowed to update the enclave image? Which jurisdictions does the attestation verifier operate from? |
| Map | The trust-boundary diagram in §2.2 is literally the Map artefact; keep it versioned beside the architecture doc |
| Measure | The verifier's measurement allow-list, freshness nonce logs, attestation failure counters, rollback-counter incidents |
| Manage | Incident playbook for a failing quote (quarantine the fleet node, rotate KEKs, revoke old measurements) |

### 2.5 The attestation-verifier subtlety

A pattern this exercise is designed to surface: learners often draw
the attestation service as a trusted blob in the TCB. That is almost
always wrong.

- The **hardware root CA** (AMD, Intel, NVIDIA, Apple, …) is in the
  TCB, because its signature anchors the attestation quote.
- The **verifier code** that parses the quote, checks the chain,
  validates the measurement, and binds a fresh nonce is in the TCB.
- The **operator** who runs the verifier (uptime, access to its
  database, who can deploy a new build) is **not** automatically
  trusted. If that operator can swap the verifier binary, they can
  trivially accept attacker-controlled quotes.

The design memo must either (a) argue that the verifier is operated
by a party the data owner already trusts (e.g., the data owner
themselves, a regulator, a dedicated appliance), or (b) list a
mitigation (vendor-signed verifier releases, reproducible builds,
quote-log transparency) that keeps the operator *out* of the TCB.

## 3. Implementation (deliverable shape)

This exercise has no runnable code — the "implementation" is a
small memo the learner commits to the repo. Recommended layout:

```
exercise-01-tee-trust-model/
├── 1_scenario.md                  # 2–3 sentences restating the workload
├── 2_trust_boundary.md            # the §2.2 diagram, filled in for the scenario
├── 3_threat_map.md                # the §2.3 table, scenario-specific
├── 4_residual_risks.md            # accepted risks + business rationale
└── 5_rmf_mapping.md               # Govern/Map/Measure/Manage per §2.4
```

Steps to produce each file:

1. **Pick (or restate) the scenario.** If the exercise provides one,
   paraphrase it in 2–3 sentences so your classification cannot hide
   behind the original wording. If you are free to pick, choose
   something concrete (e.g., "hosted GPT-class LLM inference where
   enterprise customers upload confidential prompts").
2. **List the TCB members explicitly.** Do not draw a cloud. Write
   a flat list: CPU firmware, PSP/SEAM, guest kernel, inference
   runtime, model weights, keys, verifier code. For each item say
   *why* it is in the TCB. The exercise fails if the kernel and
   the Python runtime are not on this list for a VM-based TEE.
3. **Fill the threat-map table one row per asset.** Use the §2.3
   template. Every row must have a mitigation *and* a residual-
   risk sentence. "Mitigated by TEE" alone is insufficient —
   mitigated against which adversary?
4. **Name at least one MITRE ATLAS technique per asset** that the
   TEE does *not* mitigate (e.g., `AML.T0024`, `AML.T0010`,
   `AML.T0040 ML Model Inference API Access`). This is the step
   that makes the memo useful for a security review instead of a
   marketing slide.
5. **Write the residual-risks register last.** Each entry: risk
   title, why the design accepts it, compensating control (if any),
   and owner. "Acceptance" without an owner is not an acceptance.
6. **Self-check against §5 before committing.**

## 4. Validation steps

A reviewer walks the memo through these checks:

1. **TCB census.** Does the memo list every component that touches
   plaintext data or model weights and mark it inside/outside the
   TCB? Missing the guest kernel or the inference runtime is the
   most common failure.
2. **Verifier placement.** Does the memo distinguish verifier
   *code* (TCB) from verifier *operator* (not TCB by default)?
3. **Attestation freshness.** Does the memo require a per-session
   nonce in the quote or explicitly justify why it does not?
4. **Rollback handling.** Does the memo address enclave restart /
   state replay, or explicitly accept the risk?
5. **Side channels.** Does the memo state whether side channels
   are in scope (and what is done) or explicitly out of scope
   (and why)?
6. **ATLAS mapping.** Does every asset row reference at least one
   MITRE ATLAS technique the TEE does not mitigate?
7. **Non-TEE threats.** Does the memo acknowledge that
   model-inversion, membership inference, and output leakage are
   *not* fixed by the TEE and need separate controls?
8. **RMF hook.** Are Govern/Map/Measure/Manage filled, not just
   name-dropped?

## 5. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Trust-boundary diagram enumerates every plaintext-touching component and marks it inside/outside the TCB | 2 | Guest kernel + runtime must be present |
| Threat map has ≥ 8 asset rows with technique + residual + mitigation | 2 | §2.3 is the template |
| Attestation freshness (nonce) and verifier placement (code vs. operator) called out | 1 | Common blind spot |
| Side-channel and rollback risks explicitly in or out of scope | 1 | — |
| ≥ 1 MITRE ATLAS technique per asset row where applicable | 1 | — |
| Non-TEE threats (model-inversion, MIA, output leakage) listed as still-open | 1 | — |
| NIST AI RMF Govern/Map/Measure/Manage mapping present | 1 | — |
| Residual-risk register has owners + compensating controls | 1 | — |

Automatic failures:

- Drawing the hypervisor or host OS *inside* the TCB.
- Treating "TEE enabled" as sufficient without naming the
  attestation verifier.
- Claiming that TEE prevents model-inversion / MIA / output leakage.
- Omitting the guest kernel / inference runtime from the TCB of a
  VM-based TEE.

## 6. Common mistakes

- **Treating the TEE as a privacy silver bullet.** TEEs give
  confidentiality + integrity + attestation against *software outside
  the enclave*. They do nothing about a backdoored model, nothing
  about output leakage, and nothing about a user who is authorised
  to query the API doing a membership-inference attack. The threat
  map must separate "what the hardware stops" from "what the
  product still has to handle."
- **Shrinking the TCB on paper.** A SEV-SNP VM or a TDX TD runs a
  full Linux guest; that kernel and every library in userland is in
  the TCB. SGX enclaves have a smaller TCB but only if the enclave
  code was written to stay small — common Python-based ML stacks
  make it large anyway. Honest diagrams show this.
- **Confusing the attestation *service* with the hardware root of
  trust.** The hardware root CA signs the quote; the verifier code
  checks it; the verifier operator runs the service. Each has a
  different trust status, and the memo must say which is which.
- **Forgetting the freshness nonce.** A valid quote without a
  nonce binding is replayable. Several real-world CVEs have been
  variants of this mistake.
- **Ignoring rollback.** If the enclave can be restarted at will,
  an operator can replay an older policy state and then query — a
  TEE does not prevent this without explicit monotonic state.
- **Pretending the GPU is in the TCB when it is not.** A stock
  accelerator bolted to a confidential VM is outside the TCB.
  Plaintext weights and activations cross the PCIe bus unless the
  GPU itself runs in a confidential-compute mode and its
  attestation is chained to the CPU's.
- **Using the OWASP ML Top 10 as the only threat model.** The
  OWASP ML Top 10 covers ML attack classes at a high level; it is
  useful for prioritisation but does not enumerate TEE-specific
  threats. Use MITRE ATLAS for AI-specific techniques and the
  Confidential Computing Consortium documents for the hardware
  threat model.

## 7. References

- Confidential Computing Consortium. *Common Terminology for
  Confidential Computing.*
  <https://confidentialcomputing.io/wp-content/uploads/sites/10/2023/03/Common-Terminology-for-Confidential-Computing.pdf>
- Confidential Computing Consortium. *Confidential Computing — A
  Technical Analysis.*
  <https://confidentialcomputing.io/wp-content/uploads/sites/10/2023/03/CCC-A-Technical-Analysis-of-Confidential-Computing-v1.3_Updated_November_2022.pdf>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS — Adversarial Threat Landscape for Artificial
  Intelligence Systems. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- AMD. *AMD SEV-SNP: Strengthening VM Isolation with Integrity
  Protection and More* (white paper).
  <https://www.amd.com/content/dam/amd/en/documents/epyc-business-docs/white-papers/SEV-SNP-strengthening-vm-isolation-with-integrity-protection-and-more.pdf>
- Intel. *Intel Trust Domain Extensions (Intel TDX) — White Paper.*
  <https://www.intel.com/content/dam/develop/external/us/en/documents/intel-tdx-whitepaper-final9-17.pdf>
- Intel. *Intel SGX Attestation Services — Developer Reference.*
  <https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/attestation-services.html>
