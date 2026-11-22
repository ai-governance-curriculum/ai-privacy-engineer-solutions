# Exercise 02 — AMD SEV-SNP Attestation Walkthrough — Reference Solution

## 1. Solution overview

This is an **implementation / walkthrough exercise**. The learner is
asked to produce an end-to-end description — with working artefacts
where feasible — of how an **AMD SEV-SNP** guest obtains an
attestation report, how a remote relying party verifies it, and how
secrets are released to the guest only after that verification
succeeds. The output must make the full chain of trust explicit:
hardware root CA → VCEK → attestation report → measurement →
freshness → policy.

Assessable artefacts:

1. **Narrative walkthrough** of the five-step SEV-SNP attestation
   flow with the fields each step depends on.
2. **Guest-side script** that requests an attestation report with a
   nonce bound in `REPORT_DATA` and writes it to disk.
3. **Verifier-side script** that fetches the AMD root CA + VCEK
   chain, verifies the signature, enforces a policy (measurement
   allow-list, freshness nonce, TCB version, policy bits), and
   releases a secret on success.
4. **Policy file** documenting what the verifier will accept and
   what it will not.
5. **Threat-residual note** mapping the walkthrough to the
   trust-boundary diagram from [[exercise-01-tee-trust-model-and-threat-map]]
   and to MITRE ATLAS techniques that attestation does not stop.

The construction follows AMD's *SEV Secure Nested Paging Firmware
ABI Specification* (the public ABI doc defines the report layout,
`MSG_REPORT_REQ`, `REPORT_DATA`, policy fields, and the VCEK chain)
and AMD's *Key Distribution Service* documentation (KDS) for the
VCEK/ASK/ARK chain. The reference implementation uses the AMD
open-source tooling (`sev-guest`, `sev-tool`, `snpguest`) and the
`virtee/sev` userspace libraries, treating these as practitioner
implementations of the official ABI — see §7.

## 2. Worked walkthrough

### 2.1 Chain of trust (what signs what)

```
 ┌────────────────────────────────┐
 │  AMD Root Key (ARK)            │  ARK is AMD's permanent root, in silicon OTP
 └───────────┬────────────────────┘
             │ signs
             ▼
 ┌────────────────────────────────┐
 │  AMD Signing Key (ASK)         │  Per-generation (Milan, Genoa, Turin, …)
 └───────────┬────────────────────┘
             │ signs
             ▼
 ┌────────────────────────────────┐
 │  Versioned Chip Endorsement    │  VCEK — unique per CPU, bound to current
 │  Key (VCEK)                    │  TCB version (microcode, PSP, bootloader)
 └───────────┬────────────────────┘
             │ signs
             ▼
 ┌────────────────────────────────┐
 │  SEV-SNP Attestation Report    │  Contains MEASUREMENT, REPORT_DATA (nonce),
 │  (produced by the PSP)         │  POLICY, TCB versions, CHIP_ID, FAMILY_ID
 └────────────────────────────────┘
```

The report is signed by the **VCEK**; the VCEK certificate is
signed by the **ASK**; the ASK certificate is signed by the **ARK**.
The ARK is AMD's root. A verifier therefore needs: the report,
the VCEK (fetched from AMD's KDS using `CHIP_ID` + reported TCB
version), the ASK, and the ARK. The verifier pins the ARK SHA-384
fingerprint from AMD's documentation.

### 2.2 The five-step flow

