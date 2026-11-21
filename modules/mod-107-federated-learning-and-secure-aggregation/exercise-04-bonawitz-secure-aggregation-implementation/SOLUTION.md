# Exercise 04 — Bonawitz Secure Aggregation Implementation — Reference Solution

## 1. Solution overview

This is an **implementation exercise**. The learner implements the
core of the Bonawitz et al. (2017) secure aggregation protocol: a set
of clients each hold an update vector; a server wants the *sum* of
those vectors but must not be able to see any single client's
contribution. The construction uses pairwise additive masks derived
from Diffie–Hellman shared keys, plus an independent self-mask whose
seed is secret-shared so that aggregation survives dropouts.

The assessable artifacts:

1. A **reference implementation** of a 2-round secure aggregation
   protocol between `n` clients and one server, parameterised by the
   number of tolerated dropouts `t`.
2. **Unit tests** proving three properties: (a) correctness without
   dropouts (the aggregator recovers the exact sum), (b) correctness
   with up to `t` dropouts (the aggregator still recovers the sum
   over the surviving clients), and (c) hiding (any `n-1` subset of
   client views contains no information about the one honest client's
   update; verified by equal-distribution test).
3. A **threat / scope note** that enumerates what the protocol *does
   not* protect against (model-poisoning updates still survive the
   aggregation, and the server still sees the sum) and references
   OWASP ML Top 10, MITRE ATLAS, and NIST AI RMF for the governance
   wrapper.

The implementation follows the construction in Bonawitz et al.,
*Practical Secure Aggregation for Privacy-Preserving Machine Learning*
(ACM CCS 2017) — pairwise masks + self-mask + Shamir secret sharing
for dropout recovery, over a prime-order group (`x25519` for the DH
operations, SHA-256 for the HKDF). The threat-model framing is kept
consistent with NIST AI RMF 1.0 so the artifact slots into a cross-
device deployment review ([[exercise-01]]).

## 2. Worked implementation

### 2.1 Protocol recap (what we are building)

Let `n` clients `C_1, …, C_n` each hold a vector `x_i ∈ Z_R^d`
(updates clipped and quantised to a ring of modulus `R`).

- **Round 0 — key setup.** Every pair `(i, j)` establishes a shared
  key `s_{i,j}` via Diffie–Hellman. Define the pairwise mask
  `PRG(s_{i,j})` with sign depending on order: `+PRG(s_{i,j})` on `C_i`,
  `-PRG(s_{i,j})` on `C_j` (so the two cancel when summed).
- **Round 1 — commit masked update.** Each `C_i` picks a private
  self-mask seed `b_i`, computes
  `y_i = x_i + PRG(b_i) + Σ_{j<i} PRG(s_{j,i}) - Σ_{j>i} PRG(s_{i,j}) (mod R)`,
  and sends `y_i` to the server.
- **Round 2 — reveal recovery shares.** Each client secret-shares
  `b_i` and `s_{i,*}` with the other clients using Shamir t-of-n. The
  server, after collecting `y_i` from the surviving set `U_2`,
  requests from `U_2` either (a) `b_i` for every `i ∈ U_2` or
  (b) the shares of `s_{i,*}` for every `i ∉ U_2`, never both. From
  these, the server reconstructs the aggregate `Σ_{i ∈ U_2} x_i`.

The key security property, stated informally: the server cannot
distinguish any single honest client's input from any other — all
information about `x_i` is masked by seeds that the server either
never learns or only learns in a form that cancels in the sum.

### 2.2 Reference implementation (self-contained)

