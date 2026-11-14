# Exercise 06 — Role Scope and Deferral Contract — Reference Solution

## 1. Solution overview

This is a **design exercise** on the privacy-engineer's position in the
governance ladder. A privacy engineer sits between the DPO / privacy
counsel, the security team, the ML / platform teams, and the product /
business owners. Each of those neighbours overlaps with the engineer on
some decisions. Without an explicit role contract, the engineer either
over-reaches (makes legal calls they are not qualified to make) or
under-delivers (defers so broadly that nothing ships).

The assessable output is a **Role Scope and Deferral Contract** — a short,
written artifact that a privacy engineer can hand to a new manager on day
one. It has three parts:

1. **In-scope decisions** — things the privacy engineer owns and signs.
2. **Co-owned decisions** — things the privacy engineer co-signs with a
   named partner (DPO, security lead, ML lead, product owner, counsel) and
   the explicit tie-breaker when they disagree.
3. **Deferred decisions** — things the privacy engineer never decides, who
   they defer to, and the exact trigger phrases that force the deferral.

The output includes a worked example contract and a decision-flow diagram.

## 2. Worked answer

### 2.1 The governance ladder, in one diagram

Top (strategic) to bottom (tactical):

1. **Board / Executive** — accountability for the organisation's privacy
   posture; approves risk appetite.
2. **DPO / Chief Privacy Officer / General Counsel (privacy)** — legal
   interpretation; supervisory-authority liaison; final arbiter on
   lawful basis, cross-border transfers, breach-notification decisions,
   and regulator correspondence.
3. **Privacy Program / Privacy Office** — standards, policies,
   training, DPIA templates, vendor diligence process.
4. **Privacy Engineering** — the engineer's seat. Translates the above
   into pipeline configs, tests, logs, DSR plumbing, and review gates.
   Owns execution of controls and the evidence that they work.
5. **Security Engineering** — identity, access, crypto, incident
   response. Privacy engineering consumes these.
6. **ML / Platform / Data Engineering** — builds and runs the pipelines
   that privacy engineering gates.
7. **Product / Business Owners** — decides product intent and user
   outcomes within the envelope the ladder imposes.

The privacy engineer's authority runs *down* to execution and *up* to
proposed designs; it does not run *up* into legal interpretation or
*sideways* into incident commander authority on a security event.

### 2.2 In-scope decisions (the engineer owns and signs)

These are decisions the privacy engineer makes without asking permission,
documents in a visible place, and takes the heat for if they turn out
wrong.

- **Classification tag** on a dataset / field: `pii`, `sensitive`, `confidential`,
  `public`. The engineer applies the organisation's taxonomy; disputes go to
  the privacy office.
- **Technical controls** implementing a named obligation: redaction rules,
  pseudonymisation configs, differential-privacy parameters (within a
  policy-set range), consent-state join conditions in a training pipeline,
  retention / TTL schedules consistent with a retention policy.
- **DSR plumbing** — which stores a subject identifier resolves to, the
  fan-out order, the retry policy, and the evidence receipt format.
- **Privacy review gate** — reviewing design docs for completeness and
  routing them to the right co-signers. The engineer may block a launch for
  insufficient evidence but does not reinterpret the law.
- **Evidence and audit artifacts** — what the audit log captures and how
  long it is retained for privacy-event reconstruction.
- **Pipeline-level fairness and bias instrumentation** — which subgroup
  metrics are produced, how they are surfaced, and the regression threshold.

### 2.3 Co-owned decisions (and the tie-breaker)

Each row has: the decision, the engineer's role, the co-owner's role, and
the explicit tie-breaker.

| Decision | Engineer's role | Co-owner | Tie-breaker |
|---|---|---|---|
| Lawful basis for a new processing activity | Proposes the basis based on the design; builds the enforcement | DPO / counsel | DPO / counsel |
| Necessity / proportionality of a feature that uses personal data | Proposes minimised feature set | Product owner | Privacy office (with DPO escalation) |
| Scope of a DPIA | Drafts the technical section | DPO / privacy office | DPO |
| Whether a given incident is a notifiable breach | Supplies the technical scope: what data, to whom, for how long | Security IR lead + DPO | DPO + counsel |
| Transfer mechanism for data that leaves a jurisdiction | Implements the mechanism (region pinning, SCCs flag in the DPA) | Counsel | Counsel |
| Deployment of a high-risk AI system under the EU AI Act | Owns Article 10 artifacts | ML lead + product owner | Privacy office |
| Model unlearning vs retraining response to an erasure request | Scopes the technical response and cost | ML lead | DPO on the sufficiency question |
| Threshold for bias metrics in a credit-scoring or hiring system | Proposes the threshold with evidence | Product owner + fairness lead | Privacy office; legal if protected class involved |
| Vendor's privacy adequacy | Reviews technical controls | Procurement + security | Procurement with DPO sign-off on legal adequacy |
| What "meaningful information about the logic" means for a given ADM system | Drafts the engineer-grade explanation | Product + counsel | Counsel (and model-card owner) |

