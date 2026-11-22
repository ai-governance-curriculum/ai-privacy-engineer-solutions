# Exercise 01 — Per-Tenant Retrieval ACL Design — Reference Solution

## 1. Solution overview

This is a **design exercise**. The learner is standing up a Retrieval-Augmented
Generation (RAG) service that serves multiple tenants (customers, business
units, or security realms) from a shared vector index, and must specify how
retrieval access control is enforced so that **no completion is grounded on a
document the asking identity is not permitted to read**.

The forcing constraint is that in RAG, authorisation is applied to *retrieval*,
not to the LLM: the model is a sink that will faithfully reproduce whatever
documents it is grounded on. If the ACL check fires after retrieval (or only
at the UI), the privacy boundary has already been crossed — the chunks are in
the prompt, the embedding query revealed what the caller was looking for, and
any logs of that turn now contain cross-tenant content.

The assessable output is a short design memo with four parts:

1. **Tenant model** — one sentence defining what a "tenant" is in this system
   (customer org, workspace, security realm, or user identity) and whether the
   label is mutable.
2. **ACL architecture** — the chosen enforcement topology (physical isolation,
   logical partition, or row/metadata filter), with the rationale against the
   alternatives.
3. **End-to-end data-flow diagram** — ingest → embed → index → retrieve →
   rerank → prompt → log, with the ACL check annotated at every stage where
   tenant data crosses a trust boundary.
4. **Threat mapping** — at least three attack paths that the chosen design
   defeats *and* at least one it does not, mapped to OWASP ML Top 10 and MITRE
   ATLAS entries, with a NIST AI RMF function (Govern / Map / Measure / Manage)
   named for each control.

Hybrid designs (per-tenant index for the top-N customers, shared index with
metadata filter for the long tail) are acceptable and often correct; the memo
must declare the split explicitly and specify which tenants sit on which side.

## 2. Worked implementation

### 2.1 The three canonical isolation topologies

Every production RAG ACL design is some combination of these three. The memo
must pick one as the primary story and name the others as rejected
alternatives (or as the long-tail fallback in a hybrid design).

| Topology | What is isolated | Blast radius of a filter bug | Cost shape | When to pick |
|---|---|---|---|---|
| **Physical isolation** — one index (or one DB, one namespace) per tenant | Everything: index, embedding model if fine-tuned, cache, logs | Smallest — a query against tenant B's index cannot return tenant A's documents even if the filter is wrong | Linear in tenants; wasteful for many small tenants | Regulated tenants (health, finance, gov), sovereignty requirements, very-large tenants, or any tenant with a contractual "no shared infra" clause |
| **Logical partition** — one index, one namespace/collection per tenant inside it | Documents and metadata within the index engine's partition primitive (e.g., Pinecone namespace, Weaviate class, pgvector schema) | Depends on whether the vector engine *enforces* partition selection or just defaults to it | Near-free; most engines charge per vector, not per partition | Default for SaaS with hundreds-to-thousands of tenants of similar size |
| **Metadata filter** — one shared index, one `tenant_id` column, filter applied at query time | Only the rows the filter actually matches | Largest — a missing or wrong filter returns every tenant's documents | Cheapest; one index is one index | Internal / low-risk tenants, small corpora, or the long-tail side of a hybrid design |

The answer is almost never "metadata filter alone" for a multi-tenant customer-facing
product. The forcing properties are:

- **Vector engines re-rank across the whole index.** Without a pre-filter, an
  unfiltered ANN search returns the globally-nearest chunks regardless of tenant.
  Some engines apply filters pre-ANN, others post-ANN with no recall guarantee;
  the memo must name which, because post-ANN filters can starve correct results
  under load and encourage engineers to disable filtering "to fix recall."
- **One missing `AND tenant_id = ?` leaks the whole corpus.** Compared to a
  namespace miss (returns empty) or a physical miss (connection refused),
  metadata-filter failures are silent.
- **Embedding caches and reranker caches are new attack surfaces.** If
  embeddings are cached by document hash across tenants, two tenants that
  upload the same document share a cache key — fine — but a cache keyed only on
  query text leaks which queries each tenant ran.

### 2.2 Tenant identity — where it comes from

The ACL is only as trustworthy as the identity it reads. The memo must name
the identity source and the trust boundary at which it is bound to the request.