```python
# secagg.py — pedagogical reference. Not production: see §5.
from __future__ import annotations

import hashlib
import hmac
import os
import secrets
from dataclasses import dataclass, field
from typing import Dict, List, Tuple

from cryptography.hazmat.primitives.asymmetric.x25519 import (
    X25519PrivateKey, X25519PublicKey,
)
from cryptography.hazmat.primitives.serialization import (
    Encoding, PublicFormat, PrivateFormat, NoEncryption,
)

# ---------- parameters -----------------------------------------------------
PRIME = (1 << 61) - 1          # Mersenne-prime ring for the shamir shares
R     = 1 << 32                # modulus for the masked vector arithmetic
VEC_D = 128                    # update dimensionality (unit test uses 128)


# ---------- helpers --------------------------------------------------------
def _prg(seed: bytes, length: int) -> List[int]:
    """Expand `seed` into `length` ring-R elements. HKDF-SHA256 counter mode."""
    out, counter, buf = [], 0, b""
    while len(out) < length:
        block = hmac.new(seed, counter.to_bytes(4, "big"), hashlib.sha256).digest()
        buf += block
        counter += 1
        while len(buf) >= 4 and len(out) < length:
            out.append(int.from_bytes(buf[:4], "big") % R)
            buf = buf[4:]
    return out


def _hkdf_shared(secret: bytes, salt: bytes, info: bytes = b"secagg") -> bytes:
    """HKDF-extract-then-expand, 32 bytes out — the PRG seed."""
    prk = hmac.new(salt, secret, hashlib.sha256).digest()
    return hmac.new(prk, info + b"\x01", hashlib.sha256).digest()


def _vec_add(a: List[int], b: List[int]) -> List[int]:
    return [(x + y) % R for x, y in zip(a, b)]


def _vec_sub(a: List[int], b: List[int]) -> List[int]:
    return [(x - y) % R for x, y in zip(a, b)]


# ---------- Shamir t-of-n over Z_p -----------------------------------------
def _poly_eval(coefs: List[int], x: int, p: int) -> int:
    y = 0
    for c in reversed(coefs):
        y = (y * x + c) % p
    return y


def shamir_split(secret: int, n: int, t: int, p: int = PRIME) -> List[Tuple[int, int]]:
    coefs = [secret] + [secrets.randbelow(p) for _ in range(t - 1)]
    return [(i, _poly_eval(coefs, i, p)) for i in range(1, n + 1)]


def shamir_combine(shares: List[Tuple[int, int]], p: int = PRIME) -> int:
    # Lagrange interpolation at 0
    total = 0
    for j, (xj, yj) in enumerate(shares):
        num, den = 1, 1
        for m, (xm, _) in enumerate(shares):
            if m == j:
                continue
            num = (num * (-xm)) % p
            den = (den * (xj - xm)) % p
        total = (total + yj * num * pow(den, -1, p)) % p
    return total


# ---------- data structures ------------------------------------------------
@dataclass
class ClientState:
    cid: int
    sk: X25519PrivateKey
    pk: X25519PublicKey
    b_seed: int = 0                         # self-mask seed (int < PRIME)
    pairwise: Dict[int, bytes] = field(default_factory=dict)  # j -> seed
    shares_b: Dict[int, Tuple[int, int]] = field(default_factory=dict)       # j -> (xj,yj)
    shares_pair: Dict[int, Dict[int, Tuple[int, int]]] = field(default_factory=dict)
    update: List[int] = field(default_factory=list)


# ---------- Round 0 --------------------------------------------------------
def client_setup(cid: int) -> ClientState:
    sk = X25519PrivateKey.generate()
    return ClientState(cid=cid, sk=sk, pk=sk.public_key())


def client_derive_pairwise(me: ClientState, others_pks: Dict[int, X25519PublicKey]) -> None:
    for j, pk_j in others_pks.items():
        if j == me.cid:
            continue
        shared = me.sk.exchange(pk_j)
        # Deterministic salt by unordered pair so both sides agree.
        a, b = sorted([me.cid, j])
        salt = f"pair-{a}-{b}".encode()
        me.pairwise[j] = _hkdf_shared(shared, salt)


# ---------- Round 1 --------------------------------------------------------
def client_commit(me: ClientState, x_i: List[int]) -> List[int]:
    me.update = x_i
    me.b_seed = secrets.randbelow(PRIME)
    b_mask = _prg(me.b_seed.to_bytes(32, "big"), VEC_D)
    y = [(xi + bi) % R for xi, bi in zip(x_i, b_mask)]
    for j, s_ij in me.pairwise.items():
        pair_mask = _prg(s_ij, VEC_D)
        if j > me.cid:
            y = _vec_sub(y, pair_mask)
        else:
            y = _vec_add(y, pair_mask)
    return y


# ---------- Round 2 --------------------------------------------------------
def client_share_secrets(
    me: ClientState, all_cids: List[int], t: int
) -> Tuple[Dict[int, Tuple[int, int]], Dict[int, Dict[int, Tuple[int, int]]]]:
    """Each client splits b_seed and every pairwise seed with Shamir t-of-n."""
    n = len(all_cids)
    shares_b = {j: s for j, s in zip(all_cids, shamir_split(me.b_seed, n, t))}

    shares_pair = {}
    for j, seed_bytes in me.pairwise.items():
        seed_int = int.from_bytes(seed_bytes, "big") % PRIME
        shares_pair[j] = {
            k: s for k, s in zip(all_cids, shamir_split(seed_int, n, t))
        }
    return shares_b, shares_pair


def server_aggregate(
    y_vecs: Dict[int, List[int]],
    survivors: List[int],
    dropouts: List[int],
    # per-helper disclosures the server collected in round 2:
    b_shares: Dict[int, List[Tuple[int, int]]],       # survivor id -> shares of its own b_seed
    pair_shares: Dict[int, Dict[int, List[Tuple[int, int]]]],  # dropped id -> {other id -> shares of s_{dropped, other}}
) -> List[int]:
    """Reconstruct Σ_{i ∈ survivors} x_i from the masked y_i."""
    if not survivors:
        return [0] * VEC_D

    agg = [0] * VEC_D
    for i in survivors:
        agg = _vec_add(agg, y_vecs[i])

    # Undo every *survivor's* self-mask using its reconstructed b_seed.
    for i in survivors:
        b_seed = shamir_combine(b_shares[i])
        agg = _vec_sub(agg, _prg(b_seed.to_bytes(32, "big"), VEC_D))

    # Add back every pairwise mask that the dropouts injected but that
    # cannot cancel because the dropout's own y_i is missing.
    for d in dropouts:
        for other, shares in pair_shares[d].items():
            s_do = shamir_combine(shares)
            seed_bytes = s_do.to_bytes(32, "big")
            mask = _prg(seed_bytes, VEC_D)
            if other > d:
                agg = _vec_add(agg, mask)   # d had subtracted this; add back
            else:
                agg = _vec_sub(agg, mask)   # d had added this; subtract back
    return agg
```

