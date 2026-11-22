# Exercise 05 — Agent-Tool Data Scope and Identity Propagation — Reference Solution

## 1. Solution overview

This is a **design exercise** with a working-example artifact. The
learner must specify how an LLM agent, when it calls a tool
(function / API / retriever), carries the *calling user's identity*
and the *data scope they are entitled to* — not the agent's own
service identity.

The forcing properties:

- **The agent is a confused deputy by default.** The agent process
  runs as a service principal with broad access to internal APIs,
  databases, and the corpus. If a tool call authenticates as the
  agent, it bypasses per-user ACL. This is the single highest-impact
  misconfiguration in production agent systems.
- **Model outputs are untrusted input to tools.** Anything the LLM
  decides to put into a tool argument is attacker-influenced via
  prompt injection or jailbreak. Tool authorisation cannot trust the
  argument to declare the right scope ("delete_user(user_id=…)" is
  not a scope statement; it is a request).
- **Capability must be bounded at the tool boundary.** The tool is
  the enforcement point. The agent cannot be the enforcement point
  because the model is probabilistic.

The assessable output:

1. **Identity-propagation design** — how a signed, bound user
   identity travels from the public gateway through the agent, into
   the tool, with no component downstream of the gateway trusting
   caller-controlled identity assertions.
2. **Capability model** — the set of tools, each with the data scope
   it can touch, the identities allowed to invoke it, and the
   per-invocation constraints (idempotency, dry-run, step-up auth).
3. **Tool-contract template** — a schema every tool must implement
   (input/output, identity consumption, error semantics, logging).
4. **Threat mapping** — three concrete attack scenarios (confused
   deputy, injection-driven scope escalation, cross-tenant tool
   argument) with the specific control that defeats each, mapped to
   OWASP ML Top 10, MITRE ATLAS, and NIST AI RMF.
5. **One worked walk-through** — a representative multi-step agent
   turn traced end-to-end with the identity and scope at each hop.

## 2. Worked implementation

### 2.1 Identity propagation — the end-to-end picture

```
 user -> gateway (OIDC / mTLS)
           |  mint request-scoped principal token:
           |    { sub, tenant_id, scopes[], exp(short), jti, aud=agent }
           v
         agent service
           |  validates principal token;
           |  attaches it to the agent's internal context;
           |  CANNOT mint a new principal token, only forward it;
           |  tool dispatcher exchanges principal token for a
           |  tool-specific capability token at call time:
           |    { sub, tenant_id, scopes∩tool_requires, tool_id,
           |      request_id, exp(≤ 60s), jti, aud=tool }
           v
         tool
           |  validates capability token;
           |  REJECTS any argument that implies a scope not present
           |  in the token;
           |  enforces data-scope at its own storage layer
           |  (NOT by trusting argument values).
```

Three key properties:

- **Request-scoped**, not session-scoped. The principal token is
  minted per inbound request so that revocation is immediate and
  blast radius is one turn.
- **Audience-locked.** The principal token's `aud` is the agent and
  the capability token's `aud` is the specific tool. A stolen
  principal token cannot be replayed into a tool directly; a stolen
  capability token cannot be replayed into another tool.
- **Short expiry.** The capability token has a tight expiry (≤ 60s
  in the reference) so a leaked token is useless almost immediately.

The agent service **cannot** mint principal tokens — the gateway
owns that capability. If the agent could mint, it could escalate at
will.

### 2.2 Capability model

Every tool is registered in a capability registry with four
declarations:

```yaml
# tools/registry.yaml
- id: order_lookup
  description: "Look up an order by ID."
  requires_scopes:
    - "order:read"
  data_scope: "per-user: results filtered to {sub} unless admin scope present"
  constraints:
    idempotent: true
    step_up_auth: false
  allowed_audiences: ["agent"]  # only the agent dispatcher may request

- id: order_refund
  description: "Issue a refund on an order."
  requires_scopes:
    - "order:write"
    - "payments:refund"
  data_scope: "per-user: limited to orders where order.user_id == {sub}"
  constraints:
    idempotent: true               # enforced via request_id
    step_up_auth: true             # require fresh user AMR within 5m
    dry_run_first: true            # agent must call with preview=true
    max_amount: "<= order.total"
  allowed_audiences: ["agent"]

- id: kb_search
  description: "Semantic search over the tenant knowledge base."
  requires_scopes:
    - "kb:read"
  data_scope: "per-tenant: partition pinned to {tenant_id}, see exercise-01"
  constraints:
    idempotent: true
    step_up_auth: false
  allowed_audiences: ["agent"]
```

Three ideas are encoded here:

- **Scopes are minimal.** The capability token issued to the tool
  contains only the intersection of the principal's scopes and the
  tool's `requires_scopes`. If the user did not consent to
  `payments:refund`, the agent cannot call `order_refund` on their
  behalf.
- **Data scope is a property of the tool, not the argument.**
  "Return orders where `order.user_id == {sub}`" is enforced by the
  tool's storage layer using the token's `sub`. The tool ignores any
  `user_id` the agent passed as an argument, or treats a mismatched
  `user_id` as a request to a scope the token does not grant and
  returns 403.
- **High-impact tools require step-up auth, dry-run, and
  idempotency.** The agent must call `order_refund` with
  `preview=true` first and show the user. The user confirms with an
  AMR within the 5-minute window. Idempotency keys ensure a repeated
  LLM call does not refund twice.

### 2.3 Tool-contract template

Every tool implements the same contract. This is what the
capability registry maps onto.

```python
# tools/contract.py
from __future__ import annotations

from dataclasses import dataclass
from typing import Any, Protocol


@dataclass(frozen=True)
class CapabilityToken:
    sub: str              # user subject (opaque ID)
    tenant_id: str
    scopes: frozenset[str]
    tool_id: str
    request_id: str
    exp: int              # unix seconds
    jti: str
    aud: str              # must equal the tool id


class Tool(Protocol):
    id: str
    requires_scopes: frozenset[str]

    def invoke(self, token: CapabilityToken, args: dict[str, Any]) -> dict:
        """Validate token, check scope subset, then enforce data_scope
        at the storage layer. Return a dict; raise AuthorisationError or
        ValidationError on failure. MUST NOT trust caller-provided
        user_id / tenant_id values as scope assertions."""
        ...
```

Three mandatory behaviours at `invoke` time:

1. Verify the token signature, audience (`aud == self.id`), expiry,
   and `jti` non-replay.
2. Verify `requires_scopes ⊆ token.scopes`.
3. Enforce `data_scope` at the storage query: every SQL / API call
   constrains by `token.sub` and `token.tenant_id`, regardless of
   what `args` claims.

Tools that return data must also:

- Log to `agent.tool_call` and `agent.tool_result` via the shipper
  in [[exercise-04]], with arguments redacted and `token.sub`
  hashed.
- Expose a `dry_run=True` path if the tool is non-idempotent or
  irreversible.

### 2.4 Worked walk-through — "refund my last order"

```
user: "please refund my last order"
  |
  v
gateway: authenticates user; mints principal token
  principal = { sub: u_123, tenant_id: t_alpha,
                scopes: [order:read, order:write, payments:refund],
                aud: agent, exp: now+5m, jti: j1 }
  |
  v
agent service (LLM loop):
  step 1: calls order_lookup(user='me') via dispatcher
    dispatcher mints cap1 = { sub: u_123, tenant_id: t_alpha,
                              scopes: [order:read],
                              tool_id: order_lookup, request_id: r_42,
                              aud: order_lookup, exp: now+60s, jti: c1 }
    tool order_lookup:
      - verifies cap1
      - queries orders WHERE user_id = :sub AND tenant_id = :tenant
      - returns [{order_id: o_777, total: 54.90, ...}]
      - NOTE: it ignored args['user']='me' as a scope statement

  step 2: LLM decides to call order_refund(order_id=o_777, preview=true)
    dispatcher mints cap2 = { sub: u_123, tenant_id: t_alpha,
                              scopes: [order:write, payments:refund],
                              tool_id: order_refund, request_id: r_42,
                              aud: order_refund, exp: now+60s, jti: c2 }
    tool order_refund (preview):
      - verifies cap2
      - verifies order_id belongs to u_123 at the storage layer
      - returns preview {amount: 54.90, reason_required: true}

  step 3: agent surfaces preview + asks user to confirm
    user confirms with step-up AMR; gateway mints principal' with
      "payments:refund" AMR claim and fresh exp

  step 4: agent calls order_refund(..., preview=false, idempotency_key=r_42)
    dispatcher mints cap3 (same shape as cap2, new jti, aud=order_refund)
    tool order_refund:
      - validates step-up AMR claim fresh within 5m
      - uses idempotency_key to deduplicate
      - issues refund; returns {status: ok, txn_id: t_99}
  |
  v
agent surfaces final result to user; emits structured audit events via
redacting shipper. No raw tool_args reach the central log.
```

