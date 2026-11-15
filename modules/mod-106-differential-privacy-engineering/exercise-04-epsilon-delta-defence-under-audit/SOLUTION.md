# Exercise 04 — Epsilon-Delta Defence Under Audit — Reference Solution

## 1. Solution overview

This is a **design exercise** that simulates an auditor cross-examining
the privacy engineer about a DP release. The learner is given (or
chooses) a specific (ε, δ) pair for a shipped model or dataset and must
produce a written **defence pack** that answers the twelve questions an
auditor — internal reviewer, external assessor, or regulator — is likely
to ask, in the order they are likely to ask them.

The assessable output is a single document, structured as a Q&A, that
a non-specialist reviewer can read linearly and then mark "accept" or
"reject with specific remediation." The defence pack must:

1. State the release, the mechanism, and the exact privacy claim.
2. Walk the auditor through the twelve defence questions below.
3. Attach the artifacts (configs, logs, accountant transcripts) that
   back each answer.
4. Name the residual risks the release does *not* defend against, so
   the auditor is not surprised by them later.

The structure below is adapted from the evaluation questions in NIST
Special Publication 800-226, §4 ("Specifying and Comparing Differential
Privacy Guarantees") and §5 ("Evaluating Implementations").

## 2. Worked answer — the twelve-question defence pack

The following is the reference answer for the running example
**"a DP-SGD fine-tuned text classifier shipped at ε = 8, δ = 1e-5
user-level, Opacus PRV accountant."** The same twelve questions apply
to any release; the answers change.

### Q1. What exactly does the ε you report mean?

"The release satisfies (ε = 8, δ = 1e-5)-differential privacy under the
**add/remove one user** neighbouring relation, at the **user** unit of
privacy. The guarantee is end-to-end over the training run and the
released model weights. We do not claim DP at inference time beyond
what the DP training already provides."

Attachments: §3.1 of NIST SP 800-226 defining (ε, δ); the release
manifest's neighbour / unit fields.

### Q2. Why those numbers and not tighter ones?

"Smaller ε at fixed utility would require more noise, which the accuracy
budget cannot absorb (we hit a <performance target> ceiling). Smaller δ
would violate δ ≪ 1/N at N = <N>. The chosen (ε, δ) is the loosest
point that passed our model-quality bar."

Attachments: the accuracy-vs-ε sweep plot; the N-vs-δ check; the
business owner sign-off on the utility threshold.

### Q3. What's the unit of privacy and why?

"User. A user may contribute many records; we group by `user_id` before
Poisson-sampling in the data loader. Example-level DP would make the
reported ε look smaller at the cost of a weaker real guarantee for any
user with more than one record."

Attachments: the data loader code showing user-level grouping; the
decision memo rejecting example-level DP.

### Q4. What accountant did you use, and why that one?

"Opacus `PRVAccountant` (PRV = Privacy Random Variable, Gopi–Lee–
Wutschitz 2021). It is tighter than RDP at our σ and q, which saves
budget; it is the Opacus-recommended default for new DP-SGD work. RDP
was considered as the robust alternative; the delta between the two was
<x> at our hyperparameters, small enough to prefer PRV's tightness."

Attachments: accountant transcript from the training run; an RDP
comparison re-run on the same transcript.

### Q5. What's the mechanism, concretely?

"DP-SGD (Abadi et al. 2016): per-sample L2-norm clipping at C = 1.0,
Gaussian noise with σ = <σ_value>, Poisson subsampling with rate
q = <q_value>, run for T = <steps> steps across <epochs> epochs."

Attachments: training script with the DP optimizer construction call
highlighted; the data loader diff that enables Poisson sampling.

### Q6. How do you know the mechanism is implemented correctly?

"Three checks: (a) the library's internal validator passed (Opacus
`ModuleValidator.validate(model, strict=True)` returned no errors); (b)
a smoke test with `noise_multiplier = 0` and `C` finite produces a
model whose membership-inference AUC against a held-out attacker is
substantially above our shipping model's — confirming that noise is in
fact what reduces leakage; (c) a known-correct re-implementation on a
toy dataset reproduces ε within library rounding."

Attachments: the validator log; the no-noise smoke-test MIA report;
the toy-dataset reproduction.

### Q7. What population does the guarantee cover, and who does it leave out?

"The guarantee covers every user in the training population at the
time of sampling. It does **not** cover: downstream users who query the
model at inference (they are protected only insofar as DP training
already limits what the model memorises); non-training users whose data
is correlated with training users (DP gives individual, not relational,
guarantees); and training-pipeline operators with access to
non-noised intermediate state during training."

Attachments: NIST SP 800-226 §4.2 (unit of privacy, groups, correlated
data) for context.

### Q8. What happens when ε composes with other releases?

"We do **not** compose with other DP releases from the same training
corpus. If a second release is planned from the same users, a joint
accountant must be run, and the total ε budget must be re-reviewed. The
privacy register records the current spend against a <maximum per-user
budget> ceiling."

Attachments: privacy register row for this release; the budget-ceiling
policy.

### Q9. What's the δ budget, and is it small enough?

"δ = 1e-5. Dataset N = <N>; 1/N = <1/N>. Therefore δ ≪ 1/N holds
numerically. NIST SP 800-226 recommends δ much smaller than 1/N, and
the chosen value sits <k>× below 1/N."

Attachments: manifest's `delta_vs_N_check` field.

### Q10. What's the attack surface *inside* the DP boundary?