A full-protocol integration (what a reviewer actually runs) wires
these together through a trivial in-memory "server":

```python
def simulate(n: int, t: int, dropped: List[int], updates: Dict[int, List[int]]):
    clients = {i: client_setup(i) for i in range(n)}
    pks = {i: c.pk for i, c in clients.items()}
    for c in clients.values():
        client_derive_pairwise(c, {j: pks[j] for j in pks if j != c.cid})

    y_vecs, b_shares_aggregated, pair_shares_aggregated = {}, {}, {}
    all_cids = list(clients.keys())

    for i, c in clients.items():
        y_vecs[i] = client_commit(c, updates[i])
        sb, sp = client_share_secrets(c, all_cids, t)
        # Other clients would normally store these; for the pedagogical
        # simulation we hand them straight to the aggregator-side map.
        for j, share in sb.items():
            b_shares_aggregated.setdefault(i, []).append(share)
        for j, per_other in sp.items():
            for k, share in per_other.items():
                pair_shares_aggregated.setdefault(i, {}).setdefault(j, []).append(share)

    survivors = [i for i in all_cids if i not in dropped]
    # The aggregator needs only `t` shares per secret, so slice to t:
    b_shares   = {i: b_shares_aggregated[i][:t] for i in survivors}
    pair_slice = {d: {j: pair_shares_aggregated[d][j][:t] for j in pair_shares_aggregated[d]}
                  for d in dropped}
    return server_aggregate(y_vecs, survivors, dropped, b_shares, pair_slice)
```

### 2.3 Tests (illustrative)