### 2.4 Deferred decisions (never the engineer's call)

These are decisions where the engineer defers on sight. The right response
to any of these is "I need DPO / counsel / security IR / the board to make
the call; here is the technical input." Each has a trigger phrase or
pattern.

- **Regulator correspondence.** Any direct communication with a
  supervisory authority (national DPA, FTC, CPPA, CMA, ICO, etc.). Defer
  to DPO / counsel.
- **Legal interpretation of a statute.** "Is this GDPR Art. 6(1)(f)
  legitimate interest?" — defer to counsel.
- **Breach-notification decision.** Whether and when to notify a
  regulator or a subject under GDPR Art. 33/34, HIPAA 45 CFR 164.410, FTC
  Safeguards 16 CFR 314.5, or state laws — defer to DPO + counsel. The
  engineer supplies the scope.
- **Lawsuit or investigation hold.** Defer to legal. Follow the hold
  without further argument.
- **Disclosures to law enforcement.** Even if the request looks routine,
  defer to counsel. Preserve, do not delete; respond only after counsel
  signs.
- **Workforce or employee data access requests** that are not routine
  subject-access responses — defer to HR + counsel.
- **Decision to discontinue a feature that affects users** on privacy
  grounds alone — the engineer can block launch, but discontinuing a
  shipped feature is a product decision co-owned with product + counsel.
- **Settlement of a privacy dispute** — defer to counsel.
- **Approval of a new processor / sub-processor** — defer to
  procurement + DPO. The engineer provides technical findings.
- **Any request framed as "can we just use this data for X?"** — defer
  to the privacy office for a lawful-basis / DPIA call. The engineer may
  implement under the office's sign-off, but does not grant the
  permission.

### 2.5 Worked implementation — example contract

Scenario: a mid-size SaaS company has created the role "Senior Privacy
Engineer" reporting into the CISO, dotted-line to the DPO. The engineer
has no direct reports. The company has customers in the EU, California,
and New York; HIPAA does not apply; FERPA does not apply.

```text
Role Scope and Deferral Contract — Senior Privacy Engineer
Version: 1.0
Effective date: <insert>
Owner: Senior Privacy Engineer
Reviewers: CISO (reporting line), DPO (dotted line), VP Product (consumer)

A. In-scope decisions I own
1. Classification tags applied to datasets / fields using the published
   taxonomy.
2. Implementation of controls for GDPR Art. 5, 25, 32, and the
   corresponding CCPA / CPRA controls (deletion, access, SPI limit, GPC
   honour).
3. DSR fan-out technical design, SLA, and receipt format.
4. Privacy review-gate operation and sign-off completeness check.
5. Technical content of the DPIA (Section C), including residual-risk
   identification.
6. Evidence capture and audit-log retention for privacy events.

B. Co-owned decisions and the tie-breakers
B1. Lawful basis — I propose, DPO decides.
B2. Necessity / proportionality of a new feature using PI — I propose a
    minimised design, Product decides product-fit, DPO breaks ties on
    legal sufficiency.
B3. DPIA scope — DPO decides scope; I draft the technical analysis.
B4. Breach notifiability — Security IR (commander) + DPO decide.
B5. International-transfer mechanism — DPO / counsel decide; I implement.
B6. Vendor privacy adequacy — I supply the technical review; Procurement
    + DPO decide.
B7. ADM "meaningful information" copy — I draft engineer-grade text;
    Counsel + Product sign.
B8. High-risk AI system classification under the EU AI Act — DPO +
    Product decide; I own Article 10 artifacts.

C. Deferred decisions I never make
C1. Regulator correspondence — deferred to DPO.
C2. Statutory interpretation — deferred to counsel.
C3. Breach notification timing / content — deferred to DPO + counsel.
C4. Legal / investigation hold — deferred to counsel.
C5. Law-enforcement disclosures — deferred to counsel.
C6. Settlement of a privacy dispute — deferred to counsel.
C7. Employee data disclosures beyond routine SAR — deferred to HR +
    counsel.

D. Working protocols
D1. Any request of the form "can we just use this data for X?" routes to
    the privacy office before I implement anything.
D2. If I believe a design is unlawful, I escalate to DPO within 24 hours
    and block the launch in parallel.
D3. If I believe a design is lawful but risky, I file a residual-risk
    note and continue.
D4. I write down every decision I make and the reasons, in the privacy
    review channel; silent decisions do not count.

E. Review cadence
E1. This contract is reviewed semi-annually or when the organisation adds
    a new jurisdiction, data category, or AI use-case category.
```

