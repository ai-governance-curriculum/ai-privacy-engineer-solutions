# Exercise 03 — NVIDIA FLARE Cross-Silo Training — Reference Solution

## 1. Solution overview

This is an **implementation exercise** for the cross-silo topology
(see [[exercise-01]] for the topology memo). The learner stands up an
NVIDIA FLARE job with a small named silo count, drives one round of
FedAvg-style scatter–local train–gather across the silos, and produces
a cross-silo run manifest that a reviewer can plug into a joint
controller / DPIA packet.

The assessable artifacts:

1. A **FLARE job directory** with a server config, a client config,
   and a trainer executor.
2. A **trainer executor** (Python) that implements local fit + eval
   against a local partition and returns the parameter delta (or
   full parameter vector) through a `Shareable`.
3. A **run manifest** that records: provisioned silo names, start-up
   authentication (TLS/PKI), aggregator strategy, number of rounds,
   per-silo data size, and the explicit privacy-mechanism status
   (DP engaged / not engaged, secure aggregation engaged / not
   engaged).
4. A **threat-model note** that maps the deployment to NIST AI RMF
   functions and lists the relevant MITRE ATLAS techniques for a
   cross-silo system (compromised silo, poisoned update, exfiltration
   via aggregator).

NVIDIA FLARE is used here as a **practitioner example** of a
cross-silo FL framework, following standard FLARE job conventions
(provisioner, FLContext, Executor, Controller, Shareable). Authoritative
privacy/governance claims — unit of privacy, neighbouring relation,
reporting content — are tied to NIST AI RMF 1.0, MITRE ATLAS, and OWASP
ML Top 10. FLARE's own documentation (nvflare.readthedocs.io) is the
reference for API shapes.

## 2. Worked implementation

### 2.1 Job layout

NVIDIA FLARE jobs are directories with one server-side `config_fed_server.json`
and one client-side `config_fed_client.json` per app. A minimal job
for this exercise looks like:

```
nvflare-cross-silo-job/
├── app_server/
│   └── config/
│       └── config_fed_server.json
├── app_hospital_a/
│   ├── config/
│   │   └── config_fed_client.json
│   └── custom/
│       └── trainer.py
├── app_hospital_b/
│   ├── config/
│   │   └── config_fed_client.json
│   └── custom/
│       └── trainer.py
├── app_hospital_c/
│   ├── config/
│   │   └── config_fed_client.json
│   └── custom/
│       └── trainer.py
└── meta.json
```

Each silo gets its own `app_*` folder because, in a real deployment,
each silo ships its own code, pins its own dataset path, and is
provisioned with its own identity. Using three identical apps is a
documentation choice: the reviewer can see what *per-silo override*
looks like.

### 2.2 `meta.json`

```json
{
  "name": "mod107_cross_silo_fedavg",
  "resource_spec": {},
  "deploy_map": {
    "app_server": ["server"],
    "app_hospital_a": ["hospital-a"],
    "app_hospital_b": ["hospital-b"],
    "app_hospital_c": ["hospital-c"]
  },
  "min_clients": 3,
  "mandatory_clients": ["hospital-a", "hospital-b", "hospital-c"]
}
```

`mandatory_clients` is the cross-silo "no quorum-shrinking" flag — the
server will refuse to start the round without every named silo. This
is the FL analogue of a joint-controller agreement: every named party
is in, or the round does not run.

### 2.3 Server config (`app_server/config/config_fed_server.json`)