```python
import random
from secagg import simulate, VEC_D, R

def _rand_vec() -> list[int]:
    return [random.randrange(R) for _ in range(VEC_D)]

def _vec_sum(xs):
    out = [0] * VEC_D
    for x in xs:
        out = [(a + b) % R for a, b in zip(out, x)]
    return out

def test_no_dropouts():
    random.seed(0)
    updates = {i: _rand_vec() for i in range(5)}
    agg = simulate(n=5, t=3, dropped=[], updates=updates)
    assert agg == _vec_sum(updates.values())

def test_with_one_dropout():
    random.seed(1)
    updates = {i: _rand_vec() for i in range(5)}
    agg = simulate(n=5, t=3, dropped=[2], updates=updates)
    assert agg == _vec_sum(updates[i] for i in updates if i != 2)

def test_hiding_distribution():
    """The masked y_i, in isolation, is uniform over Z_R^d (statistical)."""
    random.seed(2)
    n_trials = 500
    counts = [0] * 10
    for _ in range(n_trials):
        updates = {i: _rand_vec() for i in range(3)}
        # peek at what the server would see for client 0 in Round 1
        from secagg import client_setup, client_derive_pairwise, client_commit
        clients = {i: client_setup(i) for i in range(3)}
        pks = {i: clients[i].pk for i in clients}
        for c in clients.values():
            client_derive_pairwise(c, {j: pks[j] for j in pks if j != c.cid})
        y0 = client_commit(clients[0], updates[0])
        counts[y0[0] % 10] += 1
    # Rough chi-sq on 10 buckets, n=500 — large tolerance, this is a
    # smoke test not a cryptographic proof.
    expected = n_trials / 10
    chi = sum((c - expected) ** 2 / expected for c in counts)
    assert chi < 20.0
```

## 3. Validation steps

1. **Baseline arithmetic.** `test_no_dropouts` must pass: the
   aggregator exactly recovers the plain sum of inputs.
2. **Dropout tolerance.** `test_with_one_dropout` must pass with at
   least one dropout, and the test file should include one run per
   dropout cardinality from 0 to `n - t` (the protocol guarantees
   recovery as long as *at least* `t` clients remain to reconstruct
   shares).
3. **Hiding smoke test.** `test_hiding_distribution` runs the Round 1
   output through a chi-square on the first coordinate; it should
   consistently pass with a comfortable margin.
4. **Threshold enforcement.** Run `shamir_combine` on `t - 1` shares
   of a known secret; the recovered value must *not* equal the
   secret. This confirms the split is genuinely t-of-n and not an
   accidental 1-of-n.
5. **Modulus arithmetic.** Choose inputs near the ring boundary
   (e.g. `R - 1`); the aggregate must still recover
   `Σ x_i mod R` without silent overflow. If the aggregator returns
   a negative value, mask addition is using Python ints without the
   modulus.
6. **Independent RNG.** Replace `secrets` with a deterministic PRNG
   seeded per client-id; the aggregate must still be correct. If it
   is not, the cancellation logic is accidentally reusing a seed
   between clients.
7. **Quantisation sanity.** The protocol only sums; real DP-FL
   updates are floats. Confirm your quantisation layer (fixed-point
   with scale `s`, clipping at `C`) round-trips with error below the
   training's noise tolerance. If not, the quantisation layer, not
   secure aggregation, is what will dominate error.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Round 0 derives pairwise seeds via DH + HKDF, deterministically per unordered pair | 2 | `pair-min-max` salt or equivalent |
| Round 1 produces a masked vector per client with self-mask + pairwise mask | 2 | `+PRG` / `-PRG` rule is correct |
| Round 2 splits `b_seed` and pairwise seeds via Shamir t-of-n | 1 | Over a prime field; threshold enforced |
| Server reconstructs the sum over survivors in both the no-dropout and dropout cases | 2 | Dropout recovery is the hard part |
| Tests cover no-dropout, dropout, and hiding (statistical) | 1 | — |
| All vector arithmetic is modulo R | 1 | Not Python ints |
| Threat note names at least three non-goals: the server still sees the sum, poisoning is out of scope, side-channels are out of scope | 1 | |