"Three vectors remain even if DP holds end-to-end: (a) the exact
training seed is not DP — someone who can rerun training with the same
seed learns which Poisson draws occurred; (b) non-privatised logs
(loss, accuracy per step) are not DP at the per-user level and are
retained only in-cluster; (c) the model's *gradients* during training
would leak if exported. Our runbook mitigates these by seed rotation,
log scrubbing, and gradient non-export respectively."

Attachments: the training-runbook sections on seed handling, log
retention, and gradient-handling rules.

### Q11. How would you know if the implementation drifted?

"CI check: a canary training run on fixed data with fixed
hyperparameters produces a pinned ε. The build fails if the accountant
returns a value outside a tight tolerance of that pin. Library versions
are pinned in a lock file; a bump triggers a forced re-review of all
active DP releases."

Attachments: the canary CI job; the lock file; the review policy.

### Q12. If this were overturned tomorrow, what breaks, and what do you do?

"The most likely overturn is a soundness bug in the chosen accountant
or in the per-sample-gradient implementation. Mitigation: (a) each
release records the second-choice accountant's ε so we can fall back
without re-running training; (b) model weights are distributed under a
release licence that allows us to recall and re-issue; (c) the
incident-response runbook has a 'DP recall' section with the
notification matrix."

Attachments: the release-licence terms; the recall section of the IR
runbook.

## 3. Validation steps

1. **Linear-readability test.** Hand the pack to a reviewer who is
   privacy-aware but not a DP researcher. They should be able to finish
   it in one sitting and point to each attachment without help.
2. **Numeric self-consistency.** Every number in the answers (ε, δ, N,
   σ, C, q, T) must appear identically in the training script and in
   the release manifest.
3. **Attachments exist.** Each Q answer references at least one file or
   log. Open each; broken links are automatic failures.
4. **Residual-risk honesty.** Q7, Q10, and Q12 explicitly name what the
   release does *not* cover. If any is handwaved ("we think this is
   fine"), rewrite with a concrete item.
5. **Reproducibility.** The canary run referenced in Q11 can be executed
   in CI and the pinned ε value matches the value in the pack.
6. **Mapping check.** Each Q answer maps cleanly to a section of NIST
   SP 800-226 §4 or §5 (listed in the attachment list). Any answer with
   no NIST anchor is probably ad hoc.

## 4. Rubric

Score out of 12 — one point per question — with the following
qualitative overlay. Below 9/12 → rework.

| Area | Required for full marks |
|---|---|
| Definition precision (Q1, Q3, Q9) | ε, δ, neighbouring relation, unit of privacy, δ ≪ 1/N check all explicit |
| Mechanism precision (Q5, Q6) | σ, C, q, T all stated; implementation-correctness evidence attached |
| Accounting precision (Q4, Q8, Q11) | Named accountant + version; composition policy; drift detection |
| Boundary honesty (Q2, Q7, Q10, Q12) | Residual risks named, not denied |

Automatic failures:

- Any answer that says only "we comply with GDPR / HIPAA / etc." Compliance
  is a *result* of the defence; it cannot stand in for the defence.
- Reporting ε without δ, or δ without an N-based check.
- An "accountant: Opacus" answer with no accountant variant named.
- Any claim of "ε-DP" for a Gaussian mechanism.
- Attachments that are promises ("will be published") rather than files.

## 5. Common mistakes

- **Treating the pack as a sales pitch.** The auditor is looking for
  *specifics they can fail*. Vague reassurances count against you —
  specific, falsifiable claims count in your favour.
- **Confusing the DP boundary with the system boundary.** DP protects
  against distinguishing a user's presence/absence in the training set
  from the released output. It does not protect against inference-time
  prompt injection, membership-inference via side channels, or
  insider access during training.
- **Hiding the composition story.** If the same training corpus has
  already shipped another DP release, that must be surfaced in Q8.
  "Private by DP" cannot survive silent composition.
- **Answering Q7 with "all users."** All users are covered by the
  mathematical definition; the honest answer names the users whose
  guarantee degrades (users with many records under example-level DP,
  users whose data is correlated with others, etc.).
- **Confusing the training seed with DP noise.** The seed controls the
  Poisson draws and PRNG sequence. Someone who knows the seed and can
  re-run the mechanism can un-noise the output. Seed handling is a
  non-DP control that sits *inside* the DP boundary.
- **Pointing to the library version only when asked.** The accountant
  is versioned; the pack should lead with the version, not treat it as
  trivia.
- **Omitting Q12 (what breaks if overturned).** Auditors use this
  question to probe operational maturity. A pack that cannot answer it
  is not shippable, however correct the DP math is.

## 6. References

- NIST Special Publication 800-226, *Guidelines for Evaluating
  Differential Privacy Guarantees* (final, 2025) — the twelve questions
  map to §4 ("Specifying and Comparing Differential Privacy
  Guarantees") and §5 ("Evaluating Implementations").
  <https://csrc.nist.gov/pubs/sp/800/226/final>
- Dwork, C.; Roth, A. *The Algorithmic Foundations of Differential
  Privacy.* FnT TCS 2014.
  <https://www.cis.upenn.edu/~aaroth/privacybook.html>
- Abadi, M. et al. *Deep Learning with Differential Privacy.* ACM CCS 2016.
  <https://arxiv.org/abs/1607.00133>
- Gopi, S.; Lee, Y. T.; Wutschitz, L. *Numerical Composition of
  Differential Privacy.* NeurIPS 2021.
  <https://arxiv.org/abs/2106.02848>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — used here only to frame
  non-DP residual threats (model theft, prompt injection, etc.) that
  the DP boundary does not cover.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — framework for naming adversarial-ML threats referenced
  in Q10.
  <https://atlas.mitre.org/>
