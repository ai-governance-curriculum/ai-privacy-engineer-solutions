# Exercise 04 — LLM Unlearning with Gradient Ascent — Reference Solution

## 1. Solution overview

This is an **implementation exercise** on approximate unlearning in
large language models. Unlike the convex-regime influence-function
method ([[exercise-03]]) or the training-restructuring SISA method
([[exercise-02]]), gradient-ascent unlearning takes a *fine-tuned
LLM* and moves its parameters so that the model no longer produces
the content in a designated **forget set** `D_f`, while
preserving its competence on a **retain set** `D_r`. The entry
points are the family of methods studied in Jang et al. 2022
("Knowledge Unlearning for Mitigating Privacy Risks in Language
Models"), Yao et al. 2023 ("Large Language Model Unlearning"), and
Zhang et al. 2024 ("Negative Preference Optimization", NPO).

The deliverable has three parts:

1. **A reference implementation** of three unlearning losses — plain
   gradient ascent on `D_f`, gradient difference (ascent on `D_f` +
   descent on `D_r`), and NPO (a reference-model-anchored objective
   that prevents parameter collapse). Each is wired against a
   Hugging Face `transformers` causal-LM.
2. **A benchmark harness** that evaluates each method on the pair of
   metrics the literature standardised: **Forget Quality** (how
   effectively the forget-set content is suppressed) and **Model
   Utility** (how well the retain-set competence survives). The
   actual metric-set is TOFU-style ([[exercise-05]]) so the two
   exercises share evaluation code.
3. **A scope + limits note** stating the regime in which
   gradient-ascent unlearning is defensible, the collapse failure
   mode, and the governance implications (why Art. 17 compliance
   from LLM unlearning alone is almost never sufficient; the
   runbook in [[exercise-01]] always touches more surfaces).

The implementation uses PyTorch + Hugging Face `transformers`. For
model scale the reference uses a small checkpoint (TinyLlama,
Phi-2, or any 1–7 B causal LM) so the exercise runs on a single
GPU in minutes; the methodology is the same at 70 B.

Governance framing (NIST AI RMF, MITRE ATLAS, OWASP ML Top 10) is
the same as the rest of the module.

## 2. Worked implementation

### 2.1 Loss family

Fix a forget set `D_f = {(q_i, a_i)}` of question–answer pairs the
learner wants the model to no longer produce, and a retain set
`D_r` of QA pairs whose performance must be preserved. Let
`ℓ_CE(x; θ)` be token-level cross-entropy, and let `θ_ref` be a
frozen copy of the pre-unlearning parameters.

**Plain gradient ascent (GA).**

        L_GA(θ)  =  − E_{x ∼ D_f} [ ℓ_CE(x; θ) ]

Minimising `L_GA` ascends the loss on `D_f`. Pure GA collapses the
model within tens of steps — perplexity on *everything* blows up.

**Gradient difference (GD).**

        L_GD(θ)  =  − E_{x ∼ D_f} ℓ_CE(x; θ)
                    + α · E_{x ∼ D_r} ℓ_CE(x; θ)

The retain term holds utility back. `α` is the single knob: too
small and the model collapses, too large and the forget term never
moves the forget-set probability.

**Negative Preference Optimization (NPO).** Zhang et al. 2024
reformulate DPO so that the "preferred" answer is the model's own
current behaviour and the "rejected" answer is the forget content.
With `β > 0` the temperature and `π_ref` the frozen reference,

        L_NPO(θ) = − (2 / β) · E_{x ∼ D_f} [
            log σ( − β · log( π_θ(x) / π_ref(x) ) )
        ]
                   + α · E_{x ∼ D_r} ℓ_CE(x; θ).

The reference-anchored term prevents unbounded drift: once
`π_θ(x) ≪ π_ref(x)` on `D_f`, the gradient magnitude shrinks and
the optimisation stops hammering the weights. NPO is the current
default in the TOFU leaderboard and is what the reference uses.

### 2.2 Reference implementation

```python
# llm_unlearn.py — pedagogical reference. Not production: see §5.
from __future__ import annotations

import copy
from dataclasses import dataclass
from typing import Iterable, Dict

import torch
from torch import nn
from torch.nn import functional as F
from torch.utils.data import DataLoader

from transformers import AutoModelForCausalLM, AutoTokenizer


@dataclass
class UnlearnConfig:
    method: str = "npo"        # "ga" | "gd" | "npo"
    retain_alpha: float = 1.0
    npo_beta: float = 0.1
    lr: float = 1e-5
    steps: int = 200
    max_grad_norm: float = 1.0


# ---------- shared logprob helper -----------------------------------------
def masked_logprob(
    model: nn.Module, input_ids: torch.Tensor, labels: torch.Tensor, pad_id: int
) -> torch.Tensor:
    """Sum of log p(y|x) over non-pad label tokens, per example."""
    outputs = model(input_ids=input_ids, labels=None)
    logits = outputs.logits[:, :-1, :]                 # predict token at t+1
    targets = labels[:, 1:]
    logp = F.log_softmax(logits, dim=-1)
    gathered = logp.gather(-1, targets.clamp_min(0).unsqueeze(-1)).squeeze(-1)
    mask = (targets != pad_id).float()
    return (gathered * mask).sum(dim=-1)


def masked_ce(
    model: nn.Module, input_ids: torch.Tensor, labels: torch.Tensor, pad_id: int
) -> torch.Tensor:
    """Negative mean log-prob per example, averaged over the batch."""
    lp = masked_logprob(model, input_ids, labels, pad_id)
    tokcnt = (labels[:, 1:] != pad_id).float().sum(dim=-1).clamp_min(1.0)
    return (-lp / tokcnt).mean()


# ---------- the three loss variants --------------------------------------
def unlearn_step_ga(model, batch_f, pad_id) -> torch.Tensor:
    return -masked_ce(model, batch_f["input_ids"], batch_f["labels"], pad_id)


def unlearn_step_gd(model, batch_f, batch_r, pad_id, alpha) -> torch.Tensor:
    forget = -masked_ce(model, batch_f["input_ids"], batch_f["labels"], pad_id)
    retain = masked_ce(model, batch_r["input_ids"], batch_r["labels"], pad_id)
    return forget + alpha * retain


def unlearn_step_npo(model, model_ref, batch_f, batch_r, pad_id, alpha, beta):
    lp_theta = masked_logprob(model, batch_f["input_ids"], batch_f["labels"], pad_id)
    with torch.no_grad():
        lp_ref = masked_logprob(model_ref, batch_f["input_ids"], batch_f["labels"], pad_id)
    forget = -(2.0 / beta) * F.logsigmoid(-beta * (lp_theta - lp_ref)).mean()
    retain = masked_ce(model, batch_r["input_ids"], batch_r["labels"], pad_id)
    return forget + alpha * retain


# ---------- unlearning loop ----------------------------------------------
def unlearn(
    model: nn.Module,
    tokenizer,
    loader_forget: DataLoader,
    loader_retain: DataLoader,
    cfg: UnlearnConfig = UnlearnConfig(),
) -> Dict:
    pad_id = tokenizer.pad_token_id
    model_ref = copy.deepcopy(model) if cfg.method == "npo" else None
    if model_ref is not None:
        for p in model_ref.parameters():
            p.requires_grad = False

    opt = torch.optim.AdamW(model.parameters(), lr=cfg.lr)

    it_r = iter(loader_retain)
    history = []
    for step in range(cfg.steps):
        batch_f = next(iter(loader_forget))
        try:
            batch_r = next(it_r)
        except StopIteration:
            it_r = iter(loader_retain)
            batch_r = next(it_r)

        opt.zero_grad()
        if cfg.method == "ga":
            loss = unlearn_step_ga(model, batch_f, pad_id)
        elif cfg.method == "gd":
            loss = unlearn_step_gd(model, batch_f, batch_r, pad_id, cfg.retain_alpha)
        elif cfg.method == "npo":
            loss = unlearn_step_npo(
                model, model_ref, batch_f, batch_r, pad_id,
                cfg.retain_alpha, cfg.npo_beta,
            )
        else:
            raise ValueError(cfg.method)
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), cfg.max_grad_norm)
        opt.step()

        history.append({"step": step, "loss": float(loss.detach())})
    return {"history": history}
```

Minimal data format — each `batch` is a dict with `input_ids` and
`labels` both of shape `[B, T]`; `labels == pad_id` on positions
that should be ignored. The reference assumes the dataset is
pre-tokenised and padded.

### 2.3 Benchmark harness (TOFU-style)

The benchmark uses the same metric families as TOFU
([[exercise-05]]):

- **Forget Quality (FQ).** Per-prompt, sample the model's answer
  and compute ROUGE-L against the forget-set ground truth. Lower is
  better. Report before/after per method.
- **Model Utility (MU).** Per-prompt from the retain set, compute
  ROUGE-L against the retain-set ground truth. Higher is better.
  Report before/after per method.
- **Membership Inference (MIA).** Treat the forget set as the
  "member" class; a MIA (loss-based or LiRA, [[mod-105]]) trained
  on the pre-unlearn model should drop to near-chance after
  unlearning. If MIA attack AUC stays well above 0.5, the model
  still memorises the forget content and the unlearning is
  cosmetic.

```python
# bench.py — Forget-quality × Model-utility × MIA
import evaluate   # pip install evaluate rouge_score

rouge = evaluate.load("rouge")

def rouge_l(preds, refs) -> float:
    return rouge.compute(predictions=preds, references=refs, rouge_types=["rougeL"])["rougeL"]

def generate(model, tokenizer, prompt: str, max_new: int = 128) -> str:
    ids = tokenizer(prompt, return_tensors="pt").input_ids.to(model.device)
    out = model.generate(ids, max_new_tokens=max_new, do_sample=False)
    return tokenizer.decode(out[0, ids.size(1):], skip_special_tokens=True)

def score(model, tokenizer, samples):
    preds = [generate(model, tokenizer, s["prompt"]) for s in samples]
    refs = [s["answer"] for s in samples]
    return rouge_l(preds, refs)
```

Reference expectations on a toy forget set (fictitious author bios,
50 QA pairs; retain set 500 QA pairs; 1.5-B model; 200 steps):

| Method | FQ ↓ (before → after) | MU ↑ (before → after) | MIA AUC ↓ (before → after) |
|---|---|---|---|
| Plain GA | 0.9 → 0.1 | 0.8 → 0.2 (collapsed) | 0.78 → 0.52 but total model collapse |
| GD (α = 1) | 0.9 → 0.3 | 0.8 → 0.7 | 0.78 → 0.58 |
| NPO (α = 1, β = 0.1) | 0.9 → 0.2 | 0.8 → 0.78 | 0.78 → 0.54 |

NPO is the cleanest trade-off: FQ drops to near-random while MU
holds within a few points of the pre-unlearning value. Pure GA
collapses the model; GD lands between them. Numbers will vary by
model and knob settings; the *shape* of the table is what the
exercise asks the learner to reproduce.

### 2.4 Operational framing

LLM unlearning, even at its best, is **approximate**. The audit
posture is:

- Record `method`, step count, `α`, `β`, forget/retain set manifests
  (hashes), per-sample FQ/MU deltas, and MIA AUC deltas. This is
  the certificate that feeds the audit trail ([[exercise-07]]).
- Do **not** claim Art. 17 fulfilment from the LLM-unlearning step
  alone. The DSR runbook ([[exercise-01]]) must still touch
  training-set storage, prompt/response logs, RAG indexes, feature
  stores, and derivative caches — nine surfaces in total.
- For high-assurance erasure, pair the unlearning with an inference
  filter and a periodic from-scratch retraining cadence (see the
  decision tree in [[exercise-06]]).

## 3. Validation steps

1. **FQ/MU curves.** Plot Forget Quality and Model Utility across
   steps for each method. FQ should fall monotonically (or near so);
   MU under NPO/GD should stay within a chosen tolerance (e.g.,
   10 % relative drop). A plateauing FQ before the budget runs out
   is a sign of hyperparameter undershoot.
2. **Collapse canary.** Include a few general-knowledge prompts
   ("What is the capital of France?") in the retain evaluation.
   Pure GA will degrade them within ~50 steps; NPO should not.
3. **Reference-model drift.** In NPO, verify `model_ref` is frozen
   (`requires_grad = False` on all params). If training also
   updates the reference, the anchor is gone and NPO degenerates
   into GD-like dynamics.
4. **MIA sanity.** Run a loss-based MIA ([[mod-105]]) against the
   forget set before and after. Post-unlearning AUC should drop
   toward 0.5; if it stays high, the unlearning is cosmetic.
5. **Hyperparameter sensitivity.** Sweep `α` over
   `{0.25, 0.5, 1.0, 2.0}` for NPO; report the Pareto front of
   (FQ, MU). A single-point report is not enough to defend the
   chosen knob setting.
6. **Deterministic eval.** Use greedy decoding or a fixed seed for
   sampling during FQ/MU measurement; stochastic decoding injects
   noise that masks small deltas.
7. **Audit-certificate payload.** The `history` object plus the
   benchmark report must together contain: method, α, β, step count,
   forget-set hash, retain-set hash, FQ before/after per sample,
   MU before/after per sample, MIA AUC before/after. These are the
   fields [[exercise-07]] expects.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Three loss variants (GA, GD, NPO) implemented and selectable | 2 | §2.2 |
| NPO uses a frozen reference model and reference-anchored objective | 2 | §2.1, §3.3 |
| Benchmark reports Forget Quality, Model Utility, and MIA AUC before/after | 2 | §2.3, §3.4 |
| Hyperparameter sweep and Pareto report present | 1 | §3.5 |
| Scope note: LLM unlearning ≠ Art. 17 compliance on its own | 1 | §2.4 |
| Collapse canary demonstrates pure-GA failure | 1 | §3.2 |
| Audit-certificate fields populated for [[exercise-07]] | 1 | — |

Automatic failures:

- Running only pure GA and reporting "the model forgets."
- NPO without a frozen reference (parameter collapse).
- Claiming Art. 17 fulfilment from the unlearning step alone.
- Evaluating with stochastic decoding and reporting a single number.
- No retain-set evaluation (utility check is the half that's easy
  to skip).

## 5. Common mistakes

- **Over-ascending.** Even with a retain term, a few extra epochs
  can push the forget-set logits to `−∞` and the retain-set loss
  to a non-invertible regime. Clip gradients and monitor MU every
  step; stop when FQ has crossed its target rather than training
  to a fixed step budget.
- **Overfitting the retain anchor.** If `D_r` is small, GD/NPO can
  *memorise* the retain set and show spurious retain-side
  improvements. Evaluate MU on a held-out retain sample, not on
  `D_r` itself.
- **Mixing tokenizers between model and reference.** NPO assumes
  `π_ref` is the same model, pre-unlearning. If the reference is
  a different checkpoint, the log-ratio has an unrelated baseline
  and the objective is nonsense.
- **Running NPO without `log_softmax` stability.** `log_softmax` is
  numerically stable; `softmax → log` is not. The reference uses
  `F.log_softmax` and gathers log-probs directly for this reason.
- **Confusing "forget" with "refuse."** Making the model respond
  "I cannot answer" to forget-set prompts is not unlearning — it is
  a *refusal policy*. A membership-inference attack can still
  succeed. The exercise's success metric is the *generation and the
  parameter-level forgetting*, not the surface response.
- **Dropping the retain set because it is slow.** The retain term
  is what makes the method operational; removing it recovers the
  collapse of pure GA. If compute is the issue, shrink the retain
  set rather than cutting it.
- **Pointing at a tiny loss as success.** Low `L_NPO` alone is not
  informative — the loss is designed to self-limit. The signal is
  FQ ↓ and MU unchanged.
- **Treating one random seed as a result.** Unlearning is a
  stochastic optimisation. Report mean + std across ≥ 3 seeds; a
  single-seed FQ can swing by factors of 2 between runs.
- **Claiming the model is "privately" forgotten.** Only the
  certified-removal variant ([[exercise-03]], §2.5) provides a
  provable guarantee. The gradient-ascent family is best-effort;
  the audit trail must say so.
- **Deploying without a filter.** Even after good FQ, a determined
  prompt can sometimes resurface forget-set content. Pair the
  unlearned model with an inference-time filter (RAG-privacy and
  policy filters per [[mod-109]]).

## 6. References

- Jang, J.; Yoon, D.; Yang, S.; Cha, S.; Lee, M.; Logeswaran, L.;
  Seo, M. *Knowledge Unlearning for Mitigating Privacy Risks in
  Language Models.* ACL 2023.
  <https://arxiv.org/abs/2210.01504>
- Yao, Y.; Xu, X.; Liu, Y. *Large Language Model Unlearning.*
  <https://arxiv.org/abs/2310.10683>
- Zhang, R.; Lin, Y.; Bai, Y.; Weng, X.; Shi, L.; Kailkhura, B.;
  Liu, Y.; Li, B. *Negative Preference Optimization: From
  Catastrophic Collapse to Effective Unlearning.* COLM 2024.
  <https://arxiv.org/abs/2404.05868>
- Maini, P.; Feng, Z.; Schwarzschild, A.; Lipton, Z. C.; Kolter,
  J. Z. *TOFU: A Task of Fictitious Unlearning for LLMs.*
  COLM 2024. <https://arxiv.org/abs/2401.06121> — the benchmark
  this exercise's eval harness mirrors; see [[exercise-05]].
- Rafailov, R.; Sharma, A.; Mitchell, E.; Ermon, S.; Manning, C. D.;
  Finn, C. *Direct Preference Optimization.* NeurIPS 2023.
  <https://arxiv.org/abs/2305.18290> — the DPO formulation NPO
  derives from.
- Hugging Face `transformers`.
  <https://huggingface.co/docs/transformers>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
