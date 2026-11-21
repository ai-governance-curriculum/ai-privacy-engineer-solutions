# Exercise 05 — TenSEAL HE Inference on a Small Model — Reference Solution

## 1. Solution overview

This is an **implementation exercise**. The learner uses
[TenSEAL](https://github.com/OpenMined/TenSEAL) (a PyTorch /
NumPy-friendly wrapper around Microsoft SEAL's CKKS scheme) to run
inference on a **small model** — logistic regression or a tiny
MLP — where the **client's input is encrypted** under a homomorphic
encryption (HE) scheme and the server computes the forward pass
over ciphertexts. The server never sees the plaintext input or the
plaintext output; only the client (holder of the secret key) can
decrypt.

Assessable artefacts:

1. **Reference implementation** that:
   - Fits (or loads) a small plaintext model on the MNIST / Iris
     scale.
   - Sets up a CKKS context (polynomial modulus degree, coefficient
     modulus chain, global scale) with explicit parameter choices.
   - Encrypts a test input client-side, runs an encrypted forward
     pass server-side, decrypts the result client-side.
   - Verifies the HE result matches the plaintext result to within
     a CKKS-appropriate tolerance.
2. **Parameter note** that justifies polynomial modulus degree,
   coefficient modulus chain, and global scale given the
   multiplicative depth of the model.
3. **Threat / scope note** explaining what HE does and does not
   protect against. In particular: HE protects inputs in use at
   the server but does **not** by itself protect against
   malicious model behaviour (poisoning, backdoors), does
   **not** prevent output leakage to the (legitimate) client, and
   is **not** a drop-in replacement for differential privacy or
   secure aggregation.
4. **Benchmark table** reporting latency and ciphertext-size
   overhead for the chosen model — honestly scoped, per the
   learner's own measurement, not quoted from marketing numbers.

The construction follows TenSEAL's official documentation and
tutorials for encrypted inference, which is a practitioner
implementation of the CKKS scheme (Cheon–Kim–Kim–Song 2017) via
Microsoft SEAL. Threat framing is kept consistent with the NIST
AI RMF, MITRE ATLAS, and OWASP ML Top 10.

## 2. Worked implementation

### 2.1 Scope and choice of scheme

Why **CKKS** (and TenSEAL's wrapper) for this exercise:

- CKKS supports **approximate arithmetic over real / complex
  vectors**, which matches ML activations.
- TenSEAL packs the vector into a single ciphertext ("batched
  encoding"), so a fixed-size feature vector ↔ one ciphertext.
- Linear layers → `encrypted_tensor.matmul(plain_weight)`.
- Element-wise squares are cheap (`ct * ct` is one multiplicative
  level). Higher-degree polynomial activations are
  possible but expensive.
- No exponentials / softmax / ReLU directly — must use
  polynomial approximations (e.g., `x^2` or low-degree Chebyshev
  approximations of ReLU).

Because CKKS is approximate, we must:

- Choose parameters so that the **multiplicative depth** of the
  circuit fits within the coefficient modulus chain's levels.
- Choose a **global scale** small enough to leave precision after
  the final layer.
- Bound the inputs so numerical overflow doesn't collapse
  precision.

### 2.2 Model choice

Pick one of the following (either is fine; the MLP demonstrates
the harder parameter sizing):

- **Logistic regression** on Iris / Wine: `y = σ(Wx + b)`. The
  forward pass is a single linear layer; the sigmoid is replaced
  by a 3rd-order polynomial approximation around 0 (e.g.,
  `0.5 + 0.197x - 0.004x^3`, a common CKKS-friendly σ
  approximation).
- **1-hidden-layer MLP** on a 196-pixel (14×14 downsampled) MNIST
  slice: `y = W2 · square(W1 x + b1) + b2`, where `square` is
  used as the HE-friendly activation. With a hidden size of ~32
  this is the classic TenSEAL tutorial shape and is deep enough
  to force real parameter sizing.

Both train happily in plaintext PyTorch; HE is only used at
inference.

### 2.3 CKKS parameter choice (worked for the MLP)

The circuit has 2 multiplicative levels (one for `W1 x`, one for
`square`, one for `W2·h`; square counts as one level, matmul as
one). Rule of thumb for CKKS:

- `poly_modulus_degree`: 8192 for depth ≤ 2; 16384 for depth 3–4;
  32768 for deeper circuits. 8192 is fast and fits our 2–3 level
  budget. **8192 is the TenSEAL tutorial default and is the right
  starting point for the MLP sketched above.**
- `coeff_mod_bit_sizes`: a chain whose length equals
  `depth + 2` (one for the input scale, one for the final noise
  budget). A common chain for depth 2 at `poly_modulus_degree=8192`
  is `[40, 21, 21, 21, 40]` (first and last are "boundary" primes,
  middle ones match the global scale's bit-width).
- `global_scale`: `2^21` for the chain above. The scale must
  match the middle prime sizes.
- Security level: `tenseal` defaults map these parameters to the
  HE standard 128-bit security recommendation as of the HE
  Security Standard ([homomorphicencryption.org](https://homomorphicencryption.org/standard/)).
  Do **not** reduce `poly_modulus_degree` below the standard
  table's row for your coefficient modulus total.

### 2.4 Reference implementation

```python
# he_inference.py — CKKS inference on a small MLP with TenSEAL.
# Reference implementation for the exercise. Requires tenseal>=0.3.14.
import time
from typing import Tuple

import numpy as np
import tenseal as ts
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms


# ---------------------------------------------------------------------------
# 1. Train a tiny plaintext MLP on downsampled MNIST (14x14, 196 features).
# ---------------------------------------------------------------------------
class TinyMLP(nn.Module):
    def __init__(self, in_dim=196, hidden=32, out_dim=10):
        super().__init__()
        self.fc1 = nn.Linear(in_dim, hidden)
        self.fc2 = nn.Linear(hidden, out_dim)

    def forward(self, x):
        h = self.fc1(x)
        h = h * h           # square activation — CKKS-friendly
        return self.fc2(h)


def downsample(img: torch.Tensor) -> torch.Tensor:
    # 28x28 -> 14x14 by 2x2 average pooling, then flatten
    return nn.functional.avg_pool2d(img, 2).flatten()


def train_plaintext(seed: int = 0) -> TinyMLP:
    torch.manual_seed(seed)
    tfm = transforms.Compose([transforms.ToTensor(), transforms.Lambda(downsample)])
    train = datasets.MNIST(".data", train=True, download=True, transform=tfm)
    loader = torch.utils.data.DataLoader(train, batch_size=128, shuffle=True)
    net = TinyMLP()
    opt = optim.Adam(net.parameters(), lr=1e-3)
    loss_fn = nn.CrossEntropyLoss()
    for epoch in range(3):
        for x, y in loader:
            opt.zero_grad()
            out = net(x)
            loss_fn(out, y).backward()
            opt.step()
    return net


# ---------------------------------------------------------------------------
# 2. CKKS context setup. Parameter choice is explained in §2.3.
# ---------------------------------------------------------------------------
def make_context() -> ts.Context:
    ctx = ts.context(
        scheme=ts.SCHEME_TYPE.CKKS,
        poly_modulus_degree=8192,
        coeff_mod_bit_sizes=[40, 21, 21, 21, 40],   # depth-2 chain with scale 2^21
    )
    ctx.global_scale = 2 ** 21
    ctx.generate_galois_keys()      # needed for vector rotations inside TenSEAL ops
    return ctx


# ---------------------------------------------------------------------------
# 3. Encrypted forward pass. Weights stay plaintext; input is encrypted.
# ---------------------------------------------------------------------------
def encrypted_forward(enc_x: ts.CKKSVector, W1, b1, W2, b2) -> ts.CKKSVector:
    # W1.T is (in_dim, hidden). TenSEAL's matmul takes a plain matrix.
    h = enc_x.matmul(W1.T.tolist()) + b1.tolist()
    h = h.square()                               # depth++
    out = h.matmul(W2.T.tolist()) + b2.tolist()  # depth++
    return out


def plaintext_forward(x: np.ndarray, W1, b1, W2, b2) -> np.ndarray:
    h = x @ W1.T + b1
    h = h * h
    return h @ W2.T + b2


def run_one(net: TinyMLP, ctx: ts.Context, x: torch.Tensor) -> Tuple[np.ndarray, np.ndarray, float]:
    W1 = net.fc1.weight.detach().numpy().astype(np.float64)
    b1 = net.fc1.bias.detach().numpy().astype(np.float64)
    W2 = net.fc2.weight.detach().numpy().astype(np.float64)
    b2 = net.fc2.bias.detach().numpy().astype(np.float64)
    x_np = x.numpy().astype(np.float64)

    # --- client side: encrypt ---
    enc_x = ts.ckks_vector(ctx, x_np.tolist())

    # --- server side: forward on ciphertext ---
    t0 = time.perf_counter()
    enc_out = encrypted_forward(enc_x, W1, b1, W2, b2)
    t1 = time.perf_counter()

    # --- client side: decrypt ---
    he_out = np.array(enc_out.decrypt(), dtype=np.float64)
    pt_out = plaintext_forward(x_np, W1, b1, W2, b2)
    return he_out, pt_out, t1 - t0


# ---------------------------------------------------------------------------
# 4. End-to-end test harness.
# ---------------------------------------------------------------------------
def main():
    net = train_plaintext()
    ctx = make_context()

    tfm = transforms.Compose([transforms.ToTensor(), transforms.Lambda(downsample)])
    test = datasets.MNIST(".data", train=False, download=True, transform=tfm)

    latencies, errs, correct = [], [], 0
    N = 32
    for i in range(N):
        x, y = test[i]
        he_out, pt_out, dt = run_one(net, ctx, x)
        latencies.append(dt)
        errs.append(np.max(np.abs(he_out - pt_out)))
        correct += int(np.argmax(he_out) == y)
    print(f"accuracy on {N} test examples: {correct / N:.3f}")
    print(f"HE-vs-PT max abs error (median): {np.median(errs):.3e}")
    print(f"server-side encrypted forward latency (median): {np.median(latencies):.3f} s")

if __name__ == "__main__":
    main()
```

Expected shape of the results (your numbers will depend on the
host — do not quote these as universal):

- Accuracy on a 32-sample slice matches plaintext within
  rounding; mismatches should be zero or extremely rare on this
  model size.
- Max-abs HE–PT error per logit is well under `1e-3` with the
  parameter choice in §2.3; if it is not, the global scale is too
  small for the depth.
- Encrypted forward pass is ~10–100× slower than the plaintext
  forward, depending on hardware. Ciphertext size is
  `poly_modulus_degree × (coeff_mod_bits / 8) × 2` bytes
  (two polynomials per CKKS ciphertext at this level).

### 2.5 Threat / scope note

| Threat | Does CKKS inference address it? | Why / Mitigation |
|---|---|---|
| Server reading user input in the clear | **Yes** — input is ciphertext | Core property of HE |
| Server reading user output in the clear | **Yes** — output ciphertext decrypts only with client key | Core property |
| Man-in-the-middle on the wire | Not by HE alone | Still need TLS between client and server |
| Model weights exfil by the server | **No** — weights are plaintext on the server | Pair with a TEE if weights are also sensitive, or treat as acceptable if the server owns the model |
| Membership inference / model inversion on outputs | **No** — the attacker is the legitimate client, who sees decrypted outputs by design | Needs DP / output filtering [[mod-105-memorisation-extraction-and-mia]] |
| Model poisoning (MITRE ATLAS `AML.T0010`) | **No** | Supply-chain controls |
| Malicious server returning a wrong ciphertext (integrity) | **No** — vanilla CKKS gives semantic security, not integrity | Either MAC the ciphertext, use verifiable HE, or wrap the server in a TEE |
| Side channels on the HE evaluator | **No** | Document and treat as out of scope |
| Parameter misconfig leading to broken security (e.g., `poly_modulus_degree` too small) | **No** — security is only as strong as the parameter choice | Follow the HE Security Standard tables |
| Replay of encrypted inputs by the server | **No** | Client-side nonce + log, not an HE property |

### 2.6 OWASP ML / ATLAS / NIST AI RMF mapping

- **NIST AI RMF** — the HE boundary is a **Map** artefact (data-
  flow diagram showing where plaintext never lives); the
  parameter choice is a **Measure** artefact (document the
  security level claim and verify it against the HE standard
  table); the **Manage** story must include how parameters are
  rotated if the HE standard changes.
- **MITRE ATLAS** — HE closes `AML.T0020 Data from Local System`
  at the inference server, but leaves `AML.T0024 Exfiltration via
  ML Inference API` wide open (the client sees plaintext outputs).
- **OWASP ML Top 10** — HE is relevant to data confidentiality
  entries but does not address model-poisoning or adversarial-
  example entries.

## 3. Validation steps

1. **Run the reference implementation** end-to-end on a
   developer machine. Confirm accuracy matches plaintext and
   per-logit absolute error is `< 1e-3`.
2. **Sanity-check the parameters.** Confirm with
   `ctx.parameters()` (or manual inspection) that the modulus
   chain and global scale match §2.3. If you change the model's
   depth, change the chain.
3. **Cross-check security level.** Look up
   `poly_modulus_degree=8192` with total `coeff_mod` bits
   `= 40+21+21+21+40 = 143` in the HE Security Standard table
   (<https://homomorphicencryption.org/standard/>) and confirm
   the entry is ≥ 128-bit security. If not, raise
   `poly_modulus_degree` to 16384.
4. **Depth-exceeded negative test.** Add a second `square()` to
   `encrypted_forward` and confirm the forward either errors out
   (chain exhausted) or produces visible garbage. The chain must
   be re-sized before adding depth.
5. **Round-trip test.** Encrypt and decrypt `x` with no operations
   and confirm the max-abs error is `≤ 2^{-global_scale_bits/2}`
   roughly (CKKS is approximate — exact equality is not expected).
6. **Ciphertext size check.** Serialise one `CKKSVector` and
   report its size. Expect tens-to-hundreds of KB per
   encrypted feature vector; much larger than the plaintext.
7. **Latency table.** Report median encrypted forward latency on
   your hardware vs. plaintext forward. Do not cite external
   benchmarks as yours.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Correct CKKS context: parameters match stated depth | 2 | §2.3 |
| Plaintext vs. HE output match within tolerance (`<1e-3` per logit for the reference model) | 2 | — |
| Encrypted forward pass runs; no decryption in the server function | 1 | Server only sees ciphertexts |
| Security-level cross-check against the HE Security Standard table | 1 | `poly_modulus_degree` + total coeff-modulus bits |
| Latency + ciphertext-size measurements included | 1 | Learner-run, not quoted |
| Threat/scope table calls out output-leakage, integrity, parameter-config residuals | 2 | §2.5 |
| ATLAS + NIST AI RMF mapping present | 1 | — |

Automatic failures:

- Using a scheme the model doesn't fit (e.g., BFV with real-valued
  activations — fix is CKKS).
- Choosing parameters that give `<128-bit` security per the HE
  standard without a documented reason.
- Running decryption inside the "server" function (defeats the
  point).
- Claiming HE provides integrity against a malicious server.
- Claiming HE prevents model-inversion / MIA (it does not).

## 5. Common mistakes

- **Over-claiming on privacy.** HE protects the input on the
  server side. The client still sees the output in plaintext;
  anything the model leaks through outputs is unaffected.
- **Mismatching the chain to depth.** Each multiplication drops
  one level from the modulus chain. Running out of levels makes
  decryption nonsense. Pick the chain to match the circuit depth
  before writing the forward pass.
- **Using `poly_modulus_degree=4096` for speed.** 4096 is
  too small for a depth-2 CKKS circuit at the 128-bit security
  tables — the HE Security Standard's row for the chosen
  coeff-modulus total pushes you up to 8192 or 16384.
- **Forgetting Galois keys.** Rotations (used by `matmul` for
  vector packing) require Galois keys. Call
  `generate_galois_keys()` after context creation.
- **Treating "accuracy matches plaintext" as HE correctness.**
  Accuracy is an argmax; CKKS can be approximate and still get
  argmax right. The precision check should look at **per-logit
  error**, not accuracy alone.
- **Missing ReLU.** CKKS has no native comparison. Either use a
  square activation (as above), a low-degree polynomial
  approximation of ReLU over a bounded range, or restructure the
  model. Shipping ReLU to CKKS directly will not compile.
- **Not scoping to small models.** CKKS over a deep CNN or a
  transformer is a research project, not a tutorial. Keep the
  exercise's claim scoped to "small logistic regression / tiny
  MLP"; do not generalise results.
- **Confusing TenSEAL with Microsoft SEAL.** TenSEAL is a
  practitioner implementation that wraps SEAL. Parameter semantics
  come from SEAL (and the HE Security Standard); TenSEAL is where
  the Python API conventions live.

## 6. References

- Cheon, J. H.; Kim, A.; Kim, M.; Song, Y. *Homomorphic Encryption
  for Arithmetic of Approximate Numbers.* ASIACRYPT 2017.
  <https://eprint.iacr.org/2016/421> — the CKKS scheme itself.
- Microsoft SEAL — <https://github.com/microsoft/SEAL>.
- TenSEAL (OpenMined) — <https://github.com/OpenMined/TenSEAL>
  (practitioner implementation / Python wrapper used here).
  Encrypted-inference tutorial:
  <https://github.com/OpenMined/TenSEAL/tree/main/tutorials>.
- Homomorphic Encryption Standard —
  <https://homomorphicencryption.org/standard/> — the community
  reference for safe parameter choices.
- Albrecht, M. R., Chase, M., Chen, H., et al. *Homomorphic
  Encryption Security Standard.*
  <https://homomorphicencryption.org/wp-content/uploads/2018/11/HomomorphicEncryptionStandardv1.1.pdf>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