```json
{
  "format_version": 2,
  "num_rounds": 10,
  "min_clients": 3,
  "workflows": [
    {
      "id": "scatter_and_gather",
      "path": "nvflare.app_common.workflows.scatter_and_gather.ScatterAndGather",
      "args": {
        "min_clients": 3,
        "num_rounds": 10,
        "wait_time_after_min_received": 10,
        "aggregator_id": "aggregator",
        "persistor_id": "persistor",
        "shareable_generator_id": "shareable_generator",
        "train_task_name": "train",
        "train_timeout": 0
      }
    }
  ],
  "components": [
    {
      "id": "persistor",
      "path": "nvflare.app_opt.pt.file_model_persistor.PTFileModelPersistor",
      "args": { "model": { "path": "nvflare.app_common.pt.ptnet.PTNet" } }
    },
    {
      "id": "shareable_generator",
      "path": "nvflare.app_common.shareablegenerators.full_model_shareable_generator.FullModelShareableGenerator",
      "args": {}
    },
    {
      "id": "aggregator",
      "path": "nvflare.app_common.aggregators.intime_accumulate_model_aggregator.InTimeAccumulateWeightedAggregator",
      "args": { "expected_data_kind": "WEIGHT_DIFF" }
    }
  ]
}
```

Key decisions:

- **`ScatterAndGather`** is FLARE's built-in FedAvg-like controller:
  scatter the global model to every mandatory silo, wait for every
  silo's response, aggregate, persist, repeat.
- **`InTimeAccumulateWeightedAggregator`** with `WEIGHT_DIFF` means
  each silo returns a *delta* instead of a full weight vector. For
  cross-silo FL this is the defensible default: deltas are smaller
  over the wire and make it slightly harder to recover the client's
  starting point from a single round.
- **`wait_time_after_min_received = 10 s`** is small on purpose. In
  cross-silo FL every silo is expected to respond; the wait is for
  network jitter, not for dropouts. The cross-device analogue would be
  much larger (minutes).
- **`PTFileModelPersistor`** is used so the server emits a checkpoint
  per round. That checkpoint is what the DPIA will later cite as
  "the global model artifact," and the per-round diff is what the
  threat model attaches to for inference attacks.

### 2.4 Client config (`app_hospital_a/config/config_fed_client.json`)

```json
{
  "format_version": 2,
  "executors": [
    {
      "tasks": ["train"],
      "executor": {
        "path": "trainer.LocalTrainer",
        "args": {
          "local_epochs": 1,
          "lr": 0.01,
          "data_path": "/data/hospital_a",
          "batch_size": 32
        }
      }
    }
  ],
  "task_result_filters": [],
  "task_data_filters": [],
  "components": []
}
```

The `data_path` is **different per silo** — each hospital holds its
own partition and FLARE's deploy map scopes which app lands where.
The cross-silo property that "every silo trains only on its own
data, never ships raw data" is enforced here by the filesystem
layout, not by Python.

### 2.5 Trainer executor (`app_hospital_a/custom/trainer.py`)