- **Request-scoped identity.** Caller identity is extracted from a signed
  token (OAuth / OIDC / mTLS) at the API gateway and attached to the request
  context as a `tenant_id` + `principal_id` tuple that is **not re-derivable
  from headers downstream**. Any service that reads `tenant_id` from a
  user-controllable field (header, query param, body) is wrong by construction.
- **Propagation.** `tenant_id` is forwarded with every internal RPC via a
  signed/attested context (e.g., SPIFFE/SPIRE SVID, service-mesh identity,
  or signed baggage). Agent tools in particular must receive the caller's
  identity, not the agent's service identity — see [[exercise-05]].
- **Mutability.** Tenant labels on documents are append-only or
  change-controlled; a bulk re-label of documents (e.g., tenant merge) goes
  through a workflow that records who did it, when, and how retrieval was
  paused during the transition.

### 2.3 Reference control points (the data-flow diagram, annotated)

```
                       [AuthN at gateway]
                         |  (identity bound to request-scoped ctx)
                         v
 ingestion ---> chunk ---> embed ---> index(tenant_id, doc_id, chunk)
   ^                                     ^
   |                                     |
   [tenant_id derived from upload path   [index engine partition selector
    or signed uploader identity,         pinned to ctx.tenant_id; metadata
    NOT from document content]           filter AND tenant_id = ctx.tenant_id]
                                          |
                                          v
 query ---> embed ---> retrieve ---> rerank ---> prompt assembly ---> LLM
             ^            ^             ^            ^
             |            |             |            |
             [embedding   [partition    [reranker    [prompt builder re-checks
              call does   pinned +       runs only   each chunk.tenant_id ==
              not log      metadata      on chunks   ctx.tenant_id BEFORE
              raw query    filter]       already     concatenating]
              with ctx                   filtered]
              that could
              leak tenant
              text]
                                                 |
                                                 v
                                        completion ---> response filter
                                                           |
                                                           v
                                              structured audit log
                                              (hashes of doc_ids,
                                               NOT raw chunks)
                                              -- see [[exercise-03]]
```

Every arrow that crosses a trust boundary has an ACL assertion on it. The
memo states which assertions are **mandatory** (fail-closed) vs. **defence in
depth** (fail-closed but redundant):

| # | Control | Mandatory? | Failure mode |
|---|---|---|---|
| C1 | Gateway AuthN produces a bound `tenant_id` | Mandatory | Reject request |
| C2 | Partition selector pinned to `ctx.tenant_id` at retrieval | Mandatory | Reject query |
| C3 | Metadata filter `AND tenant_id = ctx.tenant_id` re-applied *after* the partition, pre-ANN | Mandatory (even if C2 already partitions) | Reject query |
| C4 | Prompt builder re-asserts `chunk.tenant_id == ctx.tenant_id` on every retrieved chunk before concatenation | Mandatory (defence in depth) | Drop chunk, alert |
| C5 | Reranker and LLM client never see cross-tenant chunks in a single batch | Mandatory | Drop batch |
| C6 | Embedding cache is **keyed by (`tenant_id`, text_hash)** or is per-tenant | Mandatory | Flush, re-embed |
| C7 | Audit log stores `tenant_id`, `principal_id`, `doc_id_hash` — not raw chunk text | Mandatory | See [[exercise-03]] |
| C8 | Admin / cross-tenant "support" access path is a separate, logged, time-bound role | Mandatory | — |

C4 is the single highest-value belt-and-braces check. It is cheap, it is
local, and it is the only control that catches a filter regression without
needing an integration test to hit exactly the wrong tenant pair.

### 2.4 Worked example — a SaaS tenant model

The design memo should instantiate the above against a concrete example.
A workable reference shape:

- **Product:** a B2B SaaS where each customer org is a tenant; inside an org,
  there are workspaces (projects); inside a workspace, there are users with
  roles.
- **Primary isolation:** logical partition per *org* (namespace per tenant
  in the vector engine); physical isolation reserved for enterprise tier
  customers who contractually require it.
- **Secondary filter:** `AND org_id = ctx.org_id AND workspace_id IN
  ctx.allowed_workspace_ids` applied at retrieval and re-checked at prompt
  assembly.
- **Tenant identity:** OIDC → gateway → request ctx carrying `org_id`,
  `user_id`, and `workspace_ids`.
- **Admin override:** a documented "break-glass" role that must declare a
  ticket ID at login; its sessions are surfaced in the tenant's own audit
  feed (`support_access_events`) within 24h.
