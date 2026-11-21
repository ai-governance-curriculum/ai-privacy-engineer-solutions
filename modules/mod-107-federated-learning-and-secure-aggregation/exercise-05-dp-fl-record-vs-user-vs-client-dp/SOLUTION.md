# Exercise 05 — DP-FL: Record- vs. User- vs. Client-Level DP — Reference Solution

## 1. Solution overview

This is a **design + implementation exercise** that sits between the
pure-DP module (mod-106) and the FL plumbing modules (mod-107
exercises 02 and 03). The learner must:

1. Distinguish **record-level**, **user-level**, and **client-level**
   differential privacy in the FL setting and state the neighbouring
   relation each one implies.
2. Choose the right unit for a given deployment (the cross-device
   next-word-prediction case, the cross-silo hospital case, and a
   case where one user straddles multiple clients).
3. Modify the FedAvg code from [[exercise-02]] to implement
   **DP-FedAvg** (McMahan, Ramage, Talwar, Zhang, 2018) at the
   chosen unit, with per-unit clipping and aggregator-side Gaussian
   noise.

The assessable artifacts:

1. A **DP-unit decision table** with three worked scenarios: the
   chosen unit, the neighbouring relation, the clipping group, the
   accountant, and the rejected alternative.
2. A **modified FedAvg strategy** that clips each client's delta to a
   fixed L2 norm `C` before aggregation and adds Gaussian noise with
   scale `σ · C` **at the server**, with user-level grouping when the
   chosen unit is user-level.
3. A **privacy report** that states (ε, δ) under an explicitly named
   accountant (Opacus `RDPAccountant` / `PRVAccountant` or TFF's
   `DpFedAvg` accountant) and documents how dropouts and sampling are
   handled.

Canonical references: McMahan et al., *Learning Differentially Private
Recurrent Language Models* (ICLR 2018), which introduces DP-FedAvg and
is the standard citation for user-level DP in FL; NIST SP 800-226 for
the reporting conventions (δ ≪ 1/N, accountant must be named); NIST
AI RMF 1.0 for the governance wrapper.

## 2. Worked answer

### 2.1 The three units

| Level | Neighbouring relation | What the clip groups | Example deployment |
|---|---|---|---|
| **Record-level** | Add/remove one *row* from one client's partition | Per-row gradient; standard DP-SGD inside the client | Cross-silo (hospitals, banks) when the controller's privacy claim is at the record level |
| **User-level** | Add/remove one *user* across the whole federation | All of one user's contributions, no matter which client they land on | Cross-device consumer apps; one user may own multiple devices |
| **Client-level** | Add/remove one *client* (one device or one silo) | The client's whole update to the global model | Cross-device next-word-prediction (one phone ≈ one user); cross-silo when an institution counts as the unit |

Three identities that are usually confused:

- **Record-level DP in FL** is *DP-SGD inside the client*
  (see mod-106 exercise 02), with no additional noise on the
  aggregate. It protects each row *from an observer of the server*
  because the client has already noised the gradient.
- **Client-level DP** is **the server clipping and noising per-client
  updates**. It protects each client as a unit against anyone who
  sees the global model, including other clients.
- **User-level DP** collapses to client-level DP **only when one user
  equals one client**. If one user spans multiple phones (cross-device
  with shared accounts), user-level DP needs user-keyed participation
  sampling or user-keyed grouping *before* clipping — client-level DP
  alone leaks for multi-client users.

### 2.2 Decision table (worked scenarios)

| Scenario | Chosen unit | Neighbouring relation | Clip group | Accountant | Rejected alternative + why |
|---|---|---|---|---|---|
| Next-word prediction on 50 M phones, each phone is one user | **Client-level ≈ user-level** | Add/remove one phone (= one user) | Whole-client delta, L2 ≤ C at the server | PRV accountant over the subsampled Gaussian at the aggregator | Record-level — a phone has thousands of records; protecting *rows* would still leak the user through the aggregate |
| Fraud model across 5 named banks with full participation | **Record-level** (DP inside each silo) | Add/remove one row inside a silo's dataset | Per-row gradient, inside the silo via DP-SGD | RDP / PRV inside each silo; no additional server noise | Client-level — "remove one bank" is a contractual action, not a DP mechanism; adding noise sized to bank-granularity wastes accuracy and misstates the claim |
| On-device assistant where users sign in on multiple devices | **User-level**, decoupled from client identity | Add/remove one *user*, even across devices | Group all of a user's device-updates by user-id *before* clipping | Opacus-style accountant over the per-round Poisson-sampled user set | Client-level — one user with three devices would get 3× the influence; client-level DP would "protect" each device and under-protect the user |