```python
from pathlib import Path

import torch
from torch import nn, optim
from torch.utils.data import DataLoader

from nvflare.apis.executor import Executor
from nvflare.apis.fl_constant import ReturnCode
from nvflare.apis.fl_context import FLContext
from nvflare.apis.shareable import Shareable, make_reply
from nvflare.apis.signal import Signal
from nvflare.app_common.app_constant import AppConstants
from nvflare.app_common.abstract.model import ModelLearnable
from nvflare.app_common.pt.ptnet import PTNet


class LocalTrainer(Executor):
    def __init__(
        self, local_epochs: int = 1, lr: float = 0.01,
        data_path: str = "", batch_size: int = 32,
    ):
        super().__init__()
        self.local_epochs = int(local_epochs)
        self.lr = float(lr)
        self.data_path = Path(data_path)
        self.batch_size = int(batch_size)
        self.device = "cuda" if torch.cuda.is_available() else "cpu"
        self.model = PTNet()

    def _load_local_data(self) -> DataLoader:
        # Each silo reads its own on-disk partition. This is the line
        # that enforces "data stays home" — the silo never serialises
        # a batch of rows into a Shareable.
        tensors = torch.load(self.data_path / "train.pt")
        dataset = torch.utils.data.TensorDataset(tensors["x"], tensors["y"])
        return DataLoader(dataset, batch_size=self.batch_size, shuffle=True)

    def execute(
        self, task_name: str, shareable: Shareable,
        fl_ctx: FLContext, abort_signal: Signal,
    ) -> Shareable:
        if task_name != AppConstants.TASK_TRAIN:
            return make_reply(ReturnCode.TASK_UNKNOWN)

        # 1. Load the global weights from the server.
        global_weights = ModelLearnable.from_shareable(shareable)
        self.model.load_state_dict(global_weights, strict=True)
        starting = {k: v.detach().clone() for k, v in self.model.state_dict().items()}

        # 2. Train locally for `local_epochs` passes on the silo's data.
        self.model.to(self.device).train()
        loader = self._load_local_data()
        opt = optim.SGD(self.model.parameters(), lr=self.lr)
        loss_fn = nn.CrossEntropyLoss()
        n_examples = 0
        for _ in range(self.local_epochs):
            for x, y in loader:
                if abort_signal.triggered:
                    return make_reply(ReturnCode.TASK_ABORTED)
                x, y = x.to(self.device), y.to(self.device)
                opt.zero_grad()
                loss = loss_fn(self.model(x), y)
                loss.backward()
                opt.step()
                n_examples += y.size(0)

        # 3. Package the *delta* for the aggregator. Weight-diffs are
        #    what the server config declared as the expected kind.
        delta = {k: (v.detach().cpu() - starting[k].cpu())
                 for k, v in self.model.state_dict().items()}

        reply = ModelLearnable.to_shareable(delta)
        reply.set_header("num_examples", n_examples)
        reply.set_return_code(ReturnCode.OK)
        return reply
```

Key points for the reviewer:

- **`data_path` is per-silo and never travels.** The Shareable only
  carries weight deltas. A reviewer who reads this code can audit
  that raw records are not exfiltrated.
- **`num_examples` is set in the header.** FLARE's weighted aggregator
  uses this. In cross-silo FL this is usually the real row count;
  user-level DP-FL ([[exercise-05]]) replaces it with a user count.
- **Deltas, not weights.** This matches the server's
  `expected_data_kind = WEIGHT_DIFF` and makes the aggregator's
  weighted sum semantically a FedAvg update.
- **Abort signalling is honoured.** Real silos may be interrupted;
  returning `TASK_ABORTED` cleanly keeps the round from hanging the
  whole federation.

### 2.6 Provisioning (what identity actually looks like)

NVIDIA FLARE ships a **provisioner** that produces a startup kit per
site (server + each silo). The kit contains that silo's X.509 cert,
private key, root CA, and start-up scripts. The command is roughly:

```bash
nvflare provision -p project.yml
```

With a project file such as:

```yaml
api_version: 3
name: mod107_cross_silo
participants:
  - name: server
    type: server
    org: coordinator
    fed_learn_port: 8002
    admin_port: 8003
  - name: hospital-a
    type: client
    org: hospital-a
  - name: hospital-b
    type: client
    org: hospital-b
  - name: hospital-c
    type: client
    org: hospital-c
  - name: admin@coordinator
    type: admin
    org: coordinator
    role: project_admin
```

What the reviewer cares about:

- **Every silo is a named, provisioned identity with its own cert.**
  There is no "anonymous silo." This is the system-level
  manifestation of the cross-silo trust model from [[exercise-01]].
- **The server has its own identity.** The server is the aggregator,
  not a silo; its compromise has a different threat profile and the
  DPIA must distinguish.
- **The admin role is separate.** Operations on the federation
  (deploy job, abort run) are performed via an admin role, not via
  the silo roles — a baseline separation of duties.

### 2.7 Run manifest (produced after the run)

