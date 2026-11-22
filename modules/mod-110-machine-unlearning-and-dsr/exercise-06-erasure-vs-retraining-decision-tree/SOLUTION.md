# Exercise 06 — Erasure vs Retraining Decision Tree — Reference Solution

## 1. Solution overview

This is a **design exercise**. Given an incoming erasure request
(GDPR Art. 17 or an equivalent regime), the privacy engineer must
pick one of five response paths and defend the choice against a
reviewer. The deliverable is a **decision tree** — flowchart-shape,
with every branch labelled by observable facts about the request,
the system, and the data — plus a per-leaf playbook explaining what
the operator does, what evidence they log, and what the memo to
the subject says.

The assessable artifacts:

1. **The decision tree itself** (flowchart + a plain-text
   representation the operator can follow). Branches are keyed on
   observable facts, not on the operator's judgement.
2. **Five leaf playbooks**, one per response path:
   (A) Source-only deletion (no model surgery needed);
   (B) Inference filter (short-term, pre-retrain);
   (C) Exact unlearning (SISA, [[exercise-02]]);
   (D) Approximate unlearning (influence functions [[exercise-03]]
       or gradient-ascent LLM variants [[exercise-04]]);
   (E) Full retraining.
   Each playbook states inputs, actions, evidence, SLA, and
   escalation.
6. **A rationale document** per leaf: why that path dominates the
   alternatives for the request class, under which failure modes
   it degrades, and which runbook ([[exercise-01]]) it slots into.

The decision tree is grounded in GDPR Art. 17 (what must be done
and when), the exact/approximate taxonomy from Nguyen et al.
(2022) *Survey of Machine Unlearning* and Bourtoule et al. (2021)
*Machine Unlearning*, and the governance framing of NIST AI
RMF 1.0.

## 2. Worked answer

### 2.1 Five response paths — the taxonomy

| Path | When it applies | Cost | Guarantee | Audit strength |
|---|---|---|---|---|
| A — Source-only delete | Record is in source tables only; model never saw it (new record, not in last training run); or model is a frozen pre-trained artefact not fine-tuned on the record | Low | None at model level; full at source | Strong via DB logs |
| B — Inference filter | Short-term bridge until next retrain; model memorises the record but outputs can be filtered post-hoc; appropriate for text completion, retrieval | Low | Operational only (filter can be bypassed) | Weak at model level; strong at serving layer |
| C — Exact unlearning (SISA) | Training was run under SISA; the record maps to a single shard; retrain cost is ≤ `(1/S) · (R − r*) / R` of a full run | Medium | **Exact** (bit-identical to retrain minus record) | Strong — manifest + bit-identity check |
| D — Approximate unlearning | Training was not SISA; model is small enough for influence functions or an LLM with a known fine-tune cohort | Medium | Approximate; metric-reported (TOFU-style) | Medium — certified-removal variant gives a ε-DP guarantee |
| E — Full retraining | No SISA; high-assurance guarantee required; approximate method fails the operational gate; or batched erasure volume outstrips (C)/(D) economics | High | Exact | Strong |

The paths are not mutually exclusive in practice: a typical
production pipeline uses (B) on the serving layer as a bridge,
(C) or (D) on the model, and schedules (E) periodically as a
backstop.

### 2.2 The decision tree (plain text)

```
START: Art. 17 request accepted after identity verification (see [[exercise-01]]).

Q1  Was the record in the authoritative training store?
    ├── NO  → Path A (delete from source). Log; close the request.
    └── YES →

Q2  Was the model actually trained on the record?
    (Check training manifest; cross-reference last-retrain date vs. record ingest date.)
    ├── NO (record ingested after last training) →
    │       Path A for sources. Defer model action until next scheduled retrain.
    │       Log the deferral with an explicit retrain date.
    └── YES →

Q3  Does the request carry an Art. 17(3) exception?
    (Legal obligation to retain, public interest, legal claim defence, …)
    ├── YES → Refuse with written reasons (Runbook C step 8). Close.
    └── NO →

Q4  Is the model a fine-tuned LLM on a cohort that includes the record?
    ├── YES →

    Q4a  Was the fine-tune under SISA/sharded regime?
         ├── YES → Path C (SISA unlearning on the relevant shard).
         └── NO →

    Q4b  Is the operational gate: TOFU-style FQ p-value ≥ target AND MU drop ≤ tolerance?
         ├── YES → Path D (NPO-family unlearning, [[exercise-04]]).
         └── NO  → Path E (retrain with record removed).

    ├── NO (model is not an LLM; non-fine-tuned or classical ML) →

    Q4c  Was training under SISA?
         ├── YES → Path C.
         └── NO →

    Q4d  Is the loss convex / near-convex AND parameter count ≲ 10^7
         AND no previous Basu–Pope–Feizi fragility result on this architecture?
         ├── YES → Path D (influence functions, [[exercise-03]]).
         └── NO  → Path E.

Q5  Can we batch this request with other pending erasure requests?
    (If using (D) or (E), batching reduces unit cost.)
    ├── YES → Hold request in a bounded queue up to the Art. 12(3) +2-month extension
              IF the extension notice was issued by day 30 (see [[exercise-01]] §2.5).
    └── NO  → Execute immediately.

Q6  Did the chosen path meet the operational gate post-execution?
    ├── YES → Write audit-trail entry ([[exercise-07]]). Send Art. 17 response.
    └── NO  → Escalate to Path E and retry. Log the method change.
```

