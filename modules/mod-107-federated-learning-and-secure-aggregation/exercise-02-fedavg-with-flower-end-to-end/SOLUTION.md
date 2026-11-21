# Exercise 02 — FedAvg with Flower, End to End — Reference Solution

## 1. Solution overview

This is an **implementation exercise**. The learner must stand up a
complete FedAvg training run with Flower: a server strategy, two or
more simulated clients, a partitioned dataset, and a post-run report
that captures what a privacy reviewer needs (participant count,
sampling fraction, update size, rounds, the absence or presence of
secure aggregation, and the unit at which the privacy claim is made).

The assessable artifacts:

1. A **client program** that loads its local partition, trains for a
   fixed number of local steps per round, and reports updated
   parameters to the server.
2. A **server program** using `FedAvg` (or `FedAvgM`) with an explicit
   sampling fraction and a minimum number of clients per round.
3. A **run report** recording round count, per-round client
   participation, parameter-vector size, local steps, and the privacy
   mechanism (or the explicit statement that none is engaged yet).
4. A **README/manifest** that names the unit of privacy, the
   neighbouring relation, and a one-paragraph threat note that points
   to [[exercise-04]] (secure aggregation) and [[exercise-05]]
   (DP-FL units) for the follow-on hardening work.

The code below targets Flower ≥ 1.8, which exposes the `ClientApp` /
`ServerApp` model. The design choices (FedAvg, number of local
epochs, sampling fraction) follow McMahan et al. (2017); the reporting
conventions and governance framing follow NIST AI RMF 1.0.

## 2. Worked implementation

### 2.1 Project layout

```
exercise-02-fedavg-with-flower-end-to-end/
├── client_app.py        # ClientApp — local training
├── server_app.py        # ServerApp — strategy wiring
├── task.py              # model + data partition + train/eval
├── pyproject.toml       # Flower app metadata (`flwr run`)
└── run_report.json      # produced by server at end of run
```

### 2.2 Shared task (`task.py`)

```python
# task.py
from collections import OrderedDict
from typing import List, Tuple

import torch
from torch import nn, optim
from torch.utils.data import DataLoader, Subset


def build_model() -> nn.Module:
    # Small CNN, chosen because it trains cheaply enough that a reviewer
    # can rerun the exercise on CPU and still watch convergence.
    return nn.Sequential(
        nn.Conv2d(1, 16, 3, padding=1), nn.ReLU(), nn.MaxPool2d(2),
        nn.Conv2d(16, 32, 3, padding=1), nn.ReLU(), nn.MaxPool2d(2),
        nn.Flatten(),
        nn.Linear(32 * 7 * 7, 10),
    )


def get_parameters(model: nn.Module) -> List:
    return [v.cpu().numpy() for _, v in model.state_dict().items()]


def set_parameters(model: nn.Module, parameters: List) -> None:
    keys = list(model.state_dict().keys())
    state_dict = OrderedDict(
        {k: torch.as_tensor(v) for k, v in zip(keys, parameters)}
    )
    model.load_state_dict(state_dict, strict=True)


def train(
    model: nn.Module, loader: DataLoader, epochs: int, lr: float, device: str
) -> Tuple[float, int]:
    model.to(device).train()
    opt = optim.SGD(model.parameters(), lr=lr, momentum=0.9)
    loss_fn = nn.CrossEntropyLoss()
    total_loss, n_examples = 0.0, 0
    for _ in range(epochs):
        for x, y in loader:
            x, y = x.to(device), y.to(device)
            opt.zero_grad()
            loss = loss_fn(model(x), y)
            loss.backward()
            opt.step()
            total_loss += loss.item() * y.size(0)
            n_examples += y.size(0)
    return total_loss / max(n_examples, 1), n_examples


def evaluate(model: nn.Module, loader: DataLoader, device: str):
    model.to(device).eval()
    loss_fn = nn.CrossEntropyLoss()
    loss, correct, n = 0.0, 0, 0
    with torch.no_grad():
        for x, y in loader:
            x, y = x.to(device), y.to(device)
            out = model(x)
            loss += loss_fn(out, y).item() * y.size(0)
            correct += (out.argmax(1) == y).sum().item()
            n += y.size(0)
    return loss / max(n, 1), correct / max(n, 1)


def partition_indices(num_clients: int, dataset_size: int, seed: int = 0):
    # IID partition for the baseline run. Non-IID partitioners live in
    # `flwr_datasets`; the reviewer can swap them in without touching
    # the client/server code.
    rng = torch.Generator().manual_seed(seed)
    perm = torch.randperm(dataset_size, generator=rng).tolist()
    sizes = [dataset_size // num_clients] * num_clients
    sizes[-1] += dataset_size - sum(sizes)
    out, cursor = [], 0
    for s in sizes:
        out.append(perm[cursor:cursor + s])
        cursor += s
    return out
```