Automatic failures:

- Using the raw DH shared bytes as the PRG seed (no HKDF): leaks
  bits and breaks compatibility of the two sides.
- Letting the server request *both* `b_seed` for survivors *and*
  pairwise shares for the same survivor: the server can then unmask
  that client's original update. The exchange is a one-of-two choice
  per survivor.
- Splitting secrets with Shamir over `R` (the ring) instead of a
  prime field.
- Claiming "secure aggregation implies DP": it does not. Secure
  aggregation hides individual updates *before* summation; the sum
  itself can still leak. See [[exercise-05]] for DP on top.

## 5. Common mistakes

- **Treating the pedagogical implementation as production.** The
  above code is intentionally self-contained; a production
  implementation needs authenticated channels, replay protection,
  version-pinned curve + hash choices, dropouts-during-round-2
  handling (round 2 itself tolerates dropouts), and side-channel
  mitigations. Use a vetted library (e.g., TFF's secure aggregation
  path) for real deployments.
- **Confusing "secure aggregation" with "private aggregation."** The
  server learns the exact sum `Σ x_i`. That sum can still memorise
  individual clients; add DP noise *before* masking (see
  [[exercise-05]]) if that matters for the threat model.
- **Protecting against the wrong adversary.** Bonawitz secure
  aggregation assumes an *honest-but-curious server* (with optional
  active-security extensions). It does nothing about a malicious
  client sending a poisoning update: that update will cancel masks
  with its peers and get summed into the aggregate like any other.
  Byzantine-robust aggregation is a separate line of work.
- **Mishandling the one-of-two reveal.** For every client the server
  gets either (a) its self-mask or (b) shares of its pairwise seeds
  with dropouts — never both. Implementing both leaks the plain
  update.
- **Secret-sharing in the wrong field.** Pairwise seeds are 32-byte
  values; they are too large for naive Shamir over small primes.
  Use a prime larger than the seed interpreted as an integer, or
  split byte-wise (each byte via Shamir over GF(2⁸) or a prime
  slightly above 255). The reference uses a 61-bit prime and
  encodes seeds modulo it, which is acceptable only because the
  PRG re-expands them.
- **Dropping the self-mask.** Without `b_i`, a client that *appears*
  in the survivor set but whose peers all dropped out leaves a mask
  the server can zero. The self-mask ensures the client is hidden
  against the server even when its pairwise masks disappear.
- **Reporting aggregation success on an unverified sum.** The server
  can verify the sum by publishing a commitment to the global model
  before the round and checking it after; without that step a
  malicious aggregator can swap in a different sum. The exercise
  deliberately stops before active-security extensions, but the
  manifest should name them as follow-ons.
- **Missing the quantisation step.** Updates must be clipped,
  quantised to an integer ring, and only then masked. Floating-point
  updates have different arithmetic (non-associative under mod) and
  the masks will not cancel.

## 6. References

- Bonawitz, K.; Ivanov, V.; Kreuter, B.; Marcedone, A.; McMahan, H. B.;
  Patel, S.; Ramage, D.; Segal, A.; Seth, K. *Practical Secure
  Aggregation for Privacy-Preserving Machine Learning.* ACM CCS 2017.
  <https://eprint.iacr.org/2017/281> — the construction implemented
  here.
- Kairouz, P.; McMahan, H. B.; et al. *Advances and Open Problems in
  Federated Learning*, §4.2. FnTML 2021.
  <https://arxiv.org/abs/1912.04977>
- Bell, J. H.; Bonawitz, K. A.; Gascón, A.; Lepoint, T.; Raykova, M.
  *Secure Single-Server Aggregation with (Poly)Logarithmic Overhead.*
  ACM CCS 2020. <https://eprint.iacr.org/2020/704> — the follow-on
  that scales to millions of clients.
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- NIST Special Publication 800-186, *Recommendations for
  Discrete-Logarithm-Based Cryptography* (for curve choice).
  <https://csrc.nist.gov/pubs/sp/800/186/final>
