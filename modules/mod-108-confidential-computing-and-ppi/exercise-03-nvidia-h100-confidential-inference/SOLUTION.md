# Exercise 03 — NVIDIA H100 Confidential Inference — Reference Solution

## 1. Solution overview

This is an **implementation / walkthrough exercise**. The learner
must describe — with working artefacts where feasible — how to run
an inference workload on an **NVIDIA H100 GPU in Confidential
Computing (CC) mode**, bound to a CPU TEE (SEV-SNP or TDX), such
that neither the host operator nor a co-tenant can read model
weights, prompts, or activations.

Assessable artefacts:

1. **Architecture note** that shows which components are inside
   the TCB (CPU TEE + H100 in CC mode) and which remain outside
   (hypervisor, host OS, PCIe switch firmware, orchestrator).
2. **Enablement checklist** — the exact VBIOS mode, kernel driver
   state, and nvidia-smi commands needed to put the H100 into CC
   mode and confirm it.
3. **Attestation walkthrough** — how a relying party verifies the
   **chained** attestation: CPU report (SEV-SNP or TDX) **plus**
   GPU attestation report from NVIDIA RIM / NRAS.
4. **Reference inference wrapper** that refuses to decrypt model
   weights until the combined CPU+GPU attestation has succeeded.
5. **Residual-risk note** covering what CC mode does and does not
   fix (bandwidth tax, co-tenancy unsupported, driver/VBIOS as
   TCB, output leakage, side channels, MITRE ATLAS mapping).

The construction follows NVIDIA's official Confidential Computing
documentation for the Hopper architecture: the *Hopper Confidential
Computing Deployment Guide*, the H100 product brief for CC mode,
and the NVIDIA Attestation Services (NRAS / Remote Attestation
Service) documentation. The TEE pairing uses either AMD SEV-SNP
([[exercise-02-amd-sev-snp-attestation-walkthrough]]) or Intel TDX
on the host. Governance language is kept consistent with the NIST
AI RMF and MITRE ATLAS.

## 2. Worked implementation walkthrough

### 2.1 What H100 CC mode gives you

NVIDIA's Confidential Computing mode on Hopper (H100) extends a
CPU TEE's confidentiality + integrity + attestation guarantees
to the GPU and to the data on the PCIe path between them:

- **Encrypted PCIe path.** Data moving between the CPU TEE VM and
  the GPU is encrypted/integrity-protected using an ephemeral
  symmetric key negotiated at session setup. Plaintext never
  appears on the physical PCIe bus.
- **Encrypted GPU memory (HBM).** Model weights and activations
  live in HBM under a key owned by the GPU; neither the
  hypervisor nor other tenants can read HBM.
- **GPU attestation.** The H100 produces a signed attestation
  report whose signer chains to NVIDIA's root. The report
  measures GPU VBIOS, driver pieces, and the configured mode
  (CC-on, developer-mode-off), and can be freshness-bound to a
  nonce the relying party provides.
- **Single-tenant / passthrough binding.** CC mode on H100 is
  single-tenant: the GPU is assigned via PCIe passthrough to one
  guest TEE VM for the life of the session. **MIG (Multi-Instance
  GPU) is not supported with CC mode on H100** at the time of
  the Hopper deployment guide — the design must acknowledge this.
- **Performance cost.** The encryption path costs effective PCIe
  bandwidth (NVIDIA publishes benchmark numbers in the Hopper CC
  deployment guide and in the NVIDIA Developer Blog launch
  post). The exact cost is workload-dependent — small activations
  and heavy host↔device traffic hurt most; large batched LLM
  inference where host↔device traffic is dominated by the initial
  weight load is least affected. **Do not quote a specific
  percentage unless you are citing a published NVIDIA benchmark
  and matching your workload to it.**

What CC mode **does not** give you:

- No defence against a backdoored model.
- No defence against output leakage, model-inversion, or
  membership-inference attacks on the API.
- No defence against bugs in the inference runtime inside the
  guest.
- MIG-based multi-tenancy — not available in CC mode.
- Cryptographic protection *outside* the attested path (e.g., on
  the user-facing TLS endpoint — that's still the application's
  job).