```json
{
  "framework": "NVIDIA FLARE",
  "topology": "cross-silo",
  "participants": [
    { "name": "hospital-a", "role": "client", "cert_fingerprint": "..." },
    { "name": "hospital-b", "role": "client", "cert_fingerprint": "..." },
    { "name": "hospital-c", "role": "client", "cert_fingerprint": "..." },
    { "name": "server",     "role": "aggregator", "cert_fingerprint": "..." }
  ],
  "workflow": "ScatterAndGather (FedAvg over WEIGHT_DIFF)",
  "aggregator": "InTimeAccumulateWeightedAggregator",
  "rounds": 10,
  "min_clients": 3,
  "mandatory_clients": ["hospital-a", "hospital-b", "hospital-c"],
  "per_silo_examples": {
    "hospital-a": 12043,
    "hospital-b": 8971,
    "hospital-c": 14200
  },
  "unit_of_privacy": "silo (institution)",
  "neighbouring_relation": "add/remove one silo — contractual, enforced by data-sharing agreement",
  "differential_privacy": "NOT ENGAGED in this exercise; layer on per exercise-05 if cross-silo leakage must be bounded",
  "secure_aggregation": "NOT ENGAGED in this exercise; a 3-party deployment is small enough for an MPC or TEE path",
  "transport": "mTLS via FLARE-provisioned PKI",
  "nist_ai_rmf_mapping": {
    "govern": "Joint controller agreement, DPIA per silo, admin role separation",
    "map":    "Per-silo data inventory + partition spec",
    "measure": "Per-round accuracy/F1 (weighted), delta L2 norms per silo",
    "manage": "Per-silo audit log of received weight-deltas and server replies"
  },
  "atlas_techniques_in_scope": [
    "AML.T0010 ML Supply Chain Compromise (silo code)",
    "AML.T0018 Backdoor ML Model (poisoned update from a silo)",
    "AML.T0020 Data from Local System (compromised silo exfil)",
    "AML.T0024 Exfiltration via ML Inference API (post-deployment)"
  ]
}
```

## 3. Validation steps

1. **Provisioning smoke test.** `nvflare provision -p project.yml`
   produces one startup kit per site under `workspace/`. Each kit
   contains `startup/` scripts, a site-specific cert, and the
   project's root CA.
2. **Start order.** Start the server first (`server/startup/start.sh`),
   then every silo. The server log should show three clients
   connecting via mTLS before the job is submitted.
3. **Job submission.** From the admin console,
   `submit_job nvflare-cross-silo-job`. The job state transitions
   `SUBMITTED → DISPATCHED → RUNNING → FINISHED`.
4. **All-silo participation.** The server log for each round must show
   every mandatory silo contributing a Shareable. If one silo is
   missing, the round should **not** proceed (that is the point of
   `mandatory_clients`); if it does, the server config is wrong.
5. **Delta shape sanity.** On any silo, print the L2 norm of the
   serialised delta before it leaves the executor; compare to the L2
   of the full weights. Deltas should be orders of magnitude smaller
   after the first few rounds; if they stay as large as the full
   weights, the server is probably not loading the previous round's
   state (persistor misconfig).
6. **No raw data on the wire.** Dump one round's Shareable from the
   wire log; confirm it is a float tensor dict, not a batch of
   features/labels. This is the audit step that proves raw data does
   not leave the silo.
7. **Manifest completeness.** The run manifest names the
   neighbouring relation at the silo level and explicitly states the
   DP / secure-aggregation posture. For the baseline cross-silo run
   both say "NOT ENGAGED" — and that is an acceptable baseline
   provided the contractual claim (DPIA, DUA) carries the privacy
   story.
8. **Resume.** Abort mid-round; restart; the persistor should let
   the job resume from the last persisted global model without
   starting round 1 again.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Job structure uses per-silo app folders and a `deploy_map` | 2 | Shared code across silos is fine; separate configs are required |