- **Shared resources deliberately kept non-tenant:** the base embedding
  model; the base reranker weights; the base LLM. Fine-tuned adapters, if
  any, are per-tenant (see [[mod-105]] on memorisation for the reason).

### 2.5 Threat mapping

| Attack path | Defeated by | OWASP ML Top 10 | MITRE ATLAS | NIST AI RMF |
|---|---|---|---|---|
| Caller spoofs `tenant_id` in a header | C1 (identity is not read from caller-controllable fields) | ML02 (Data Poisoning) is adjacent; mapping is really to the OWASP API Top 10's Broken Object-Level Authorization (BOLA) via the AI surface | AML.T0040 ML Model Inference API Access | Govern — identity binding, authN policy |
| Caller passes a crafted query hoping the ANN returns another tenant's chunks | C2 + C3 (pre-ANN filter + partition) | ML02-adjacent; primarily ML06 (AI Supply Chain) when cross-tenant ingest is uncontrolled | AML.T0024 Exfiltration via ML Inference API | Measure — retrieval test harness ([[exercise-02]]) |
| Document ingest smuggles another tenant's `tenant_id` into metadata | Ingestion sets `tenant_id` from the uploader identity, not from document content | ML06 AI Supply Chain | AML.T0010 ML Supply Chain Compromise | Manage — ingest provenance |
| Embedding or reranker cache leaks a tenant's queries to another tenant | C6 (per-tenant cache key) | ML09 (Output Integrity Attack) adjacent | AML.T0024 | Measure — cache-leakage test |
| Shared fine-tuned LLM memorises one tenant's data and surfaces it to another | NOT fully defeated — ACL alone does not stop memorisation. Requires per-tenant adapters or DP-trained adapters | ML01 (Input Manipulation), ML02 (Data Poisoning) | AML.T0043 Craft Adversarial Data; memorisation extraction mapping in ATLAS | Manage — see [[mod-105]] and [[mod-106]] |
| Logs written in plain text retain cross-tenant chunks (completion logs) | C7 (structured hashes) + [[exercise-03]] + [[exercise-04]] | ML06-adjacent | AML.T0024 | Measure — log-redaction audit |
| Operator with DB access reads across tenants | Not defeated by retrieval ACL; requires DB-level encryption/isolation and audit — out of scope of this memo, but called out | — | AML.T0020 Data from Local System | Govern — privileged-access policy |

The "NOT defeated" rows are the honesty check. A memo that claims the
retrieval ACL defeats memorisation, confused-deputy at the LLM, or operator
exfiltration is overclaiming and should be rejected.

### 2.6 Rejected alternatives

- **"Metadata filter on a shared index, nothing else."** Rejected for
  customer-facing multi-tenant SaaS: a single missing filter in one of N code
  paths leaks the whole corpus, silently. Acceptable only for low-risk
  internal tenants or the long-tail side of a hybrid design, and only with
  a mandatory pre-ANN filter at the engine level.
- **"Encrypt each tenant's vectors with a per-tenant key."** Rejected as the
  *primary* control: current ANN engines operate on cleartext vectors.
  Encryption at rest is a separate control and does not substitute for an
  ACL on retrieval.
- **"Trust the LLM to refuse if it sees a mixed-tenant prompt."** Rejected
  unconditionally. Prompt-level refusal is not an access control; the
  privacy violation happened when the chunk entered the prompt.
- **"One index per user."** Rejected for most SaaS: too many indices, poor
  recall on small corpora, and cross-user sharing within an org becomes
  painful. Reserve for very few, very sensitive users.

## 3. Validation steps

A reviewer walks the memo through the following checks:

1. **Tenant definition test.** The memo defines "tenant" in one sentence
   and says whether the label is mutable. Fuzzy tenant definitions are
   the root cause of most cross-tenant leakage in practice.
2. **Identity-source test.** The memo names where `tenant_id` comes from
   on each request and asserts that no downstream service reads it from a
   caller-controllable field.
3. **Mandatory-control checklist.** Every control in §2.3 is either
   present in the design or explicitly waived with a one-line
   justification. A silent omission scores zero on that row.
4. **Pre-ANN filter audit.** The memo names the vector engine and states
   whether its metadata filter runs pre-ANN or post-ANN. Post-ANN alone is
   a finding; the memo must add a second pre-filter layer or move to
   namespace isolation.