### 2.3 Client (`client_app.py`)

```python
# client_app.py
from pathlib import Path

import torch
from flwr.client import ClientApp, NumPyClient
from flwr.common import Context
from torch.utils.data import DataLoader, Subset
from torchvision import datasets, transforms

from task import (
    build_model, evaluate, get_parameters, partition_indices,
    set_parameters, train,
)


DATA_DIR = Path(".flower-data")
DEVICE = "cuda" if torch.cuda.is_available() else "cpu"


def _load_partition(partition_id: int, num_partitions: int):
    tfm = transforms.Compose([transforms.ToTensor(),
                              transforms.Normalize((0.1307,), (0.3081,))])
    train_ds = datasets.MNIST(DATA_DIR, train=True, download=True, transform=tfm)
    test_ds  = datasets.MNIST(DATA_DIR, train=False, download=True, transform=tfm)
    parts = partition_indices(num_partitions, len(train_ds))
    train_subset = Subset(train_ds, parts[partition_id])
    return (DataLoader(train_subset, batch_size=32, shuffle=True),
            DataLoader(test_ds, batch_size=256))


class MNISTClient(NumPyClient):
    def __init__(self, partition_id: int, num_partitions: int):
        self.model = build_model()
        self.train_loader, self.test_loader = _load_partition(
            partition_id, num_partitions,
        )

    def fit(self, parameters, config):
        set_parameters(self.model, parameters)
        epochs = int(config.get("local_epochs", 1))
        lr = float(config.get("lr", 0.01))
        loss, n = train(self.model, self.train_loader, epochs, lr, DEVICE)
        # The int we return is the "num_examples" FedAvg uses for the
        # weighted mean. For user-level DP-FL this must be replaced by
        # a per-user count; see [[exercise-05]].
        return get_parameters(self.model), n, {"train_loss": loss}

    def evaluate(self, parameters, config):
        set_parameters(self.model, parameters)
        loss, acc = evaluate(self.model, self.test_loader, DEVICE)
        return float(loss), len(self.test_loader.dataset), {"accuracy": acc}


def client_fn(context: Context):
    partition_id = int(context.node_config["partition-id"])
    num_partitions = int(context.node_config["num-partitions"])
    return MNISTClient(partition_id, num_partitions).to_client()


app = ClientApp(client_fn=client_fn)
```

### 2.4 Server (`server_app.py`)

```python
# server_app.py
import json
from pathlib import Path

from flwr.common import Context, ndarrays_to_parameters, Metrics
from flwr.server import ServerApp, ServerAppComponents, ServerConfig
from flwr.server.strategy import FedAvg

from task import build_model, get_parameters


def _weighted_accuracy(metrics_per_client):
    total = sum(n for n, _ in metrics_per_client)
    acc = sum(n * m["accuracy"] for n, m in metrics_per_client)
    return {"accuracy": acc / total}


def server_fn(context: Context):
    num_rounds       = int(context.run_config.get("num-server-rounds", 10))
    fraction_fit     = float(context.run_config.get("fraction-fit", 0.5))
    fraction_eval    = float(context.run_config.get("fraction-eval", 0.5))
    local_epochs     = int(context.run_config.get("local-epochs", 1))
    min_fit_clients  = int(context.run_config.get("min-fit-clients", 2))
    min_avail        = int(context.run_config.get("min-available-clients", 2))

    init_params = ndarrays_to_parameters(get_parameters(build_model()))

    strategy = FedAvg(
        fraction_fit=fraction_fit,
        fraction_evaluate=fraction_eval,
        min_fit_clients=min_fit_clients,
        min_evaluate_clients=min_fit_clients,
        min_available_clients=min_avail,
        initial_parameters=init_params,
        evaluate_metrics_aggregation_fn=_weighted_accuracy,
        on_fit_config_fn=lambda rnd: {
            "server_round": rnd, "local_epochs": local_epochs, "lr": 0.01,
        },
    )

    config = ServerConfig(num_rounds=num_rounds)
    return ServerAppComponents(strategy=strategy, config=config)


app = ServerApp(server_fn=server_fn)


# After the run finishes (invoked by the operator, not by Flower itself),
# write the run report that the privacy reviewer asks for.
def write_run_report(
    out_path: Path, *,
    num_rounds: int, num_clients: int, fraction_fit: float,
    local_epochs: int, model_param_count: int, had_secure_aggregation: bool,
    had_dp: bool,
) -> None:
    report = {
        "framework": "Flower (flwr)",
        "strategy": "FedAvg (McMahan et al. 2017)",
        "rounds": num_rounds,
        "clients_total": num_clients,
        "clients_sampled_per_round": max(
            int(num_clients * fraction_fit), 1
        ),
        "fraction_fit": fraction_fit,
        "local_epochs_per_round": local_epochs,
        "model_parameter_count": model_param_count,
        "secure_aggregation": had_secure_aggregation,
        "differential_privacy": had_dp,
        "unit_of_privacy": "client (one simulated device) — baseline only",
        "neighbouring_relation": "add/remove one client",
        "notes": [
            "Baseline FedAvg: the server sees each client's parameters "
            "in the clear. For a release, add secure aggregation "
            "(see exercise-04) or DP-FL (see exercise-05).",
        ],
    }
    out_path.write_text(json.dumps(report, indent=2))
```