The gate in Q4b/Q4d is where [[exercise-03]] / [[exercise-04]] /
[[exercise-05]] plug in operationally — the gate is measured, not
claimed. A method that passes the gate on a validation harness is
the method chosen; a method that only *approximates* passage
escalates one step.

### 2.3 The five leaf playbooks

#### A — Source-only delete

- **When:** Q1 = NO, or Q2 = NO.
- **Actions:** SQL delete on authoritative store; cascade to feature
  store, derivative caches, logs. Nothing happens to model weights.
- **Evidence:** DB row count 0 for subject key across the nine
  surfaces listed in [[exercise-01]] §2.2 (whichever applied).
- **SLA:** T_30.
- **Response to subject:** "Erasure applied at source. Record was
  not included in the current production model; it will not be
  reintroduced."

#### B — Inference filter

- **When:** Bridge before next retrain; serving-layer filter is
  feasible (generation output can be post-processed; RAG retrieval
  can exclude an index entry).
- **Actions:** Add the subject's identifier (or PII pattern) to the
  serving-time denylist. Add the forget-set entries to the
  retrieval index's exclusion list. Instrument the filter: hit
  rate must be measurable.
- **Evidence:** Filter commit SHA; red-team probe output showing
  the filter catches a known query; next-retrain ticket scheduled.
- **SLA:** T_30 (filter in place) + commitment to Path C/D/E at
  next retrain window.
- **Response to subject:** "Inference-time filter active;
  model-level removal scheduled by `<date>`."

The filter alone is **not** Art. 17 fulfilment. The explicit
commitment to a model-level action is what makes this path
defensible.

#### C — Exact unlearning (SISA)

- **When:** Q4a = YES or Q4c = YES.
- **Actions:** Run `unlearn(record_id)` per [[exercise-02]]. The one
  affected shard restarts from the slice `r* − 1` checkpoint; all
  other shards are untouched.
- **Evidence:** Shard manifest no longer contains the record;
  bit-identity test (vs. a from-scratch retrain minus the record)
  passes; Audit-certificate payload (method = SISA, shard, slice,
  hash of final checkpoint).
- **SLA:** T_30.
- **Response to subject:** "Model unlearning applied. The affected
  model partition no longer contains your data; the corresponding
  parameters are bit-identical to a model trained without your
  data."

#### D — Approximate unlearning

- **When:** Q4b = YES (LLM, NPO-family method passes gate) or
  Q4d = YES (classical ML, convex, influence-function method
  passes gate).
- **Actions:** Run [[exercise-04]] (NPO + retain term) for an LLM,
  or [[exercise-03]] (LiSSA inverse-HVP) for a classical model.
  Evaluate on the operational gate; Path E on gate failure.
- **Evidence:** TOFU-style report (FQ p-value, MU, MIA AUC) at the
  gate thresholds; hyperparameters, seed, method, step count.
- **SLA:** T_30 (standard) or T_90 if the extension notice was
  issued.
- **Response to subject:** "Model unlearning applied. Statistical
  evaluation indicates the model's behaviour on your data is
  indistinguishable from a model retrained without it, at
  `<confidence>`." The statement names the method and the metric.

#### E — Full retraining

- **When:** Q4b = NO, Q4d = NO, or any operational-gate failure.
- **Actions:** Schedule a full retrain with the record removed
  from `D`. If a batching queue is open (Q5), include the queued
  requests in the same run.
- **Evidence:** Training manifest (dataset hash minus the record),
  commit SHA, run id, final checkpoint hash, pre/post evaluation
  metrics.