### 2.6 Decision-flow — "someone is asking me to greenlight a new ML
feature"

```text
        ┌──────────────────────────────────────┐
        │ Request: "approve this feature"      │
        └────────────────┬─────────────────────┘
                         │
                         ▼
           Is this a legal interpretation?
              yes ──► defer to counsel / DPO
                         │ no
                         ▼
           Is a new personal-data category involved?
              yes ──► DPIA + DPO sign-off required
                         │ no
                         ▼
           Is a new processing purpose involved?
              yes ──► lawful-basis review with DPO
                         │ no
                         ▼
           Are existing technical controls sufficient?
              no  ──► propose controls, co-own with ML lead
                         │ yes
                         ▼
           Document decision, sign off, publish in review channel.
```

## 3. Validation steps

1. **"Who gets fired" test.** For each In-scope item, if the decision
   turns out wrong, the privacy engineer is the one named. If anyone else
   is named, the item does not belong in Section A.
2. **"Who signs regulator mail" test.** No item in Sections A or B should
   put the engineer's name on a regulator-facing letter. If one does,
   move it to C.
3. **Named tie-breaker per co-owned item.** Every Section B item has
   exactly one tie-breaker role, not a committee.
4. **Trigger-phrase test.** Every Section C item has a recognisable trigger
   phrase the engineer can match in real time.
5. **Semi-annual review set.** The contract has a review cadence and a
   trigger list for mid-cycle reviews.
6. **Cross-reference to policy.** Each Section A obligation should cite
   the organisation's underlying policy (taxonomy, retention policy, DSR
   SLA policy). Otherwise the engineer is inventing the standard, which
   is a Section B move at best.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Three sections clearly separated: In-scope, Co-owned, Deferred | 2 | A blurred structure fails |
| Each Section A item has a visible authorisation path (named policy or taxonomy) | 1 | Inventing standards fails |
| Each Section B item has exactly one named tie-breaker | 2 | "Depends" fails |
| Section C includes breach-notification, regulator correspondence, and legal interpretation | 2 | Omitting any fails that row |
| Contract names a review cadence and a change trigger list | 1 | Static contracts rot |
| At least one worked escalation example that shows A/B/C decisions in sequence | 1 | Flat lists without flow fail |
| Contract is one page or less and could be handed to a new manager | 1 | Unreadable contracts fail |

Automatic failures:
- Any item in Section A that is actually a legal interpretation.
- "DPO" used as a role the engineer plays. The engineer is not the DPO.
- A Section B item with a committee tie-breaker.
- Section C missing breach-notification deferral.

## 5. Common mistakes

- **Treating the engineer as the DPO.** The engineer executes; the DPO
  decides on lawful basis, regulator engagement, and most GDPR
  interpretations.
- **Treating the engineer as the incident commander.** The incident
  commander is security IR; the engineer supplies data-scope evidence.
- **Vague ownership** — "I own privacy" fails. Own specific decisions
  with specific artifacts.
- **No tie-breaker.** "We'll figure it out" is not a tie-breaker. Pick a
  role and name it.
- **Over-delegation.** Deferring every decision to counsel produces a
  team that cannot ship. Section A must be substantive.
- **Under-delegation.** Signing the lawful basis call is not substantive —
  it is impersonating counsel and will not survive audit.
- **Silent decisions.** A decision made on Slack in a DM without a
  written record does not count. The contract should require writing.
- **Static contract.** New jurisdictions, new data categories, and new AI
  use-case categories all change the role. Build the review cadence in.

## 6. References

- Regulation (EU) 2016/679 (General Data Protection Regulation),
  specifically Arts. 37–39 on the Data Protection Officer role.
  <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- NIST Privacy Framework v1.0 — Govern Function, which separates
  organisational privacy values and policies from execution.
  <https://www.nist.gov/privacy-framework>
- NIST AI Risk Management Framework (AI RMF 1.0) — Govern function and
  the role of organisational accountability.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- ISO/IEC 27701:2019 — leadership and roles clauses (5.2, 5.3; 7.2.1).
  <https://www.iso.org/standard/71670.html>