### 2.2 Trust boundary (chained CPU+GPU TCB)

```
┌────────────────── outside TCB ──────────────────┐
│ Cloud operator, hypervisor, host OS, PCIe       │
│ switch firmware, orchestrator, logging pipeline │
│                                                 │
│ ┌──────────── inside TCB ────────────────────┐  │
│ │  CPU TEE VM (SEV-SNP or TDX)               │  │
│ │  ├─ Guest kernel + initramfs               │  │
│ │  ├─ NVIDIA GPU driver (host + guest pieces)│  │
│ │  ├─ CUDA runtime                           │  │
│ │  ├─ Inference stack (TRT-LLM / vLLM / …)   │  │
│ │  └─ Model weights (sealed, released after  │  │
│ │     chained attestation)                   │  │
│ │                                            │  │
│ │  ┌──────── H100 GPU (CC mode) ─────────┐   │  │
│ │  │ VBIOS (measured), FW, driver stub   │   │  │
│ │  │ Encrypted HBM, encrypted PCIe       │   │  │
│ │  │ Chip-local keys, attestation RoT    │   │  │
│ │  └─────────────────────────────────────┘   │  │
│ │  Encrypted PCIe channel ↔ GPU              │  │
│ └────────────────────────────────────────────┘  │
│                                                 │
│  Roots of trust:                                │
│   - AMD ARK / Intel TDX Quoting Enclave         │
│   - NVIDIA CC root (chains to NVIDIA RIM / NRAS)│
└─────────────────────────────────────────────────┘
```

Two reports chain together in the verifier:

1. **CPU attestation** (SEV-SNP VCEK-signed or TDX TD Quote) —
   proves the CPU TEE and the measurement of the guest launch.
2. **GPU attestation** (NVIDIA-signed) — proves the GPU is a
   genuine H100 in CC mode with the expected VBIOS and
   driver versions, freshness-bound to a nonce the verifier
   issues.

Both must succeed before the model KEK is released to the enclave.

### 2.3 Enablement checklist (what the operator does)

The exact commands depend on NVIDIA's current driver release; the
authoritative source is the *Hopper Confidential Computing
Deployment Guide*. The flow below is the general shape — treat it
as a checklist, not a copy-paste script:

1. **BIOS / firmware pre-reqs on the host.**
   - CPU TEE enabled (SEV-SNP on AMD EPYC Genoa or TDX on Intel
     Xeon Sapphire Rapids or newer).
   - PCIe passthrough enabled (IOMMU on, ACS, VT-d/AMD-Vi).
2. **GPU VBIOS mode.**
   - H100 VBIOS set to CC-on (sometimes called "CC-Ready" or
     "Protected" mode in NVIDIA tooling). Confirm with
     `nvidia-smi conf-compute -q` on the host (command and exact
     output text depend on the driver release — consult the
     deployment guide).
   - Developer / unsigned-code modes must be off.
3. **Driver stack.**
   - Install NVIDIA's CC-capable data-center driver. Match driver
     major version to what the deployment guide supports.
   - On the host, the driver pins the GPU for passthrough to one
     guest VM.
4. **Guest launch.**
   - Launch the TEE VM with the H100 passed through (SEV-SNP via
     QEMU `-device vfio-pci,host=...`; TDX similarly).
   - Inside the guest, install the matching CC-aware NVIDIA driver
     and CUDA runtime. `nvidia-smi conf-compute -q` inside the
     guest should report that CC mode is active and that the GPU
     has established an encrypted session with the host.
5. **Attestation.**
   - Fetch the GPU attestation report via the CC driver API /
     NRAS CLI. Fetch the CPU report (SEV-SNP
     `SNP_GET_REPORT` or TDX `TDG.MR.REPORT` + Quoting Enclave).
     Bind the same nonce into both reports' `report_data` fields.

### 2.4 Attestation walkthrough (chained)

