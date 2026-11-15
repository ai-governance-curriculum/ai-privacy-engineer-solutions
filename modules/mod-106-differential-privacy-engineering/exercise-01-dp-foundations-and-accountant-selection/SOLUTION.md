# Exercise 01 — DP Foundations and Accountant Selection — Reference Solution

## 1. Solution overview

This is a **design exercise**. The learner must (a) state the differential
privacy (DP) definition in a form a privacy reviewer can test against, (b)
distinguish the related variants that show up in modern DP libraries
(approximate DP, concentrated DP, zero-concentrated DP, Rényi DP, Gaussian
DP / f-DP), and (c) pick a privacy accountant that is defensible for the
intended release and show their work.

The assessable output has three parts:

1. **Definition sheet** — the one-paragraph statement of (ε, δ)-DP, plus a
   short table of the related variants and their composition stories.
2. **Accountant decision tree** — a short written algorithm that takes
   `(release type, mechanism, number of compositions, tolerance for
   looseness)` and returns a specific accountant.
3. **Worked decisions** — the tree applied to two concrete releases
   (a noisy count, a DP-SGD run) with the chosen accountant, the alternative
   that was rejected, and the reason.

The canonical definitions and usage conventions follow NIST SP 800-226,
*Guidelines for Evaluating Differential Privacy Guarantees* (2025), which is
the citation a learner should default to when a reviewer asks "why that
definition." Numerical conventions (δ ≪ 1/N, ε single-digit for user-level
training) also come from 800-226.

## 2. Implementation (worked answer)

### 2.1 Definition sheet

**(ε, δ)-DP (approximate DP).** A randomised mechanism *M* satisfies
(ε, δ)-differential privacy if, for every pair of neighbouring datasets
*D*, *D'* and every measurable set *S* of outputs,