Two details worth calling out:

- The agent **never** saw a long-lived credential that could
  directly refund money. The high-impact capability is fenced by a
  fresh user AMR.
- The LLM can be prompt-injected into calling
  `order_refund(order_id=o_999)`; the tool's storage-side check
  (`o_999.user_id != u_123`) returns 403. The tool, not the LLM, is
  the enforcement point.

### 2.5 Threat mapping

| Threat | Control | OWASP ML Top 10 | MITRE ATLAS | NIST AI RMF |
|---|---|---|---|---|
| Confused deputy — agent acts with service identity, bypassing per-user ACL | Dispatcher mints a capability token from the inbound principal; tools reject tokens whose `aud` ≠ tool_id and whose `sub` is the agent's service identity | ML02 + API Top 10 BOLA-adjacent | AML.T0040 ML Model Inference API Access | Govern — least-privilege policy; Manage — authN/authZ architecture |
| Prompt-injection drives a tool call outside the user's scope | Capability token carries only `principal.scopes ∩ tool.requires_scopes`; tool checks subset. The LLM cannot invent a scope that was not granted | ML01 Input Manipulation | AML.T0043 Craft Adversarial Data | Measure — tool-call authorisation test suite |
| Prompt-injection causes cross-user data action (refund someone else's order) | Tool enforces `data_scope` at storage via `token.sub`, ignoring `args` | ML01 + ML02 | AML.T0024 Exfiltration via ML Inference API | Measure — negative test, [[exercise-02]] shape |
| Replay of a captured capability token | 60-second `exp` + `jti` non-replay + `aud` lock | — | AML.T0040 | Govern — token lifecycle |
| Agent calls high-impact tool without user intent | `step_up_auth: true` + `dry_run_first: true` + idempotency key | — | AML.T0024 | Govern — human-in-the-loop for irreversible actions |
| Tool argument carries PII into logs | Shipper from [[exercise-04]] redacts `args` per schema; `token.sub` is hashed | — | AML.T0020 | Manage — log hygiene |

Attack paths this design does **not** defeat:

- **The user genuinely intends to do the harmful thing.** The agent
  can only honour the user's scopes; if the user asks for something
  they are entitled to do, nothing here stops them.
- **A compromised tool backend.** If the tool's storage trusts the
  agent too much, the token system is bypassed. Tools must do their
  own tenant/user scoping independently — defence in depth, not a
  single choke point.
- **Model-memorised secrets.** If a tool previously returned
  secrets that then entered the LLM's context and got cached /
  trained into a derivative, this design does not help. See
  [[mod-105]].

## 3. Validation steps

Reviewer walks the design through:

1. **No agent service-identity inside tools.** Every tool rejects
   calls whose `token.sub` is the agent's service principal.
2. **Audience lock test.** A token with `aud=order_lookup` cannot
   call `order_refund` and vice versa; both reject.
3. **Expiry test.** A capability token older than its `exp` is
   rejected; replaying a captured token after 60 s fails.
4. **Scope-intersection test.** A principal without
   `payments:refund` cannot drive a successful refund call — not at
   the gateway, not at the dispatcher, not at the tool.
5. **Data-scope-at-storage test.** The tool rejects an argument
   that implies a scope beyond the token's `sub` / `tenant_id`,
   even if the agent's prompt claims the caller is "admin."
6. **Step-up auth test.** For tools flagged `step_up_auth: true`,
   lacking a fresh AMR causes a 403 even with correct scopes.
7. **Dry-run test.** For irreversible tools, a `preview=false` call
   without a preceding `preview=true` is rejected.
8. **Audit test.** A full agent turn produces events that, through
   the shipper of [[exercise-04]], contain no raw PII, no raw
   principal IDs, and no cross-tenant content.
9. **Cross-reference check.** Design references [[exercise-01]]
   (tenant ACL), [[exercise-02]] (negative tests), [[exercise-03]]
   (logging), and [[exercise-04]] (redactor).

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Request-scoped principal token minted at the gateway; agent cannot mint | 2 | Most-missed control |
| Dispatcher exchanges principal → capability token per tool, audience-locked, short expiry | 2 | — |
| Capability registry declares `requires_scopes`, `data_scope`, and constraints per tool | 1 | — |
| Tools enforce data scope at storage, not from args | 2 | — |
| High-impact tools require step-up auth + dry-run + idempotency | 1 | — |
| Threat map covers ≥ 3 defeated paths + ≥ 1 explicitly not defeated | 1 | — |
| Worked walk-through traces identity at every hop | 1 | — |

Automatic failures:

- Agent calls tools with its own service identity.
- Tool trusts `user_id` / `tenant_id` from arguments as scope.
- Long-lived tool tokens (> 5 minutes) without audience locking.
- Irreversible tool exposed with no dry-run or step-up.
- No idempotency path on write tools.

## 5. Common mistakes

- **Using the agent's service account for tool calls.** The agent
  can read everything, so every per-user ACL downstream becomes
  decorative. This is the confused-deputy pattern, and it is
  overwhelmingly the most common failure mode.
- **Trusting tool arguments as scope statements.** `user_id="alice"`
  in an argument means the LLM *wants* alice; it does not grant the
  right. The storage-side check must come from the token.
- **One capability token, every tool.** A single token that covers
  every scope reintroduces the confused-deputy problem; audience
  binding per tool is cheap and removes it.
- **Long-expiry tokens for convenience.** A 30-minute token
  surviving a prompt-injection round is enough to be exploited.
  Keep capability tokens under a minute; keep principal tokens
  under 5 minutes.
- **No step-up for irreversible actions.** Model confidence is not
  user intent. Refunds, deletes, sends, and purchases all need a
  human signal that is not the LLM's own tool call.
- **Letting the agent write to shared state without idempotency.**
  Retries at any layer (model streaming, dispatcher, tool network)
  can double-fire writes. Require a client-supplied idempotency key
  from the dispatcher.
- **Forgetting the tool's own log hygiene.** Tool arguments and
  results carry user content. They must go through the redactor
  from [[exercise-04]]; a tool that logs `args` raw undoes the
  whole policy.
- **Treating "the model will refuse" as a control.** The LLM may
  refuse; it may not. The tool is the enforcement point.

## 6. References

- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS.
  <https://atlas.mitre.org/> (`AML.T0040 ML Model Inference API
  Access`, `AML.T0043 Craft Adversarial Data`, `AML.T0024
  Exfiltration via ML Inference API`, `AML.T0020 Data from Local
  System`).
- NIST AI Risk Management Framework 1.0 — Govern (least-privilege,
  accountability), Measure (controls testing), Manage (incident
  response for prompt-injection).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- Related solutions in this track: [[exercise-01]] per-tenant
  retrieval ACL; [[exercise-02]] negative-test harness;
  [[exercise-03]] logging policy; [[exercise-04]] redact-before-log
  pipeline; [[exercise-06]] RAG system card privacy slice; and
  [[mod-105]] on memorisation / extraction for the risks this design
  does not defeat.