```
Verifier                         Guest TEE                  H100
  │                                 │                         │
  │  nonce (32B random) ───────────▶│                         │
  │                                 │  CPU report w/          │
  │                                 │  REPORT_DATA =          │
  │                                 │  SHA-512(nonce||pub_s)  │
  │                                 │                         │
  │                                 │  GPU attestation req ──▶│
  │                                 │     (same nonce)        │
  │                                 │                         │
  │                                 │◀─ GPU attestation blob  │
  │                                 │   signed by NVIDIA RoT  │
  │                                 │                         │
  │◀── (cpu_report, gpu_evidence) ──┤                         │
  │                                                           │
  │  Verifier:                                                │
  │  1. Verify CPU report (per exercise-02 for SEV-SNP,       │
  │     or TDX Quote verification for TDX).                   │
  │  2. Verify GPU evidence against NVIDIA RIM / NRAS:        │
  │     - signature chains to the NVIDIA CC root CA,          │
  │     - VBIOS measurement in allow-list,                    │
  │     - CC mode = ON, dev mode = OFF,                       │
  │     - nonce matches what we issued,                       │
  │     - GPU serial / UUID is in scope.                      │
  │  3. Cross-check: both reports bind the SAME nonce.        │
  │  4. Release the model KEK encrypted to `pub_s`.           │
```

NVIDIA provides two paths to verify the GPU evidence:

- **NRAS (NVIDIA Remote Attestation Service).** Verifier sends
  the GPU evidence to NRAS, which checks it and returns a signed
  verdict. Easiest, but the verifier must trust the NRAS operator
  (which is NVIDIA — see §2.6).
- **Local verification.** Verifier runs NVIDIA's open-source
  attestation SDK locally, fetches the NVIDIA root certs and the
  RIM (Reference Integrity Manifest) itself, and performs the
  checks offline. This keeps the verifier operator — not NVIDIA —
  in the TCB for policy decisions.

### 2.5 Reference inference wrapper (illustrative)

```python
# cc_inference.py — illustrative. Keep the real attestation calls behind
# NVIDIA's attestation SDK (see §7); do not hand-roll the GPU parsing.
import secrets, json, pathlib

from verifier import (        # the CPU-side verifier from exercise-02
    parse_report, verify_chain, verify_report_signature, enforce_policy, fetch_cert_chain
)
from nvidia_attestation_sdk import (  # see https://github.com/NVIDIA/nvtrust
    get_gpu_evidence,
    verify_gpu_evidence_local,   # or verify_with_nras
)

def chained_attestation(policy: dict, model_kek_wrapper):
    nonce = secrets.token_bytes(32)
    session_priv, session_pub = make_ephemeral_keypair()
    # --- CPU side ---
    cpu_report = guest_request_cpu_report(nonce, session_pub)
    r = parse_report(cpu_report)
    vcek_der, ark_ask = fetch_cert_chain(policy["product"], r["chip_id"][:8], r["reported_tcb"])
    vcek_pub = verify_chain(vcek_der, ark_ask)
    verify_report_signature(vcek_pub, r)
    enforce_policy(r, policy["cpu"], nonce, session_pub)
    # --- GPU side ---
    gpu_evidence = get_gpu_evidence(nonce=nonce)
    gpu_verdict = verify_gpu_evidence_local(
        evidence=gpu_evidence,
        policy={
            "vbios_allow_list": policy["gpu"]["vbios_allow_list"],
            "driver_floor":     policy["gpu"]["driver_floor"],
            "require_cc_mode":  True,
            "require_dev_mode_off": True,
            "allowed_gpu_uuids": policy["gpu"]["allowed_uuids"],
        },
        expected_nonce=nonce,
    )
    if not gpu_verdict.ok:
        raise RuntimeError(f"GPU attestation failed: {gpu_verdict.reasons}")
    # Cross-check: both reports bind the same nonce
    assert gpu_verdict.nonce == nonce, "GPU/CPU nonce mismatch"
    # --- Release model KEK encrypted to session_pub ---
    wrapped_kek = model_kek_wrapper.wrap_for(session_pub)
    return wrapped_kek, session_priv

def main():
    policy = json.loads(pathlib.Path("policy.json").read_text())
    kek_wrap, priv = chained_attestation(policy, model_kek_wrapper=KMSWrapper())
    # Inside the guest, unwrap the KEK with session_priv, decrypt model weights,
    # and start serving. Refuse to serve if attestation ever fails.
```

