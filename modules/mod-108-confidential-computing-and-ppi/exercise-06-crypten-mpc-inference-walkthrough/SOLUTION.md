# Exercise 06 — CrypTen MPC Inference Walkthrough — Reference Solution

## 1. Solution overview

This is an **implementation / walkthrough exercise**. The learner
uses [CrypTen](https://github.com/facebookresearch/CrypTen)
(a research-grade secure multi-party computation (MPC) library
built on PyTorch) to run **two-party inference** where neither
party sees the other's secret: one party holds the model, the
other holds the input, and both learn only the output (or, in the
asymmetric variant, only one of them does).

Assessable artefacts:

1. **Reference implementation** that splits a small plaintext
   PyTorch model into an MPC computation across two parties
   (ALICE holds the model, BOB holds the input) and runs the
   forward pass under secret sharing.
2. **Correctness check** — MPC output matches the plaintext
   forward pass to the fixed-point precision CrypTen uses.
3. **Threat / security model note** stating what CrypTen does
   and does not promise: semi-honest / passive security in the
   two-party setting, **no** malicious-party integrity, **no**
   output privacy once the designated recipient decrypts.
4. **Benchmark table** — latency and bytes-on-the-wire, learner-
   measured on their own hardware.
5. **Residual-risk note** mapping the walkthrough to MITRE ATLAS,
   OWASP ML Top 10, and NIST AI RMF; and comparing MPC's trust
   pattern against the HE pattern from
   [[exercise-05-tenseal-he-inference-on-a-small-model]] and the
   TEE pattern from [[exercise-02-amd-sev-snp-attestation-walkthrough]].

The construction follows CrypTen's official documentation and
tutorials; CrypTen is a practitioner implementation of secret-
sharing based secure computation (arithmetic SS for linear ops,
boolean SS for comparisons, with online-offline split via Beaver
triples). Threat framing is kept consistent with NIST AI RMF and
MITRE ATLAS.

## 2. Worked walkthrough

### 2.1 What CrypTen gives you, in one paragraph

CrypTen represents every tensor as a **secret-sharing** across
parties: each party holds a random share such that the shares
sum (mod a fixed-point ring) to the plaintext value, but any
single share reveals nothing. Linear operations (add, matmul,
conv) can be done locally on shares plus some lightweight
communication; non-linear operations (ReLU, comparisons,
reciprocals, exponentials) are built from protocols on top of
Beaver-style multiplication triples and secret-shared sign
bits. CrypTen targets **semi-honest** adversaries: the protocol
is private so long as parties follow the protocol honestly
(they may *try* to learn something from the transcript, but may
not deviate). It does **not** provide integrity against a
malicious party, and it does not hide the model architecture —
only the parameter values and the input.

### 2.2 Model and party setup

Scope: use the same small model as
[[exercise-05-tenseal-he-inference-on-a-small-model]] or a plain
2-layer MLP on MNIST. Two parties:

| Party | Holds | Learns (after protocol) |
|---|---|---|
| ALICE (rank 0) | Plaintext model weights | Nothing (symmetric variant) or Nothing (asymmetric: BOB learns) |
| BOB (rank 1) | Plaintext input | The predicted class (asymmetric) or nothing (symmetric) |

In the asymmetric reveal variant (most common), BOB receives the
plaintext output; ALICE sees only the ciphertext transcript.

### 2.3 Reference implementation

```python
# mpc_inference.py — CrypTen two-party inference.
# Requires `crypten` and matching torch. CrypTen is a research-grade
# library; see §5 for security caveats.
import os
import time
from contextlib import contextmanager

import crypten
import crypten.communicator as comm
import torch
import torch.nn as nn

crypten.init()                              # starts the CrypTen context
torch.set_num_threads(1)                    # deterministic timing

ALICE = 0                                   # model party (holds weights)
BOB   = 1                                   # data party (holds inputs)


class TinyMLP(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(196, 32)
        self.fc2 = nn.Linear(32, 10)

    def forward(self, x):
        return self.fc2(torch.relu(self.fc1(x)))  # CrypTen implements ReLU for us


def train_plaintext() -> TinyMLP:
    # Train on downsampled MNIST (see exercise-05 §2.4 for the same recipe).
    from torchvision import datasets, transforms

    def downsample(img):
        return nn.functional.avg_pool2d(img, 2).flatten()

    tfm = transforms.Compose([transforms.ToTensor(), transforms.Lambda(downsample)])
    train = datasets.MNIST(".data", train=True, download=True, transform=tfm)
    loader = torch.utils.data.DataLoader(train, batch_size=128, shuffle=True)
    net = TinyMLP()
    opt = torch.optim.Adam(net.parameters(), 1e-3)
    loss_fn = nn.CrossEntropyLoss()
    for _ in range(2):
        for x, y in loader:
            opt.zero_grad()
            loss_fn(net(x), y).backward()
            opt.step()
    return net


def mpc_forward(plain_model: TinyMLP, x_plain: torch.Tensor) -> torch.Tensor:
    """
    Two-party MPC forward pass.

    Each party invokes this (CrypTen launches the two processes via
    `crypten.mpc.multiprocess_wrap`). Only the rank-matching party's
    `x_plain` / model is the real one; the other party supplies a
    dummy of the right shape.
    """
    rank = comm.get().get_rank()

    # 1) Encrypt the model from ALICE's side; shape is public.
    model_enc = crypten.nn.from_pytorch(plain_model, torch.empty(1, 196)).encrypt(src=ALICE)
    model_enc.eval()

    # 2) Encrypt the input from BOB's side.
    x_enc = crypten.cryptensor(x_plain, src=BOB)

    # 3) Forward pass on encrypted tensors.
    t0 = time.perf_counter()
    out_enc = model_enc(x_enc)
    t1 = time.perf_counter()

    # 4) Reveal only to BOB (asymmetric reveal). ALICE gets nothing.
    out_plain = out_enc.get_plain_text(dst=BOB)
    if rank == BOB:
        print(f"MPC forward wall-time: {t1 - t0:.3f} s")
        return out_plain
    return None


def run():
    net = train_plaintext() if comm.get().get_rank() == ALICE else TinyMLP()
    # BOB supplies the input; ALICE supplies a shape-only dummy.
    x = load_bob_input() if comm.get().get_rank() == BOB else torch.zeros(1, 196)

    out = mpc_forward(net, x)
    if comm.get().get_rank() == BOB:
        pred = out.argmax(dim=1)
        print(f"BOB learned prediction: {pred.tolist()}")


def load_bob_input() -> torch.Tensor:
    # In a real deployment BOB would read from its local store.
    from torchvision import datasets, transforms
    def downsample(img):
        return nn.functional.avg_pool2d(img, 2).flatten()
    tfm = transforms.Compose([transforms.ToTensor(), transforms.Lambda(downsample)])
    test = datasets.MNIST(".data", train=False, download=True, transform=tfm)
    x, _ = test[0]
    return x.unsqueeze(0)


if __name__ == "__main__":
    # Use CrypTen's multiprocess launcher: two processes, same script.
    crypten.mpc.run_multiprocess(run)()
```

### 2.4 Correctness check

Alongside the MPC forward, run the plaintext forward (done on
ALICE's side after the exercise, using BOB's input revealed for
test purposes only) and compare:

```python
# correctness.py — compare MPC vs plaintext for the TEST inputs only.
plain_out = plain_model(bob_input)
mpc_out   = mpc_forward(plain_model, bob_input)   # BOB's revealed output
# CrypTen uses 16/32-bit fixed-point; allow ~1e-3 absolute error.
assert torch.allclose(mpc_out, plain_out, atol=1e-3), \
    f"MPC/plaintext mismatch: max err = {(mpc_out - plain_out).abs().max()}"
```

### 2.5 Threat / security-model note

| Threat | Does CrypTen address it? | Why / Residual |
|---|---|---|
| ALICE learning BOB's input | **Yes** under semi-honest assumption | Core property of SS |
| BOB learning ALICE's weights | **Yes** under semi-honest assumption | Core property |
| Both parties learning the output | **Only to the designated recipient**; the other party gets nothing | The `dst=` argument of `get_plain_text` controls this |
| ALICE deviating from the protocol (malicious) | **No** — CrypTen is semi-honest, not maliciously secure | Use a maliciously-secure protocol (e.g., SPDZ with MACs) or wrap MPC parties in TEEs ([[exercise-02-amd-sev-snp-attestation-walkthrough]]) |
| BOB deviating from the protocol | **No** | Same |
| Collusion (3rd-party colluding with ALICE in a 3-party variant) | Depends on protocol | Document threshold; two-party is a two-party trust model |
| Network observer on the wire | Not by MPC alone | Still need TLS between parties |
| Output privacy from the designated recipient | **No** by design — the recipient sees plaintext | Needs DP / output filtering [[mod-105-memorisation-extraction-and-mia]] |
| Model-inversion via repeated queries | **No** | MITRE ATLAS `AML.T0024 Exfiltration via ML Inference API` |
| Supply-chain poisoning of the model (MITRE ATLAS `AML.T0010`) | **No** | Build-pipeline controls |
| Side channels (timing / network traffic patterns) | **No** | Padding + constant-time protocols if needed |
| Replay of a transcript | **No** | Nonces at the application layer |
| Parameter mis-calibration (fixed-point width too small) | **No** — correctness can silently degrade | Validate against plaintext; see §3 |

### 2.6 Where MPC sits vs. HE and TEE

| Dimension | HE (TenSEAL / CKKS) | MPC (CrypTen) | TEE (SEV-SNP / TDX + GPU CC) |
|---|---|---|---|
| Model party | Holds plaintext weights | Holds secret-shared weights (plaintext before the protocol) | Holds plaintext weights inside the enclave |
| Input party | Encrypts and sends | Secret-shares with model party | Encrypts to the enclave (via attestation) |
| Communication cost | 1 round, large ciphertexts | Many rounds, small-to-moderate messages per op | 1 round for attestation + normal API traffic |
| Non-linearities | Hard — polynomial approximations required | Easy — ReLU, comparisons, exp via SS protocols | Easy — runs native PyTorch |
| Trust assumption | Server is semi-honest with full visibility of ciphertexts; security from cryptographic hardness | **Parties are semi-honest; no party sees the plaintext of the other side** | Hardware vendor + attested image |
| Scaling | Model size not a big deal; depth is the pain point | Both sides must participate; latency grows with model | Scales like plaintext inference with ~bandwidth tax |
| Hardware root of trust | None | None | Required (AMD / Intel / NVIDIA / Apple) |
| Deployment pattern | Cloud-friendly (server has ciphertext, client has key) | Cross-silo-friendly (two institutions compute together) | Cloud-friendly with attestation |

CrypTen (two-party MPC) is the natural choice when **two parties
are already in a relationship but do not trust each other with
their own data** — think hospital + research lab, or bank +
regulator — and both sides can afford to run a real compute node.

### 2.7 NIST AI RMF mapping

| Function | Artefact for this exercise |
|---|---|
| Govern | Legal basis for cross-organisation computation (DPA / BAA / joint-controller agreement); who signs off on "semi-honest is enough"? |
| Map | Two-party data-flow diagram (ALICE / BOB / network); the §2.5 threat table |
| Measure | Correctness test against plaintext; latency + bytes-on-wire measurements; drift detection |
| Manage | Escalation plan if correctness check fails or if a counterparty is suspected of deviating from the protocol |

## 3. Validation steps

1. **Install and sanity-check.** `pip install crypten`; run the
   CrypTen `mnist_mlp` tutorial from the repo to confirm the
   install.
2. **Run the reference implementation.** `python mpc_inference.py`
   launches two processes. Only BOB's output should print.
3. **Correctness check.** Compare the MPC output to the plaintext
   forward (§2.4). Max absolute error should be `≤ 1e-3`.
4. **Reveal-target test.** Change `get_plain_text(dst=BOB)` to
   `dst=ALICE` and confirm BOB no longer sees the output (and
   vice versa). This surfaces the `dst=` argument to the
   learner.
5. **Semi-honest-only negative test.** Modify one party to return
   a wrong share (simulate malicious). Confirm CrypTen does not
   detect it: the output silently changes. This demonstrates why
   semi-honest ≠ maliciously secure.
6. **Latency / bandwidth measurement.** Report median
   forward-pass latency and the total bytes exchanged per
   inference (CrypTen exposes a `comm.get().log_comms()` helper
   or you can wrap the socket). Do not quote external benchmarks.
7. **Model architecture is public.** Confirm (by inspection) that
   CrypTen's `from_pytorch` reveals the shape of every layer to
   both parties; only the parameter *values* and the input values
   are hidden.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Two-party setup clearly names ALICE/BOB roles and what each learns | 1 | §2.2 |
| MPC forward pass matches plaintext within fixed-point tolerance | 2 | `atol=1e-3` |
| `src=` and `dst=` are used correctly (only designated party reveals) | 2 | §2.3 |
| Threat table enumerates semi-honest-only, no-output-privacy, no-integrity | 2 | §2.5 |
| Comparison with HE and TEE patterns | 1 | §2.6 |
| Latency + bandwidth measured locally | 1 | Learner-run |
| ATLAS + NIST AI RMF mapping present | 1 | — |

Automatic failures:

- Revealing the output to the wrong party.
- Claiming CrypTen is maliciously secure (it is semi-honest).
- Claiming MPC prevents the designated recipient from leaking
  the output.
- Encrypting with `src=` from the wrong party (so a side with no
  real data broadcasts garbage).
- Pretending the architecture is secret — it is public in CrypTen's
  model.

## 5. Common mistakes

- **Treating CrypTen as production-ready.** CrypTen is a
  research-grade library; its own documentation says so. For
  production use, pair it with a maliciously-secure protocol or
  wrap both parties in TEEs.
- **Mixing `src` and `dst`.** `cryptensor(x, src=BOB)` says BOB
  is providing `x`. `get_plain_text(dst=BOB)` says BOB receives
  the plaintext. Swapping them destroys privacy.
- **Assuming non-linearities are free.** `torch.relu` in CrypTen
  is not a local op — it runs an MPC comparison sub-protocol.
  Deep models with many non-linearities get expensive quickly.
- **Forgetting the semi-honest caveat.** A party that deviates
  can learn *more* than the output, or can cause a wrong answer
  that is not detected. Semi-honest ≠ safe.
- **Thinking both parties see the output.** `get_plain_text`
  without `dst=` reveals to *both*; the exercise should show the
  asymmetric-reveal variant explicitly.
- **Confusing SS with HE.** In secret sharing, no single party
  can "decrypt" alone; recovery requires the other share. In HE,
  the client holds the sole secret key. The threat models are
  different.
- **Running both processes on one machine and calling the
  latency "realistic".** Localhost loopback hides the real cost.
  For a benchmark worth citing, run across two separate machines
  on a representative network.
- **Treating the model architecture as secret.** In CrypTen's
  default pattern the architecture is public; only weights and
  inputs are private. If architecture is also sensitive, that is
  a different (and much harder) problem.

## 6. References

- Knott, B.; Venkataraman, S.; Hannun, A.; Sengupta, S.; Ibrahim,
  M.; van der Maaten, L. *CrypTen: Secure Multi-Party Computation
  Meets Machine Learning.* NeurIPS 2021.
  <https://arxiv.org/abs/2109.00984>
- CrypTen — <https://github.com/facebookresearch/CrypTen>
  (practitioner implementation used here; see tutorials
  directory).
- Beaver, D. *Efficient Multiparty Protocols Using Circuit
  Randomization.* CRYPTO 1991.
  <https://doi.org/10.1007/3-540-46766-1_34> — the Beaver-triple
  construction underlying SS multiplication.
- Damgård, I.; Pastro, V.; Smart, N.; Zakarias, S. *Multiparty
  Computation from Somewhat Homomorphic Encryption* (SPDZ).
  CRYPTO 2012. <https://eprint.iacr.org/2011/535>  —
  maliciously-secure MPC reference, cited as the step beyond
  semi-honest.
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