> Pr[M(D) ∈ S] ≤ exp(ε) · Pr[M(D') ∈ S] + δ.

Three implementation-level choices turn this definition into something a
reviewer can test:

| Choice | What a reviewer inspects |
|---|---|
| **Neighbouring relation** — "add/remove one record" vs. "replace one record" (bounded vs. unbounded DP); user-level vs. example-level in ML | A written sensitivity statement for the mechanism |
| **Unit of privacy** — row, user, device, household | The join key the mechanism groups on before clipping/sampling |
| **δ budget** — must be cryptographically small; convention is δ ≪ 1/N where N is the dataset size (NIST SP 800-226 §3.1) | The release manifest stating N and δ |

A release that cannot answer these three questions is not (ε, δ)-DP in any
reviewable sense, regardless of what the code says.

**Variant map.** These are *relaxations* used to get tighter composition.

| Variant | One-line statement | Converts to (ε, δ)-DP? | When you reach for it |
|---|---|---|---|
| Pure DP (ε-DP) | δ = 0 case of (ε, δ)-DP | Trivially | One-shot Laplace/exponential mechanisms |
| Approximate DP ((ε, δ)-DP) | Definition above | — | The report format regulators expect |
| Concentrated DP (CDP, Dwork & Rothblum 2016) | Bounds *subgaussian* privacy loss | Yes, via conversion | Historical; superseded by zCDP |
| Zero-concentrated DP (ρ-zCDP, Bun & Steinke 2016) | Bounds *Rényi divergence of all orders by ρα* | Yes: (ε, δ)-DP with ε = ρ + 2√(ρ·ln(1/δ)) | Gaussian mechanism, many compositions, when you want clean algebra |
| Rényi DP ((α, ε')-RDP, Mironov 2017) | Bounds Rényi divergence at order α | Yes: (ε'+ ln(1/δ)/(α-1), δ)-DP | DP-SGD (default in TF Privacy and Opacus) |
| Gaussian DP / f-DP (Dong, Roth & Su 2022) | Tracks the trade-off function between neighbouring datasets | Yes, exactly via the dual | Tighter accounting for subsampled Gaussian; used by the PRV / "Gaussian DP accountant" |

### 2.2 Accountant decision tree

Input: `(mechanism, release_type, n_compositions, subsampling, tolerance)`.

1. **Is the mechanism a single-shot Laplace / exponential / noisy-max over
   bounded sensitivity?**
   - Yes → use **basic composition in pure DP**. No accountant needed;
     ε adds, δ adds. Document the sensitivity bound and stop.
2. **Is the mechanism the Gaussian mechanism, composed a small, fixed
   number of times, with no subsampling?**
   - Yes → use the **analytic Gaussian mechanism** (Balle & Wang 2018)
     for the per-step bound, and **zCDP** to compose. Convert to (ε, δ)
     at the end.
3. **Is the mechanism subsampled Gaussian noise applied to clipped
   gradients (i.e., DP-SGD) or its minibatch variants?**
   - **Yes, and I want the tightest ε my library gives me** →
     **PRV (Privacy Random Variable) accountant** / **Gaussian DP
     accountant**. In Opacus this is `accountant="prv"`; in TF Privacy
     this is the `rdp_accountant` promoted to a GDP/PRV accountant in
     newer versions.
   - **Yes, and I need a battle-tested path** → **RDP accountant** with
     a wide α grid (e.g., 1.25–64). Slightly looser than PRV but has
     years of audit history and is the Opacus / TF Privacy default.
4. **Is the mechanism something exotic (sparse vector, report-noisy-max
   composed with Gaussian, PATE's data-dependent aggregation)?**
   - Use the mechanism-specific accountant published with the paper
     (e.g., for PATE: GNMAX data-dependent accountant, Papernot et al.
     2018). Do **not** try to compose it with a generic Gaussian
     accountant.
5. **Do you need the spent budget to be *monotone* across many ad-hoc
   queries (interactive analytics)?** Use a **privacy filter / odometer**
   (Rogers et al. 2016) layered on one of the above.

Default for new work on neural networks with subsampled Gaussian: the
**PRV accountant** if the library supports it; otherwise RDP with a wide
α grid. This is the recommendation that matches both Opacus's and TF
Privacy's own documentation.

### 2.3 Worked decisions

**Release A — DP noisy count of users who clicked a feature, released
daily for 90 days.**

- Mechanism: Gaussian noise added to a bounded count (sensitivity 1 under
  bounded DP), no subsampling, 90 sequential compositions.
- Chosen accountant: **zCDP**. Each Gaussian release contributes
  ρ = 1/(2σ²); 90 releases → total ρ = 90/(2σ²); convert to (ε, δ) at
  the end.
- Rejected alternative: basic composition in pure DP — loose by roughly a
  factor √(90·ln(1/δ)), which inflates ε needlessly.
- Rejected alternative: PRV accountant — overkill; the mechanism is not
  subsampled and the zCDP conversion is already tight for Gaussian.

**Release B — DP-SGD fine-tune of a classifier, 10 epochs, sample rate
q = 256/60 000, σ_noise = 1.1, clip = 1.0, δ = 1e-5.**

- Mechanism: subsampled Gaussian on clipped minibatch gradients, ~2 344
  steps, user-level neighbouring relation.
- Chosen accountant: **PRV accountant** (Opacus `PRVAccountant` or TF
  Privacy's `compute_dp_sgd_privacy_statement` which uses the PLD/PRV
  variant). Report ε and the α grid or discretisation parameters alongside
  the result.
- Rejected alternative: RDP with narrow α grid — RDP is correct but
  typically yields ε larger than PRV for the same σ at this scale,
  which spends privacy budget for no modelling benefit.
- Rejected alternative: reporting only per-step ε and multiplying — this
  is not how DP composes under subsampling; the per-step bound understates
  the amplification benefit and over-counts the Gaussian tail.

In both cases the release manifest records: the neighbouring relation,
the unit of privacy, the sensitivity bound (or clip norm), the noise
scale, the number of compositions, the accountant used, and the final
(ε, δ) with δ ≪ 1/N.

## 3. Validation steps

A learner can self-check the artifact:

1. **Definition sanity.** Rewrite the (ε, δ)-DP inequality from memory.
   If δ appears outside the `exp(ε)·Pr` term, revise.
2. **Neighbour statement test.** Write the sentence "Two datasets are
   neighbours when ____." If the blank cannot be filled with a *concrete*
   operation (add/remove user U, replace record R), the mechanism is not
   well-defined.
3. **δ convention check.** Confirm δ ≪ 1/N. A δ of 1e-3 against a
   100-record dataset is numerically bigger than 1/N and violates the
   NIST SP 800-226 guidance.
4. **Accountant match.** For each worked release, re-run the decision
   tree starting from the mechanism; the terminal node must match the
   accountant you chose.
5. **Conversion check.** Where the accountant works in zCDP / RDP,
   confirm the (ε, δ) report uses the published conversion formula
   (Bun-Steinke 2016 for zCDP; Mironov 2017 Theorem 3 for RDP → (ε, δ)).
6. **Library sanity.** Run the accountant on a trivial configuration
   (σ huge, steps = 1); it should return ε near zero. If not, the
   accountant is mis-parameterised.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| (ε, δ)-DP definition stated correctly with neighbouring relation named | 2 | "Add/remove one record" or "replace one record"; the choice must be explicit |
| Unit of privacy is stated (row vs. user vs. device) and consistent with the mechanism | 1 | User-level for ML training unless justified |
| Variant table lists at least pure DP, (ε, δ)-DP, zCDP, and RDP with correct composition stories | 2 | GDP/f-DP optional for full marks |
| Decision tree terminates in a specific accountant for each release | 2 | Not "whatever Opacus defaults to" |
| Worked releases name the rejected alternatives and *why* they were rejected | 2 | Looseness, inapplicability, or cost |
| δ convention (δ ≪ 1/N) is checked numerically in the write-up | 1 | — |

Automatic failures:

- Reporting ε without δ or without the accountant used.
- Claiming "pure DP" for a Gaussian mechanism.
- Choosing an accountant without stating what mechanism it is accounting
  for (accountants are mechanism-specific).
- Using basic composition for DP-SGD.

## 5. Common mistakes

- **Confusing "neighbouring" conventions.** Bounded DP (replace-one) and
  unbounded DP (add/remove) yield sensitivities that differ by a factor
  of 2 for counts. Document which convention the mechanism uses and keep
  it consistent across the sensitivity analysis and the accountant.
- **Reusing a Gaussian accountant for the Laplace mechanism.** The
  Gaussian accountant assumes the subsampled Gaussian mechanism; Laplace
  needs a Laplace- or pure-DP-aware accountant.
- **Picking RDP because "it's what everyone uses."** RDP is a sound
  default, but the PRV/GDP accountants typically give tighter ε for the
  same noise. The choice should be deliberate.
- **Ignoring the subsampling amplification.** DP-SGD's tight ε depends on
  Poisson subsampling. Using uniform batches (shuffle-then-chunk) breaks
  the amplification argument and inflates true ε; the library accountant
  silently assumes Poisson.
- **Double-counting composition.** If the accountant already composes
  the mechanism across steps internally, you do not then multiply by
  epochs.
- **Setting δ by taste.** δ must be cryptographically small; NIST
  SP 800-226 recommends δ much smaller than 1/N. δ = 0.01 on a
  one-million-record training set is not a DP release.
- **Reporting the worst-order α as "the" RDP.** RDP is a family indexed
  by α; you report the *converted* (ε, δ), not a single RDP point.

## 6. References

- NIST Special Publication 800-226, *Guidelines for Evaluating
  Differential Privacy Guarantees* (final, 2025).
  <https://csrc.nist.gov/pubs/sp/800/226/final> — treated as the
  authoritative reporting convention for this module.
- Dwork, C.; Roth, A. *The Algorithmic Foundations of Differential
  Privacy.* Foundations and Trends in Theoretical Computer Science, 2014.
  <https://www.cis.upenn.edu/~aaroth/privacybook.html>
- Bun, M.; Steinke, T. *Concentrated Differential Privacy: Simplifications,
  Extensions, and Lower Bounds.* TCC 2016.
  <https://arxiv.org/abs/1605.02065>
- Mironov, I. *Rényi Differential Privacy.* CSF 2017.
  <https://arxiv.org/abs/1702.07476>
- Dong, J.; Roth, A.; Su, W. *Gaussian Differential Privacy.* JRSS-B 2022.
  <https://arxiv.org/abs/1905.02383>
- Balle, B.; Wang, Y.-X. *Improving the Gaussian Mechanism for
  Differential Privacy: Analytical Calibration and Optimal Denoising.*
  ICML 2018. <https://arxiv.org/abs/1805.06530>
- Abadi, M. et al. *Deep Learning with Differential Privacy.* ACM CCS 2016.
  <https://arxiv.org/abs/1607.00133>
- Opacus documentation — privacy accountants.
  <https://opacus.ai/docs/faq>
- TensorFlow Privacy documentation — `compute_dp_sgd_privacy_statement`.
  <https://github.com/tensorflow/privacy>
- NIST AI Risk Management Framework (AI RMF 1.0) — used in this module as
  the operational vocabulary for release-governance decisions.
  <https://www.nist.gov/itl/ai-risk-management-framework>