Reference policy file:

```yaml
# policy.yaml for chained CPU+GPU attestation
product: Genoa
cpu:
  expected_vmpl: 0
  tcb_floor: {bl: 0x16, tee: 0x00, snp: 0x14, ucode: 0xC4}
  allowed_measurements:
    - "d41d8cd9...placeholder..."    # see exercise-02 for how to compute
gpu:
  vbios_allow_list:
    - "96.00.XX.XX.XX"   # H100 CC-capable VBIOS versions from the NVIDIA deployment guide
  driver_floor: "550.XX.XX"
  allowed_uuids: ["GPU-abcdef12-...-..."]  # per GPU serial / UUID
```

### 2.6 Residual risks (what CC mode does not fix)

| Residual | Why CC mode doesn't fix it | Compensating control |
|---|---|---|
| Vulnerable code inside the guest (CUDA runtime, Python, model server) | Attestation proves image identity, not image safety | CVE scanning, measurement rotation on patch, minimal container image |
| Output leakage / model-inversion / MIA (MITRE ATLAS `AML.T0024`) | CC mode protects data in use; the API still returns outputs | Output DP, rate limiting, policy filters [[mod-105-memorisation-extraction-and-mia]] |
| Supply-chain compromise of model weights (MITRE ATLAS `AML.T0010`) | The enclave happily runs a backdoored model | Signed training provenance (SLSA / in-toto), reproducible training |
| GPU side channels (power / timing / microarchitectural) | Documented as out of scope for CC mode | Dedicated host, no co-tenancy on the server (CC mode enforces this), monitor performance counters |
| MIG multi-tenancy | Unavailable in H100 CC mode | Architect single-tenant; schedule per workload |
| Trust in NRAS operator | NRAS verdict is authoritative only if you trust NVIDIA's service | Prefer local verification; if NRAS is used, log the verdict hash and cross-check periodically |
| Bandwidth / latency tax | PCIe encryption has a performance cost | Measure for your workload; budget accordingly in §cost-memo (see [[exercise-07-confidential-ai-risk-and-cost-memo]]) |
| Rollback of sealed model state | CC attestation is per-session; it doesn't carry monotonic state | Guest enforces monotonic counters on cached state |

## 3. Validation steps

1. **Pre-flight on the host.** `nvidia-smi conf-compute -q`
   reports CC-ready; `dmesg | grep -i 'SEV-SNP\|TDX'` confirms
   the CPU TEE is active.
2. **Launch the guest TEE VM** with H100 passthrough. Inside
   the guest, confirm `nvidia-smi conf-compute -q` reports CC
   active and an encrypted session established. (Exact field
   names vary with driver; consult the deployment guide.)
3. **Issue a chained attestation.** Run the wrapper above with a
   freshly-generated nonce. Verifier logs should show both
   reports binding the same nonce.
4. **Negative tests.**
   - Flip a byte in the GPU evidence → local verification must
     reject.
   - Downgrade the VBIOS allow-list below the reported version →
     must reject.
   - Omit the nonce from the GPU report → must reject.
   - Flip the `dev_mode_off` requirement → confirm the policy
     rejects reports from a dev-mode GPU.
5. **Run a tiny inference** (e.g., Llama-2-7B via `vllm` or TRT-LLM)
   and confirm it works end-to-end inside the attested VM. Do not
   quote absolute throughput numbers in the solution; measurement
   of the CC tax is out of scope here (see
   [[exercise-07-confidential-ai-risk-and-cost-memo]]).
6. **Teardown test.** Reboot the guest and reconfirm the model KEK
   is only released after a fresh chained attestation.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Trust-boundary diagram chains CPU TEE + H100 and keeps hypervisor out | 1 | §2.2 |