| `mandatory_clients` lists every silo and `min_clients` matches | 1 | Cross-silo semantics |
| Server uses `ScatterAndGather` with `WEIGHT_DIFF` and a weighted aggregator | 2 | Full-weights aggregation is acceptable with a documented rationale |
| Trainer loads per-silo data from a per-silo path; never puts raw features in a Shareable | 1 | — |
| Trainer sets `num_examples` header | 1 | Weighted mean only works if this is right |
| Startup kits are provisioned via `nvflare provision` (mTLS, named identities) | 1 | No anonymous silos |
| Run manifest names the unit of privacy at the silo level and the neighbouring relation | 1 | — |
| Run manifest explicitly states the DP / secure-aggregation posture | 1 | "NOT ENGAGED" is acceptable with the contractual claim documented |

Automatic failures:

- One `app/` folder shared across silos with no `deploy_map` — the
  exercise cannot distinguish "silo A trained on silo B's data" from
  a misconfiguration.
- Shipping raw rows inside a Shareable (any float/int features
  keyed by anything other than a model parameter name).
- Reporting a user-level DP ε on a cross-silo system without a
  per-user clip; the mechanism must match the unit.
- Running without TLS ("insecure" FLARE start) and treating the
  output as a credible cross-silo artifact.

## 5. Common mistakes

- **Cross-silo is not "FedAvg with three clients."** The forcing
  property is identity + contractual trust, not headcount. The
  `mandatory_clients` list, per-silo provisioning, and per-silo
  app folders are what make this exercise cross-silo; take any of
  those out and the system is a small cross-device toy. See
  [[exercise-01]] for the topology memo.
- **Treating `num_rounds` and `min_clients` as decorative.** In
  cross-silo FL, `min_clients == len(mandatory_clients)`. A deployment
  that silently proceeds without a silo is hiding a data-governance
  incident ("why did hospital-b not participate?") behind a training
  log.
- **Using `WEIGHTS` instead of `WEIGHT_DIFF`.** Full-weights
  aggregation works; it's just more revealing because any single
  silo's full weight vector trivially reconstructs its local update
  given the previous round's global. `WEIGHT_DIFF` is not a privacy
  mechanism — only secure aggregation and DP-FL are — but it does
  reduce accidental exposure.
- **Skipping the persistor.** Without a model persistor, resuming
  mid-run is impossible and the artifact "round-N global model" has
  no corresponding file on disk for the DPIA to cite.
- **Colocating admin with a silo.** An admin role is for running the
  federation; a silo role is for running training. Combining them
  collapses separation of duties.
- **Confusing cross-silo FL with multi-tenant training.** Cross-silo
  FL means *data stays home* and *updates are shared*. Multi-tenant
  training on a shared cluster means data is *pooled* under a
  governance boundary. The code review step that proves which one
  you have is "trace the path a batch of raw features takes." In
  cross-silo FL, that path ends inside the silo.
- **Omitting the ATLAS mapping.** The cross-silo threat model is
  about compromised silos and poisoned updates; the manifest should
  list the ATLAS techniques in scope (supply-chain compromise,
  backdoor, exfiltration) so a reviewer can run through them one
  by one.

## 6. References

- Roth, H. R.; Cheng, Y.; Wen, Y.; et al. *NVIDIA FLARE: Federated
  Learning from Simulation to Real-World.* arXiv:2210.13291.
  <https://arxiv.org/abs/2210.13291>
- NVIDIA FLARE documentation. <https://nvflare.readthedocs.io/>
- NVIDIA FLARE project repository and example jobs.
  <https://github.com/NVIDIA/NVFlare>
- McMahan, H. B. et al. *Communication-Efficient Learning of Deep
  Networks from Decentralized Data.* AISTATS 2017.
  <https://arxiv.org/abs/1602.05629>
- Kairouz, P.; McMahan, H. B.; et al. *Advances and Open Problems in
  Federated Learning.* FnTML 2021. <https://arxiv.org/abs/1912.04977>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