### 2.3 Implementation: DP-FedAvg at the aggregator

The simplest change on top of [[exercise-02]] is to replace Flower's
`FedAvg` with a DP strategy that (a) clips each client's delta to
norm `C`, and (b) adds Gaussian noise at scale `σ · C` to the sum
*before* averaging. Flower ships
`flwr.server.strategy.DifferentialPrivacyServerSideFixedClipping`
that does exactly this; the code below shows the shape, with
annotations.

```python
# server_app.py (fragment, replacing the FedAvg block from exercise-02)
from flwr.server.strategy import FedAvg, DifferentialPrivacyServerSideFixedClipping

def server_fn(context):
    num_rounds       = int(context.run_config["num-server-rounds"])
    fraction_fit     = float(context.run_config["fraction-fit"])
    num_clients      = int(context.run_config["num-clients"])   # N (eligible pool)

    # DP-FedAvg hyperparameters (committed up front, auditable)
    clip_norm        = float(context.run_config["dp-clip-norm"])     # C
    noise_multiplier = float(context.run_config["dp-noise-mult"])    # σ
    delta            = float(context.run_config["dp-delta"])         # δ << 1/N

    base = FedAvg(
        fraction_fit=fraction_fit,
        min_fit_clients=max(int(num_clients * fraction_fit), 2),
        min_available_clients=num_clients,
        initial_parameters=...,              # unchanged from exercise-02
        evaluate_metrics_aggregation_fn=...,  # unchanged
        on_fit_config_fn=lambda rnd: {
            "server_round": rnd, "local_epochs": 1, "lr": 0.01,
        },
    )

    # The DP wrapper clips each client's *delta* to `C`, sums, adds
    # Gaussian noise with std dev `σ · C / num_sampled_clients`, and
    # then averages. This is DP-FedAvg (McMahan et al. 2018).
    strategy = DifferentialPrivacyServerSideFixedClipping(
        strategy=base,
        noise_multiplier=noise_multiplier,
        clipping_norm=clip_norm,
        num_sampled_clients=max(int(num_clients * fraction_fit), 2),
    )
    config = ServerConfig(num_rounds=num_rounds)
    return ServerAppComponents(strategy=strategy, config=config)
```

What the reviewer verifies:

- **Clipping is before summation, not before averaging.** `C` bounds
  the sensitivity of the sum under add/remove-one-client — that is
  the mechanism the accountant accounts for.
- **Noise scale is `σ · C`, not `σ`.** The sensitivity is `C`; `σ` is
  dimensionless.
- **`num_sampled_clients` is honest.** If the strategy is pointed at
  `fraction_fit * num_clients` clients but secretly dispatches to
  fewer (because some are unavailable), the accountant is wrong. In
  cross-device deployments use *committed* sample sizes with
  dropouts folded in, or use Flower's
  `DifferentialPrivacyClientSideFixedClipping` which lets clients
  clip and the server noise over the surviving set.

#### User-level grouping when one user ≠ one client

If users span multiple clients, aggregation must *group by user*
before clipping. In Flower, this is a strategy-level responsibility:
include a `user_id` header per client contribution and clip the sum
of all of one user's deltas as one vector, then noise. A minimal
sketch, implemented as a custom strategy:

```python
class UserLevelDPFedAvg(FedAvg):
    def __init__(self, *, clip_norm, noise_multiplier, user_sample_rate, **kw):
        super().__init__(**kw)
        self.C, self.sigma, self.q = clip_norm, noise_multiplier, user_sample_rate

    def aggregate_fit(self, server_round, results, failures):
        # 1. Group deltas by user_id.
        by_user: dict[str, list] = {}
        for client, fit_res in results:
            uid = fit_res.metrics.get("user_id")
            if uid is None:
                raise ValueError("user_id missing; cannot do user-level DP")
            by_user.setdefault(uid, []).append(fit_res)
        # 2. Sum within user, clip user-wise.
        clipped = []
        for uid, fits in by_user.items():
            delta_u = sum_params([f.parameters for f in fits])
            clipped.append(l2_clip(delta_u, self.C))
        # 3. Sum clipped, add Gaussian noise ~ N(0, (sigma*C)^2 I).
        noisy_sum = add_gaussian(sum_params(clipped), self.sigma * self.C)
        # 4. Average over sampled *users* (not clients).
        return scalar_divide(noisy_sum, max(len(by_user), 1)), {}
```

Sketch only — the import helpers are the usual numpy/parameter
utilities. The point is that the clipping unit matches the DP unit;
without the group-by-user step, the mechanism is client-level DP
mislabelled as user-level.

### 2.4 Privacy report (what the artifact looks like)

```json
{
  "mechanism": "DP-FedAvg (McMahan, Ramage, Talwar, Zhang 2018)",
  "unit_of_privacy": "user",
  "neighbouring_relation": "add/remove one user (across all their devices)",
  "clip_norm_C": 1.0,
  "noise_multiplier_sigma": 1.1,
  "user_sample_rate_q": 0.001,
  "rounds": 2000,
  "delta": 1e-7,
  "dataset_size_N_users": 50000000,
  "delta_vs_N_check": "1e-7 << 1/50000000 ≈ 2e-8 — OK",
  "epsilon_reported": 6.4,
  "accountant": "PRV (opacus 1.x PRVAccountant on the subsampled Gaussian)",
  "clipping_side": "server-side after user-grouping",
  "secure_aggregation": "engaged (see exercise-04); noise added post-unmask",
  "notes": [
    "Dropouts are folded into the accountant via the committed q, not the realised one.",
    "Cross-link to exercise-01 for the cross-device trust-model rationale."
  ]
}
```

## 3. Validation steps

1. **Unit sanity.** The chosen unit matches the clip group. If the
   claim is user-level, show the grouping step (code or config) that
   sums deltas per user before clipping.
2. **Clip-before-noise.** Instrument the strategy to log the L2 norm
   of each delta *after* clipping and the std dev of the noise added.
   Clipped norms must all be `≤ C + ε_num` for numerical tolerance;
   noise std should equal `σ · C`.
3. **Sampling rate honesty.** `q` in the accountant equals the
   committed sample size divided by the eligible-pool size, not the
   number that actually responded in a given round. A cross-device
   deployment commits to the pool size and treats dropouts as a
   budget-neutral event.
4. **δ convention.** `delta << 1/N` where N is the number of
   *units*, not rows. For user-level DP on 50 M users, δ must be
   much smaller than 2 × 10⁻⁸.
5. **Accountant monotonicity.** ε grows monotonically across rounds.
   If it does not, the accountant is re-initialised per round or the
   sample rate is being mis-fed.
6. **Negative test — turn noise off.** Set `σ = 0`; ε must diverge
   (the accountant returns inf or raises). If it returns a small
   finite ε, the plumbing is wrong.
7. **Alignment with secure aggregation.** When [[exercise-04]] is
   enabled, Gaussian noise is added *after* the server unmasks the
   sum — not inside each client. A client-side noise addition on top
   of secure aggregation double-counts the noise and still leaks if
   clients do not coordinate the σ.
8. **Scenario sanity.** For each worked scenario in §2.2, re-state
   the neighbouring relation to a reviewer out loud. If the
   reviewer cannot immediately point to a mechanism operation that
   corresponds to "neighbours differ by one X," the unit is
   ambiguous.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Three units distinguished with explicit neighbouring relations | 2 | Record / user / client |