| Enablement checklist names VBIOS CC mode, driver floor, passthrough + guest driver | 1 | §2.3 |
| Attestation walkthrough binds the SAME nonce into CPU and GPU reports | 2 | Most common failure mode |
| Reference wrapper (or equivalent) runs both verifications before releasing the model KEK | 2 | — |
| Policy file lists VBIOS allow-list, driver floor, allowed UUIDs, CC-mode-required, dev-mode-off | 1 | — |
| Negative tests pass (bad nonce, bad VBIOS, dev-mode on, byte-flip) | 1 | — |
| Residual risks link to [[exercise-01-tee-trust-model-and-threat-map]] and MITRE ATLAS | 1 | — |
| MIG / co-tenancy limitation explicitly acknowledged | 1 | — |

Automatic failures:

- Running inference on a stock H100 without CC mode and calling
  it "confidential."
- Verifying only one of the two reports.
- Not binding a nonce (replayable GPU attestation).
- Trusting an NRAS verdict without logging and cross-checking.
- Quoting an absolute "X % overhead" figure that is not tied to
  a published NVIDIA benchmark or a learner-run measurement
  <!-- citation-pending: cite specific NVIDIA-published H100 CC-mode overhead benchmarks for the learner's workload shape before quoting -->.

## 5. Common mistakes

- **Treating "GPU passthrough" as confidential computing.** PCIe
  passthrough gives the guest control of the device; it does
  nothing about encryption on the bus or in HBM. Only CC mode
  does that.
- **Forgetting the VBIOS allow-list.** A GPU with a vulnerable
  VBIOS still attests validly as "an H100 in CC mode." The
  policy must gate on VBIOS version.
- **Verifying only the CPU half.** A valid SEV-SNP report tells
  you nothing about what is in HBM. The verifier must demand GPU
  evidence too, with the same nonce.
- **Trusting NRAS blindly.** NRAS is operated by NVIDIA. If the
  data owner wants NVIDIA out of the per-session TCB, local
  verification is required. "NRAS says OK" is not the same as
  "we verified."
- **Expecting MIG.** H100 CC mode is single-tenant. If the cost
  model assumes 1/7 of a GPU, the memo
  ([[exercise-07-confidential-ai-risk-and-cost-memo]]) must be
  rewritten.
- **Ignoring the driver inside the TCB.** The NVIDIA data-center
  driver is part of the guest TCB. Treat it like any other
  privileged dependency: pin versions, rotate on CVE, track in
  the measurement allow-list.
- **Over-claiming on output privacy.** CC mode does nothing about
  what the model *says*. Output leakage, membership inference,
  and model inversion remain as if the GPU were a normal GPU —
  mitigate them separately
  [[mod-105-memorisation-extraction-and-mia]].

## 6. References

- NVIDIA. *Hopper Confidential Computing Deployment Guide.*
  <https://docs.nvidia.com/confidential-computing-deployment-guide.pdf>
  (also linked from
  <https://developer.nvidia.com/blog/confidential-computing-on-h100-gpus-for-secure-and-trustworthy-ai/>).
- NVIDIA. *Confidential Computing on NVIDIA Hopper H100 GPUs —
  developer blog.*
  <https://developer.nvidia.com/blog/confidential-computing-on-h100-gpus-for-secure-and-trustworthy-ai/>
- NVIDIA. *NVIDIA Attestation Service / Remote Attestation (NRAS)
  — documentation and SDK.*
  <https://docs.nvidia.com/attestation/index.html> and
  <https://github.com/NVIDIA/nvtrust> (open-source attestation SDK,
  practitioner reference).
- NVIDIA H100 Tensor Core GPU — product datasheet (CC mode
  entry). <https://www.nvidia.com/en-us/data-center/h100/>
- AMD. *SEV Secure Nested Paging Firmware ABI Specification.*
  <https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/specifications/56860.pdf>
- Intel. *Intel Trust Domain Extensions (TDX) — Architecture
  Specifications.*
  <https://www.intel.com/content/www/us/en/developer/articles/technical/intel-trust-domain-extensions.html>
- Confidential Computing Consortium. *Common Terminology for
  Confidential Computing.* <https://confidentialcomputing.io/>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