5. **Belt-and-braces re-check.** The prompt builder re-asserts
   `chunk.tenant_id == ctx.tenant_id` before concatenation. Memos without
   C4 are rejected regardless of how strong the earlier filters look.
6. **Threat-map coverage.** The memo includes at least three defeated
   paths and at least one explicitly NOT-defeated path. Over-claiming is a
   finding.
7. **Hybrid honesty.** Any hybrid design (physical for some tenants,
   logical for the rest) names which tenants are on which side and the
   criterion used to decide.
8. **Cross-reference check.** The memo points to [[exercise-02]] for the
   negative-test harness, [[exercise-03]]/[[exercise-04]] for logging, and
   [[exercise-05]] for agent-tool identity propagation. Isolated ACL
   designs without these hooks are incomplete.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Tenant defined in one sentence; mutability stated | 1 | — |
| Primary isolation topology chosen with rationale against the other two | 2 | Must name pre-ANN vs. post-ANN filter semantics of the chosen engine |
| Identity-source control (C1) specified; no header-trust anywhere | 1 | — |
| Pre-ANN filter + partition pinning (C2 + C3) specified | 1 | — |
| Prompt-builder re-check (C4) present | 1 | Defence in depth — most missed control |
| Cache-isolation control (C6) specified | 1 | Per-tenant key or per-tenant cache |
| Admin / break-glass path named with audit requirement | 1 | — |
| Threat map covers ≥ 3 defeated paths + ≥ 1 NOT-defeated path, with OWASP ML Top 10 + ATLAS entries | 1 | Over-claiming is a finding |
| At least one NIST AI RMF function named per control | 1 | Govern/Map/Measure/Manage |

Automatic failures:

- `tenant_id` sourced from a caller-controllable header anywhere in the
  design.
- Metadata filter as the sole isolation mechanism on a customer-facing
  multi-tenant product without a pre-ANN filter guarantee.
- Missing C4 prompt-builder re-check.
- Claiming the ACL defeats memorisation-extraction attacks.

## 5. Common mistakes

- **Enforcing authorisation after retrieval.** By the time the ACL check
  runs on the retrieved list, the ANN scan already saw cross-tenant
  vectors and the query embedding may already sit in a cross-tenant
  cache. Enforcement must be pre-ANN.
- **Trusting the index engine's partition default.** Many engines route to
  the "default" namespace when the caller omits it, which turns a bug
  (`ctx.tenant_id` is null) into a *correct-looking* query against the
  wrong partition. Pin the partition selector to `ctx.tenant_id` and
  fail-closed if it is missing.
- **Keying embedding caches on text alone.** This leaks query content
  across tenants and converts a cache into a side channel. Key on
  (`tenant_id`, `text_hash`) or run per-tenant caches.
- **Forgetting the reranker.** Rerankers often run on a flat list of
  candidates. If upstream retrieval accidentally mixes tenants, the
  reranker will happily score them together and the wrongness becomes
  invisible.
- **Treating "the LLM wouldn't do that" as a control.** It is not a
  control; it is a hope. The LLM will faithfully reproduce whatever
  chunks it is grounded on. The ACL has to prevent the chunks from
  reaching the prompt.
- **Collapsing user-level and tenant-level ACLs.** A tenant ACL stops
  cross-customer leakage; a workspace / document ACL stops intra-tenant
  leakage. They are separate controls; the memo must name both if the
  product has both.
- **Not writing down the admin bypass.** Every real system has a
  support-access path. Omitting it from the memo does not remove it; it
  just removes the audit requirement, which is worse.

## 6. References

- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — Adversarial Threat Landscape for AI Systems.
  <https://atlas.mitre.org/> (techniques `AML.T0024 Exfiltration via ML
  Inference API`, `AML.T0010 ML Supply Chain Compromise`,
  `AML.T0040 ML Model Inference API Access`, `AML.T0020 Data from Local
  System`, `AML.T0043 Craft Adversarial Data`).
- NIST AI Risk Management Framework 1.0.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- Related solutions in this track: [[exercise-02]] cross-tenant negative
  tests; [[exercise-03]] prompt/completion logging policy;
  [[exercise-04]] redact-before-log pipeline; [[exercise-05]] agent-tool
  identity propagation; [[exercise-06]] RAG system card privacy slice.