- **SLA:** T_30 (standard) or T_90 if extended.
- **Response to subject:** "The model has been retrained without
  your data. The current production artefact has training-set
  hash `<sha>`, which excludes your record."

### 2.4 Rationale

- **Default Path E is wrong.** Full retraining is the strongest
  guarantee but is not always required. Many erasure requests
  target records the model never trained on (Path A), or records
  that are entering a near-term retrain cycle (deferred Path A +
  schedule). Applying Path E to every request is not "safe"; it
  burns engineering capacity and increases the chance of SLA
  breaches on the requests that genuinely need it.
- **Path B alone is not fulfilment.** An inference filter addresses
  the symptom (model surfaces the content) without touching the
  cause (the content is in the parameters). Art. 17 contemplates
  erasure, not output suppression. Path B is defensible as a bridge
  with a committed Path C/D/E date; alone, it is a refusal
  dressed as compliance.
- **SISA eats most of the cost.** If the training pipeline was
  designed with SISA from the start, 80 %+ of erasure requests
  resolve to Path C in `≤ 1/S` of a full-train cost. Retrofitting
  SISA onto an existing pipeline is expensive; building new
  pipelines on SISA is cheap. The decision to use SISA is an
  architectural decision made once, not a per-request decision.
- **Approximate unlearning needs a gate.** Path D is only
  defensible if the operational gate (Q4b/Q4d) passes. Reporting
  "we ran NPO" without TOFU-style evidence is reciting the method
  rather than demonstrating the outcome. [[exercise-05]] is the
  gate implementation.
- **Batching is not an SLA shortcut.** Batching erasure requests
  reduces per-unit cost, but only within the SLA clock. Any
  batching window that extends past `T_30` (or `T_90` with
  extension) is an SLA breach, not an optimisation.

### 2.5 Governance mapping (NIST AI RMF)

- **Govern.** The decision tree is published and reviewed with
  privacy counsel; changes go through the same review as the
  runbook set.
- **Map.** The tree names the nine surfaces from [[exercise-01]];
  every branch acts on at least one surface.
- **Measure.** Each path has an evidence block (§2.3) that feeds
  the audit trail ([[exercise-07]]). Gate thresholds (FQ p-value,
  MU drop, MIA AUC) are reviewed quarterly.
- **Manage.** Operational-gate failures trigger escalation to a
  stronger path; recurring failures trigger a method or pipeline
  revision.

### 2.6 Threat model

- **MITRE ATLAS `AML.T0024` (Exfiltration via ML Inference API).**
  Path B alone is bypassable by an attacker who finds a query that
  skirts the filter. Path C/D/E addresses the root cause.
- **ATLAS `AML.T0020` (Data from Local System).** Deleted records
  can persist in model checkpoints retained for provenance. Any
  retention of pre-unlearn checkpoints beyond the audit window is
  itself a surface — list it explicitly in the audit entry.
- **OWASP ML05 (Membership Inference).** The operational gate
  (MIA AUC near chance) is the primary check that the unlearning
  worked. A path that leaves MIA AUC > threshold is not a
  fulfilment.
- **OWASP ML04 (Data Poisoning).** Erasure requests themselves can
  be a poisoning vector — an attacker requests mass erasure to
  degrade utility. Identity verification and batching caps bound
  this risk.

## 3. Implementation / artifact shape

The committed artefact:

```
mod-110/exercise-06-erasure-vs-retraining-decision-tree/
├── SOLUTION.md                       # this file
├── decision-tree.svg                 # flowchart rendering of §2.2
├── playbooks/
│   ├── A-source-delete.md
│   ├── B-inference-filter.md
│   ├── C-sisa-unlearn.md             # cross-links to [[exercise-02]]
│   ├── D-approximate-unlearn.md      # cross-links to [[exercise-03]], [[exercise-04]], [[exercise-05]]
│   └── E-full-retrain.md
├── gates/
│   ├── gate-q4b-llm.md               # FQ, MU, MIA thresholds
│   └── gate-q4d-classical.md         # convexity, parameter cap, fragility list
└── threat-model.md                   # §2.6
```

The flowchart should be renderable from Mermaid or a committed
`.puml`, not just screenshot — the branches must be grep-able from
CI so the tree stays in sync with the runbook set.

## 4. Validation steps

1. **Branch coverage.** Walk each of the five paths through at
   least one concrete scenario; every leaf is reachable by
   observable-fact conditions, with no "and the operator decides"
   escape hatch.
2. **Gate defensibility.** Open gate-q4b-llm.md and gate-q4d-classical.md;
   each gate names a specific metric, a specific threshold, and a
   dataset on which the threshold was calibrated. A gate without a
   number is not a gate.