| Each worked scenario names the chosen unit, the clip group, the accountant, and a rejected alternative | 2 | Rejected alternative must be substantive, not negation |
| DP-FedAvg implementation clips per unit *before* noise | 2 | Clip then sum then noise is the only correct order |
| User-level grouping is implemented when user ≠ client | 1 | Can be a documented design decision when 1 user = 1 client |
| Privacy report names the accountant and library version | 1 | PRV / RDP / GDP with library pin |
| δ ≪ 1/N check against N counted in *units*, not rows | 1 | The common bug |
| Cross-link to secure aggregation + cross-device framing | 1 | [[exercise-04]] and [[exercise-01]] |

Automatic failures:

- Reporting "DP-FL with ε = X" without naming the unit. ε is
  meaningless without the unit — this is a NIST SP 800-226 hard
  requirement.
- Using record-level DP mechanics (per-row clipping) and claiming
  user-level DP.
- Setting the noise before the clip (noise scale no longer bounds
  sensitivity).
- Counting dropouts in `q` to make ε look smaller.

## 5. Common mistakes

- **"DP-FL with ε = 1" without a unit.** The most common bug in DP
  reporting. ε at user level and ε at record level on the same
  dataset can differ by two orders of magnitude; without a unit the
  number has no meaning. NIST SP 800-226 explicitly requires the unit.
- **Confusing DP-SGD-inside-clients with DP-FL.** DP-SGD inside each
  silo is *record-level* DP against an observer *of that silo*; the
  aggregate across silos can still leak per-silo information. Only
  server-side noise at the FL aggregator buys you a claim against
  the aggregator and the global model.
- **Clipping the delta's **individual layers** independently.** The
  clip must be a *global* L2 clip on the flat concatenation; layer-wise
  clipping bounds a different sensitivity and does not compose into the
  usual accountant.
- **Letting dropouts float into `q`.** Cross-device dropouts are
  pervasive; committing to a `q` and then noising over the surviving
  set yields a *bigger* real `q` than the accountant assumes. The fix
  is to commit the client sample size and divide by that, folding
  dropouts into noise overshoot, not into the accountant input.
- **Treating "one user = one client" as universal.** It is the norm
  in next-word prediction and it fails in consumer apps where one
  account signs in on multiple devices, and in households that share
  a device. Audit the identity model before picking client-level DP.
- **Composing DP with secure aggregation incorrectly.** Secure
  aggregation hides individual updates from the server; DP noise
  must still be added — either by the aggregator post-unmask (simple,
  but trusts the aggregator for noise) or distributed via a DRPG
  (Distributed Discrete Gaussian; see Kairouz et al. 2021 §4.2 for
  a survey).
- **Reporting ε without the accountant pin.** PRV ε, RDP ε, and GDP
  ε at the same `(C, σ, q, T)` are typically close but never
  identical. The accountant version must be in the manifest.
- **Choosing record-level for cross-device without checking.** On a
  phone, a single user's rows are strongly correlated; record-level
  DP does not bound user-level leakage and the privacy claim misleads
  reviewers. Default to user-level for cross-device unless there is
  a specific reason not to.

## 6. References

- McMahan, H. B.; Ramage, D.; Talwar, K.; Zhang, L. *Learning
  Differentially Private Recurrent Language Models.* ICLR 2018.
  <https://arxiv.org/abs/1710.06963> — DP-FedAvg; user-level DP
  framing.
- Abadi, M. et al. *Deep Learning with Differential Privacy.* ACM CCS
  2016. <https://arxiv.org/abs/1607.00133> — DP-SGD.
- Kairouz, P.; McMahan, H. B.; et al. *Advances and Open Problems in
  Federated Learning*, §4.2. FnTML 2021.
  <https://arxiv.org/abs/1912.04977>
- Mironov, I. *Rényi Differential Privacy.* CSF 2017.
  <https://arxiv.org/abs/1702.07476>
- NIST Special Publication 800-226, *Guidelines for Evaluating
  Differential Privacy Guarantees* (final, 2025).
  <https://csrc.nist.gov/pubs/sp/800/226/final>
- Flower DP strategies documentation.
  <https://flower.ai/docs/framework/ref-api/flwr.server.strategy.DifferentialPrivacyServerSideFixedClipping.html>
- Opacus documentation. <https://opacus.ai/>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