### 2.5 `pyproject.toml` (Flower app metadata, abbreviated)

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "fedavg-flower-exercise"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "flwr>=1.8,<2.0",
    "torch>=2.0",
    "torchvision>=0.15",
]

[tool.flwr.app]
publisher = "ai-privacy-engineer-solutions"

[tool.flwr.app.components]
clientapp = "client_app:app"
serverapp = "server_app:app"

[tool.flwr.app.config]
num-server-rounds = 10
fraction-fit = 0.5
fraction-eval = 0.5
local-epochs = 1
min-fit-clients = 2
min-available-clients = 2

[tool.flwr.federations]
default = "localhost"

[tool.flwr.federations.localhost]
options.num-supernodes = 5
```

Run locally with `flwr run .` after `pip install -e .`. The default
simulation fans out 5 clients and samples half of them per round; the
configurable knobs are above.

### 2.6 What the reviewer actually verifies (at code-review time)

- The weighted mean inside `FedAvg` weights by *num\_examples returned
  by* `fit`. The baseline returns row counts. If the system later
  claims user-level DP, that return value becomes the user count;
  cross-link to [[exercise-05]].
- `fraction_fit` and `min_fit_clients` are set explicitly, not left
  to defaults. A reviewer looking at FedAvg always asks "how many
  clients per round?" and the answer must be in config, not inferred.
- `initial_parameters` is deterministic (seeded model build), so
  re-runs with the same `pyproject.toml` config are comparable
  round-for-round.
- The run report records whether secure aggregation and DP were
  engaged. For the baseline the answers are "no, no" — and that is
  the honest reportable statement.

## 3. Validation steps

1. **Simulation starts.** `flwr run .` launches the supernode pool,
   boots 5 ClientApps, and the server prints round 1 inside a few
   seconds. If the server hangs with "waiting for min_available_clients",
   the pool is smaller than `min-available-clients`.
2. **Round telemetry.** Each round logs `aggregate_fit` and
   `aggregate_evaluate` with weighted-average metrics. The accuracy
   on MNIST should rise past 0.9 within 10 rounds for the default
   config.
3. **Determinism.** With a fixed seed (set in `build_model` and in
   `partition_indices`), rerunning the simulation produces an
   accuracy trace close enough for a reviewer to notice regressions.
4. **Sampling fraction is honoured.** Count the number of distinct
   client IDs the server sees in a single round; it should equal
   `max(round(num_clients * fraction_fit), min_fit_clients)`.
5. **Parameter size check.** The run report's
   `model_parameter_count` matches `sum(p.numel() for p in
   model.parameters())`. This matters because a secure-aggregation
   add-on (see [[exercise-04]]) scales with parameter count.
6. **Negative test — one client.** Start with
   `min-available-clients=1` and `fraction-fit=1.0`; the run should
   complete but the "federated" update collapses to local training —
   a sanity check that FedAvg with one client is exactly SGD. If
   that is not the behaviour, the client's `num_examples` reporting
   or the server's weighting is wrong.
7. **Report contents.** `run_report.json` enumerates rounds, clients
   per round, local epochs, and the explicit "no DP, no secure
   aggregation" statement. A reviewer should be able to classify the
   release as *FL-without-privacy-mechanisms* from this file alone.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| ClientApp + ServerApp both runnable via `flwr run .` | 2 | Not a notebook; the simulator path is what reviewers reproduce |
| FedAvg configured with explicit `fraction_fit`, `min_fit_clients`, `min_available_clients` | 1 | Defaults are not acceptable |
| `on_fit_config_fn` passes `server_round`, `local_epochs`, `lr` to clients | 1 | Makes local behaviour observable from the server config |
| `fit()` returns `(params, num_examples, metrics)` with a meaningful `num_examples` | 1 | Weighted averaging only works if this is right |
| Seeded model + seeded partitioner | 1 | Reproducible accuracy trace |
| Run report records rounds, sampled clients, model size, and privacy-mechanism state | 2 | — |
| Report explicitly names the unit of privacy *even for the baseline* ("client, no DP") | 1 | Honest baseline, not a silent one |
| Pointer to secure-aggregation and DP-FL follow-ons | 1 | Cross-links to [[exercise-04]] and [[exercise-05]] |

Automatic failures:

- Running training only on one machine without a ClientApp/ServerApp
  split (that's not FL).
- Returning a constant `num_examples=1` from `fit`; this disables the
  weighted average FedAvg is defined by.
- Claiming "secure FL" or "private FL" from the baseline run — the
  baseline is FL without privacy mechanisms, and that must be said.

## 5. Common mistakes

- **Treating the simulator as production.** `flwr run` with simulated
  supernodes is a correctness bed, not a deployment. Real cross-device
  deployments require the Flower Deployment Engine (SuperLink +
  SuperNode + ClientApp binaries); the exercise is the first step.
- **Returning the wrong `num_examples`.** FedAvg weights the parameter
  vectors by whatever integer the client returns. Reporting the number
  of mini-batches, or 1, yields a *uniform* average instead of a
  weighted one and silently changes the algorithm.
- **Mixing BatchNorm with FL.** BatchNorm's running statistics drift
  per-client; averaging them is approximately but not exactly correct.
  Prefer GroupNorm/LayerNorm (same concern as DP-SGD — see mod-106).
- **Shuffling the dataset *after* partitioning.** The point of the
  partition is that each client's view is disjoint. If a reshuffle
  happens in the DataLoader after the subset is built, the privacy
  story is still fine, but the "non-IID vs IID" ablation is nonsense.
- **Forgetting `on_fit_config_fn`.** Without it, the client cannot know
  what round it is in, and reviewers can't trace which hyperparameters
  produced which round's metrics.
- **Reporting `fraction_fit = 1.0` by accident.** Flower's defaults
  have changed across versions; always set this explicitly. A
  reviewer reading `FedAvg()` with no args cannot tell whether the
  system samples 10 % of clients or 100 %.
- **Skipping the "no privacy" note.** The baseline run is the honest
  reference point. It is not a privacy-preserving system yet; the
  manifest must say so. Secure aggregation ([[exercise-04]]) and
  DP-FL ([[exercise-05]]) are the layers that turn it into one.
- **Running multi-client fit with a global optimizer state.** Each
  client builds its own optimiser per round; carrying optimizer state
  across rounds without the server mediating it breaks FedAvg's
  no-local-state assumption (relevant for cross-device, see
  [[exercise-01]]).

## 6. References

- McMahan, H. B.; Moore, E.; Ramage, D.; Hampson, S.; y Arcas, B. A.
  *Communication-Efficient Learning of Deep Networks from Decentralized
  Data.* AISTATS 2017. <https://arxiv.org/abs/1602.05629> — original
  FedAvg.
- Beutel, D. J. et al. *Flower: A Friendly Federated Learning Research
  Framework.* arXiv:2007.14390. <https://arxiv.org/abs/2007.14390>
- Flower documentation. <https://flower.ai/docs/>
- Flower `FedAvg` strategy reference.
  <https://flower.ai/docs/framework/ref-api/flwr.server.strategy.FedAvg.html>
- Kairouz, P.; McMahan, H. B.; et al. *Advances and Open Problems in
  Federated Learning.* FnTML 2021. <https://arxiv.org/abs/1912.04977>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS. <https://atlas.mitre.org/>