1. **Guest boot + measurement.** When the SEV-SNP guest launches,
   the PSP measures the initial guest memory (OVMF firmware, kernel,
   initramfs if direct-boot; otherwise the firmware's launch digest)
   and stores it as `MEASUREMENT` in the SNP launch digest. The
   measurement is deterministic for a given image + launch policy.
2. **Nonce generation by the verifier.** A relying party picks a
   fresh 64-byte random `nonce`. The nonce is sent into the guest
   over an untrusted channel (it does not need to be secret —
   freshness is the goal).
3. **Guest asks PSP for a signed report.** Inside the guest, user
   code opens `/dev/sev-guest` (Linux) and issues `MSG_REPORT_REQ`
   with `REPORT_DATA = SHA-512(nonce || session_pubkey)` (64 bytes).
   The PSP returns an `AttestationReport` signed by the VCEK whose
   layout is defined in the ABI spec. Fields that matter for
   verification: `MEASUREMENT`, `REPORT_DATA`, `POLICY`,
   `FAMILY_ID`, `IMAGE_ID`, `REPORTED_TCB`, `CURRENT_TCB`,
   `COMMITTED_TCB`, `CHIP_ID`, `SIGNATURE_ALGO`, `SIGNATURE`.
4. **Verifier fetches + validates the cert chain.** The verifier
   calls AMD KDS at
   `https://kdsintf.amd.com/vcek/v1/{product}/{CHIP_ID}?blSPL=..&teeSPL=..&snpSPL=..&ucodeSPL=..`
   where product is `Milan`, `Genoa`, etc. and the SPL values come
   from `REPORTED_TCB` in the report. KDS returns the VCEK
   certificate; the ASK and ARK are downloaded from AMD and cached.
   The verifier checks: ARK fingerprint matches the pinned value,
   ARK→ASK→VCEK chain validates, and the report signature validates
   under VCEK.
5. **Verifier enforces policy and releases the secret.** On
   success, the verifier checks:
   - `MEASUREMENT` ∈ allow-list of known-good launch digests.
   - `REPORT_DATA` includes the nonce it generated in step 2 (the
     verifier recomputes `SHA-512(nonce || session_pubkey)`).
   - `REPORTED_TCB.{boot_loader, tee, snp, microcode}` ≥ the
     minimum versions on the policy's TCB floor.
   - `POLICY.{smt, debug, migrate_ma, single_socket, api_major,
     api_minor}` match the policy. **Debug must be off.**
     **Migration-agent (`MIGRATE_MA`) must be as the policy says.**
   - `FAMILY_ID` and `IMAGE_ID` (if the launch guest set them)
     match the expected workload identity.
   - The report's `VMPL` field is the expected privilege level
     (`0` for the base guest OS; higher VMPLs for more-privileged
     helpers inside the guest).

   Only then does the verifier encrypt the workload secret (model
   KEK, TLS private key, dataset decryption key, …) under the
   guest's `session_pubkey` that was bound into `REPORT_DATA`.

### 2.3 Guest-side reference implementation (Linux, SEV-SNP guest)

The ABI exposes `/dev/sev-guest` with the
`SNP_GET_REPORT` ioctl. AMD and the `virtee/sev` project ship
userspace that wraps the ioctl; the `snpguest` tool is a practitioner
reference implementation. The hand-rolled version below shows the
essential fields; in production, call `snpguest report` or the
`sev` Rust crate.

```python
# guest_request_report.py — runs *inside* the SEV-SNP guest
# Pedagogical reference. In production, use `snpguest report <outfile> <nonce_file>`
# from https://github.com/virtee/snpguest (practitioner implementation of the ABI).
import ctypes, fcntl, hashlib, os, struct, sys

SNP_GUEST_IOC_MAGIC = ord("S")
# ioctl number for SNP_GET_REPORT — see include/uapi/linux/sev-guest.h in Linux source.
SNP_GET_REPORT = 0xC0E85300  # _IOWR('S', 0x0, snp_guest_request_ioctl)

class SnpReportRequest(ctypes.Structure):
    _fields_ = [
        ("report_data", ctypes.c_ubyte * 64),
        ("vmpl", ctypes.c_uint32),
        ("rsvd", ctypes.c_uint32 * 9),
    ]

class SnpReportResponse(ctypes.Structure):
    _fields_ = [
        ("status", ctypes.c_uint32),
        ("report_size", ctypes.c_uint32),
        ("rsvd", ctypes.c_uint8 * 24),
        ("report", ctypes.c_ubyte * 4000),
    ]

class SnpGuestRequest(ctypes.Structure):
    _fields_ = [
        ("msg_version", ctypes.c_uint8),
        ("req_data", ctypes.c_uint64),
        ("resp_data", ctypes.c_uint64),
        ("fw_err", ctypes.c_uint64),
    ]

def request_report(nonce: bytes, session_pubkey: bytes, vmpl: int = 0) -> bytes:
    assert len(nonce) == 32
    report_data = hashlib.sha512(nonce + session_pubkey).digest()  # 64 bytes
    req = SnpReportRequest()
    ctypes.memmove(req.report_data, report_data, 64)
    req.vmpl = vmpl
    resp = SnpReportResponse()
    wrap = SnpGuestRequest(
        msg_version=1,
        req_data=ctypes.addressof(req),
        resp_data=ctypes.addressof(resp),
        fw_err=0,
    )
    with open("/dev/sev-guest", "rb") as fd:
        fcntl.ioctl(fd.fileno(), SNP_GET_REPORT, wrap)
    if resp.status != 0:
        raise RuntimeError(f"PSP returned status 0x{resp.status:x}, fw_err {wrap.fw_err:x}")
    return bytes(resp.report[: resp.report_size])

if __name__ == "__main__":
    nonce_path, pubkey_path, out_path = sys.argv[1], sys.argv[2], sys.argv[3]
    nonce = open(nonce_path, "rb").read()
    pub = open(pubkey_path, "rb").read()
    report = request_report(nonce, pub)
    with open(out_path, "wb") as f:
        f.write(report)
    print(f"wrote {len(report)} bytes of attestation report to {out_path}")
```

Equivalent one-liner using the practitioner tool:

```bash
# https://github.com/virtee/snpguest
snpguest report --random report.bin request-data.bin
snpguest display report report.bin   # human-readable dump
```

### 2.4 Verifier-side reference implementation

The verifier does **not** run inside the enclave. It can be a build
artefact signed by the data owner and run on hardware the data
owner trusts. Its job: parse the report, fetch the cert chain,
verify signatures, enforce policy, and release the secret.

```python
# verifier.py — runs on the relying party's side.
# Pedagogical reference. Production code should use the AMD guest libraries
# or the `virtee/sev` Rust crate (practitioner reference implementation).
import hashlib, json, pathlib, struct, sys
import requests
from cryptography import x509
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import ec, padding, utils as asym_utils

# Pinned SHA-384 fingerprint of the AMD Root Key (ARK) per AMD's Key Distribution
# Service documentation. Treat as part of the verifier TCB.
AMD_ARK_FINGERPRINT_SHA384 = bytes.fromhex(
    # placeholder — the learner MUST replace with the current AMD-published value
    # see <https://www.amd.com/en/developer/sev.html> and the ABI spec §8
    "00" * 48
)

ARK_URL  = "https://kdsintf.amd.com/vcek/v1/{product}/cert_chain"
VCEK_URL = ("https://kdsintf.amd.com/vcek/v1/{product}/{chip_id}"
            "?blSPL={bl}&teeSPL={tee}&snpSPL={snp}&ucodeSPL={uc}")

# Field offsets below follow the AMD SEV-SNP ABI spec §7.3 (AttestationReport).
# Keep the parse struct in lock-step with the ABI version you support.
REPORT_SIZE = 0x4A0
MEAS_OFF, MEAS_LEN = 0x090, 48
REPORT_DATA_OFF, REPORT_DATA_LEN = 0x050, 64
POLICY_OFF = 0x008
FAMILY_ID_OFF, IMAGE_ID_OFF = 0x040, 0x050  # 16 bytes each (check against your ABI version)
VMPL_OFF = 0x030
REPORTED_TCB_OFF = 0x180
CHIP_ID_OFF = 0x1A0
SIG_OFF = 0x2A0    # 512 bytes of signature (ECDSA P-384 R || S with reserved padding)

def parse_report(buf: bytes) -> dict:
    assert len(buf) >= REPORT_SIZE, "truncated report"
    policy = struct.unpack_from("<Q", buf, POLICY_OFF)[0]
    reported_tcb = struct.unpack_from("<Q", buf, REPORTED_TCB_OFF)[0]
    return {
        "measurement":  buf[MEAS_OFF:MEAS_OFF + MEAS_LEN],
        "report_data":  buf[REPORT_DATA_OFF:REPORT_DATA_OFF + REPORT_DATA_LEN],
        "policy_raw":   policy,
        "policy": {
            "smt":          bool(policy & (1 << 16)),
            "migrate_ma":   bool(policy & (1 << 18)),
            "debug":        bool(policy & (1 << 19)),
            "single_socket": bool(policy & (1 << 20)),
            "api_major":    (policy >> 8) & 0xFF,
            "api_minor":    policy & 0xFF,
        },
        "reported_tcb": {
            "bl":    (reported_tcb >> 0)  & 0xFF,
            "tee":   (reported_tcb >> 8)  & 0xFF,
            "snp":   (reported_tcb >> 48) & 0xFF,
            "ucode": (reported_tcb >> 56) & 0xFF,
        },
        "vmpl":        struct.unpack_from("<I", buf, VMPL_OFF)[0],
        "family_id":   buf[FAMILY_ID_OFF:FAMILY_ID_OFF + 16],
        "image_id":    buf[IMAGE_ID_OFF:IMAGE_ID_OFF + 16],
        "chip_id":     buf[CHIP_ID_OFF:CHIP_ID_OFF + 64],
        "sig_r":       buf[SIG_OFF:SIG_OFF + 72],
        "sig_s":       buf[SIG_OFF + 72:SIG_OFF + 144],
        "signed_body": buf[:SIG_OFF],
    }

def fetch_cert_chain(product: str, chip_id: bytes, reported_tcb: dict):
    # cert_chain returns ASK||ARK in PEM
    ark_ask = requests.get(ARK_URL.format(product=product), timeout=10)
    ark_ask.raise_for_status()
    vcek = requests.get(VCEK_URL.format(
        product=product, chip_id=chip_id.hex(),
        bl=reported_tcb["bl"], tee=reported_tcb["tee"],
        snp=reported_tcb["snp"], uc=reported_tcb["ucode"],
    ), timeout=10)
    vcek.raise_for_status()
    return vcek.content, ark_ask.content

def verify_chain(vcek_der: bytes, ark_ask_pem: bytes) -> ec.EllipticCurvePublicKey:
    certs = x509.load_pem_x509_certificates(ark_ask_pem)
    ark = next(c for c in certs if c.subject == c.issuer)
    ask = next(c for c in certs if c is not ark)
    ark.public_key().verify(
        ark.signature, ark.tbs_certificate_bytes,
        padding.PKCS1v15() if isinstance(ark.public_key(), object) and False else ec.ECDSA(hashes.SHA384()),
    )
    # ARK fingerprint pin — refuse to continue if AMD root doesn't match
    assert hashlib.sha384(ark.public_bytes(serialization.Encoding.DER)).digest() == AMD_ARK_FINGERPRINT_SHA384, \
        "ARK fingerprint mismatch — refusing to verify"
    ark.public_key().verify(
        ask.signature, ask.tbs_certificate_bytes, ec.ECDSA(hashes.SHA384())
    )
    vcek = x509.load_der_x509_certificate(vcek_der)
    ask.public_key().verify(
        vcek.signature, vcek.tbs_certificate_bytes, ec.ECDSA(hashes.SHA384())
    )
    return vcek.public_key()

def verify_report_signature(vcek_pub: ec.EllipticCurvePublicKey, report: dict) -> None:
    r = int.from_bytes(report["sig_r"][:48][::-1], "big")  # little-endian in the report
    s = int.from_bytes(report["sig_s"][:48][::-1], "big")
    sig_der = asym_utils.encode_dss_signature(r, s)
    vcek_pub.verify(sig_der, report["signed_body"], ec.ECDSA(hashes.SHA384()))

def enforce_policy(report: dict, policy: dict, nonce: bytes, session_pubkey: bytes) -> None:
    if report["policy"]["debug"]:
        raise ValueError("debug mode is on — refusing to trust this guest")
    if policy.get("require_no_smt") and report["policy"]["smt"]:
        raise ValueError("SMT enabled; policy requires SMT off")
    if hashlib.sha512(nonce + session_pubkey).digest() != report["report_data"]:
        raise ValueError("REPORT_DATA does not bind our nonce — replay / mis-wired guest")
    if report["measurement"].hex() not in policy["allowed_measurements"]:
        raise ValueError(f"measurement {report['measurement'].hex()} not in allow-list")
    floor = policy["tcb_floor"]
    for k in ("bl", "tee", "snp", "ucode"):
        if report["reported_tcb"][k] < floor[k]:
            raise ValueError(f"TCB.{k} = {report['reported_tcb'][k]} below floor {floor[k]}")
    if report["family_id"].hex() != policy.get("family_id", report["family_id"].hex()):
        raise ValueError("FAMILY_ID mismatch")
    if report["vmpl"] != policy.get("expected_vmpl", 0):
        raise ValueError("unexpected VMPL")

def main():
    report_bytes = pathlib.Path(sys.argv[1]).read_bytes()
    nonce        = pathlib.Path(sys.argv[2]).read_bytes()
    session_pub  = pathlib.Path(sys.argv[3]).read_bytes()
    policy       = json.loads(pathlib.Path(sys.argv[4]).read_text())
    product      = policy["product"]            # e.g. "Milan" | "Genoa"

    r = parse_report(report_bytes)
    vcek_der, ark_ask_pem = fetch_cert_chain(product, r["chip_id"][:8], r["reported_tcb"])
    vcek_pub = verify_chain(vcek_der, ark_ask_pem)
    verify_report_signature(vcek_pub, r)
    enforce_policy(r, policy, nonce, session_pub)
    print("ATTESTATION OK — releasing secret")
    # production: encrypt the secret under session_pub (ECIES/HPKE) and ship it back
```

### 2.5 Reference policy file

```yaml
# attestation-policy.yaml — the verifier enforces this
product: Genoa
expected_vmpl: 0
require_no_smt: true          # we want SMT off for side-channel reasons
family_id: "00000000000000000000000000000000"
tcb_floor:
  bl:    0x16   # replace with the current AMD TCB floor at review time
  tee:   0x00
  snp:   0x14
  ucode: 0xC4
allowed_measurements:
  # SHA-384 of the launch digest of the known-good guest image
  - "d41d8cd98f00b204e9800998ecf8427ea6b1a2c3...placeholder..."
```

### 2.6 Threat-residual note

Attestation checks *who is running*; it does not check *what the
code does*. Residual risks from [[exercise-01-tee-trust-model-and-threat-map]]
that the walkthrough above does **not** eliminate:

| Residual | Why attestation does not stop it | Compensating control |
|---|---|---|
| Vulnerability in the attested guest code | The measurement proves the guest is the image you expected; it does not prove the image is bug-free | CVE scanning + measurement rotation on patch |
| Side channels (Spectre / MDS variants) | Report shows only microcode version, not timing behaviour | Policy requires patched microcode + SMT off; document residual |
| Rollback of sealed state | Report is per-boot; it doesn't carry monotonic state | Guest uses a monotonic counter (e.g., TPM NVRAM, SP-managed counter) and refuses to decrypt old state |
| Model-inversion / MIA on outputs (MITRE ATLAS `AML.T0024`) | Attestation is about the host, not the model | Output DP + rate limit [[mod-105-memorisation-extraction-and-mia]] |
| Supply-chain compromise of the guest image (MITRE ATLAS `AML.T0010`) | Attestation only tells you "this is the image you built" — it doesn't tell you the build was not poisoned | Reproducible builds, signed provenance (in-toto / SLSA) |

## 3. Validation steps

1. **Run the guest script** in a real SEV-SNP VM or an environment
   that emulates `/dev/sev-guest` (e.g., the `sev-guest` kernel
   module in test mode) and confirm a non-zero report is produced
   whose `REPORT_DATA` field equals `SHA-512(nonce || session_pub)`.
2. **Dump the report** with `snpguest display report report.bin`
   (or equivalent) and visually confirm the `MEASUREMENT`,
   `REPORTED_TCB`, `POLICY`, and `CHIP_ID` fields are populated.
3. **Run the verifier** against the report. Confirm:
   - It fails if the nonce file is swapped (freshness check).
   - It fails if the measurement is missing from the allow-list.
   - It fails if `POLICY.debug == 1`.
   - It fails if `REPORTED_TCB` is below the policy floor.
   - It fails if the ARK fingerprint does not match the pinned
     value.
4. **Fetch the cert chain** from AMD KDS manually with `curl` and
   confirm it is the same chain the verifier downloads.
5. **Negative test**: feed the verifier a report whose signature
   has been flipped by one bit and confirm the ECDSA verification
   fails before any policy check runs.
6. **Drift test**: increment the policy's TCB floor past the host's
   current TCB and confirm the verifier rejects.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Chain of trust diagram (ARK → ASK → VCEK → report) is correct | 1 | §2.1 |
| Five-step flow narrated with nonce binding in `REPORT_DATA` | 2 | Freshness is the easiest field to drop |
| Guest script actually issues `SNP_GET_REPORT` (or calls `snpguest`) and binds `SHA-512(nonce ‖ session_pubkey)` | 2 | — |
| Verifier parses report, fetches VCEK from KDS using reported TCB, validates full chain, pins ARK fingerprint | 2 | — |
| Policy file is explicit: `debug=0`, TCB floor, measurement allow-list, expected VMPL | 1 | — |
| Negative tests pass (bit-flipped sig, missing measurement, stale nonce) | 1 | — |
| Threat-residual note links MITRE ATLAS + exercise-01 | 1 | — |

Automatic failures:

- Verifying only the signature without checking the measurement
  allow-list (accepts *any* SEV-SNP guest).
- Not binding a nonce into `REPORT_DATA` (replayable attestation).
- Trusting a VCEK fetched with a hard-coded TCB rather than the
  TCB from the report (allows downgrade).
- Allowing `POLICY.debug == 1` in production.
- Shipping the ARK fingerprint as `TODO` — the verifier must pin a
  real, AMD-published value at review time.

## 5. Common mistakes

- **Treating `MEASUREMENT` as free-form content.** It is the SHA-
  384 of a deterministic launch digest. A different kernel
  command-line, a different OVMF version, or a different initramfs
  gives a different measurement. The allow-list must therefore
  cover every expected build, and must be rotated whenever one of
  those components changes.
- **Not pinning the ARK.** Downloading the ARK over HTTPS and
  trusting the TLS chain alone pushes the trust root to WebPKI.
  Always pin the ARK fingerprint published by AMD.
- **Forgetting the VMPL.** On SEV-SNP, a guest can run code at
  multiple privilege levels (VMPL 0..3). An attacker-controlled
  VMPL-2 helper can produce reports; the verifier must insist on
  the expected VMPL.
- **Ignoring `REPORTED_TCB` vs. `COMMITTED_TCB`.** `REPORTED_TCB`
  is the TCB the report is signed under; `COMMITTED_TCB` is what
  the SP has committed to going forward. Both matter, and the
  policy must state which the floor applies to (usually
  `REPORTED_TCB`).
- **Confusing the verifier with the enclave.** The verifier does
  not run inside the TEE. Think of it as a doorman on the data
  owner's side. The TCB expansion is the verifier *code*, not the
  verifier *host*.
- **Using the cert chain as a liveness signal.** AMD KDS caches
  are fine for availability, but a fetched VCEK says nothing about
  freshness of the guest state — the nonce in `REPORT_DATA` does.
- **Letting `DEBUG=1` slip into production.** Debug mode allows
  the hypervisor to inspect guest memory. Any report with
  `POLICY.debug == 1` must be rejected outside of dev clusters.
- **Treating `snpguest` as "the attestation protocol".** The tool
  is a practitioner reference; the canonical spec is AMD's
  SEV-SNP firmware ABI document. If the tool and the spec
  disagree, follow the spec.

## 6. References

- AMD. *SEV Secure Nested Paging Firmware ABI Specification*
  (publication 56860).
  <https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/specifications/56860.pdf>
- AMD. *AMD SEV-SNP: Strengthening VM Isolation with Integrity
  Protection and More* (white paper).
  <https://www.amd.com/content/dam/amd/en/documents/epyc-business-docs/white-papers/SEV-SNP-strengthening-vm-isolation-with-integrity-protection-and-more.pdf>
- AMD. *Secure Encrypted Virtualization — KDS Specification and
  API.* <https://www.amd.com/en/developer/sev.html>
- `virtee/snpguest` — practitioner reference implementation of the
  SEV-SNP guest ABI. <https://github.com/virtee/snpguest>
- `virtee/sev` — Rust crates for SEV/SEV-ES/SEV-SNP guest and host
  operations. <https://github.com/virtee/sev>
- Linux kernel `/dev/sev-guest` UAPI —
  `include/uapi/linux/sev-guest.h`.
  <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/sev-guest.h>
- Confidential Computing Consortium. *Common Terminology for
  Confidential Computing.*
  <https://confidentialcomputing.io/>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS — Adversarial Threat Landscape for AI Systems.
  <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