3. **SLA arithmetic.** For each path, the playbook names
   the deadline (`T_30` by default; `T_90` with notice). The tree's
   Q5 batching branch respects those deadlines.
4. **Refusal cover.** Art. 17(3) is an explicit Q3 branch, not
   buried in the per-leaf playbook. "Business reasons" is not a
   listed exception.
5. **Evidence completeness.** Each playbook's "Evidence" block
   matches [[exercise-07]] fields: method, hashes, hyperparameters,
   seed, gate scores. Any missing field is a gap in the audit
   trail.
6. **Cross-link health.** Every cross-link (`[[exercise-01]]`,
   `[[exercise-02]]`, `[[exercise-03]]`, `[[exercise-04]]`,
   `[[exercise-05]]`, `[[exercise-07]]`) resolves to a section in
   the referenced document; dead links are a fail.
7. **Threat model cover.** The threat-model doc names ≥ 1 MITRE
   ATLAS technique and ≥ 1 OWASP ML Top 10 category per path.
8. **Not-fulfilment case.** The tree explicitly handles Path B
   as "bridge only." A tree that treats inference filters as
   terminal is incorrect.

## 5. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| All five paths present with distinct cost/guarantee/audit profiles | 2 | §2.1 |
| Decision tree uses observable facts for every branch (no operator judgement) | 2 | §2.2 |
| Art. 17(3) refusal branch explicit | 1 | — |
| Path D gate is defined by a concrete numeric threshold | 1 | §2.4 |
| Path B is treated as a bridge, not fulfilment | 1 | §2.3 |
| Evidence blocks per playbook map to [[exercise-07]] fields | 1 | — |
| Threat model ties each path to ≥ 1 ATLAS technique | 1 | §2.6 |
| Governance mapping (NIST AI RMF) present | 1 | §2.5 |

Automatic failures:

- Routing every request to Path E ("retrain, it's safer").
- Routing every request to Path B ("filter, it's cheap").
- Omitting the Art. 17(3) refusal branch.
- Treating Path D's gate as "if method ran without error."
- Making SLA secondary to batching.

## 6. Common mistakes

- **Defaulting to Path E.** It is the strongest guarantee and the
  most expensive. Picking it unconditionally burns capacity and
  obscures the method the architecture *should* support (Path C).
- **Treating Path B as the answer.** Inference filters are useful
  as bridges but do not satisfy the "make the data forgotten" half
  of Art. 17. A model that memorised the data still knows it; the
  filter is a serving-time concealment.
- **Picking Path D without a gate.** "We ran unlearning" is not a
  claim. "We ran NPO; FQ p-value = 0.74, MU drop = 2 %, MIA AUC =
  0.52" is a claim. The gate is the difference between a method and
  a finding.
- **Forgetting the batching ceiling.** Batching erasure is fine
  inside the SLA; past the SLA it is a breach regardless of
  internal cost arguments.
- **Hiding the refusal branch.** Art. 17(3) is law. The refusal
  branch exists and should be used when applicable, not
  avoided. Refusing with written reasons is lawful; silently
  not-fulfilling is not.
- **Ignoring the pre-unlearn checkpoint retention.** Keeping the
  pre-unlearn model checkpoint for "provenance" extends the
  surface by one. The retention policy must either delete the
  checkpoint on audit-window close or treat it as an in-scope
  surface for the erasure.
- **Treating "unlearning" as a single technique.** The literature
  is a family; the tree routes between them based on the model
  class and the gate. A tree that offers "unlearning: yes / no" is
  not useful.
- **Omitting the retrain-only reference in Path D's gate.** The FQ
  metric requires it ([[exercise-05]]); the gate is uninterpretable
  without it.

## 7. References

- Regulation (EU) 2016/679, Articles 12, 17, 19, 21, 22.
  <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- ICO Guidance — Right to erasure.
  <https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/right-to-erasure/>
- Bourtoule, L. et al. *Machine Unlearning.* IEEE S&P 2021.
  <https://arxiv.org/abs/1912.03817>
- Nguyen, T. T. et al. *A Survey of Machine Unlearning.*
  <https://arxiv.org/abs/2209.02299>
- Maini, P. et al. *TOFU.* COLM 2024.
  <https://arxiv.org/abs/2401.06121>
- Guo, C. et al. *Certified Data Removal.* ICML 2020.
  <https://arxiv.org/abs/1911.03030>
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS. <https://atlas.mitre.org/>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
