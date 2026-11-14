# Exercise 03 — Sector Trigger Map: HIPAA, FERPA, COPPA, GLBA — Reference Solution

## 1. Solution overview

This is a **design-and-interpretation exercise**. US sectoral privacy laws
(HIPAA, FERPA, COPPA, GLBA) are not "privacy laws" in the GDPR / CPRA sense —
each is a conditional regime that **attaches** only when a specific trigger
is met. The learner's job is to build a decision aid that tells an engineer,
with reference to statute and rule text, *whether* a given AI product falls
under each regime, and if so, *which* obligations it inherits.

The assessable output is a **trigger map** that for each of the four regimes:

1. Identifies the regulated entity type the trigger depends on.
2. Identifies the regulated data type the trigger depends on.
3. Lists the specific engineering duties that attach *when both entity and
   data triggers fire*.
4. Names the primary engineering artifact(s) per duty.
5. Shows at least one worked example — ideally a mixed example where more
   than one of the four regimes fires — and one counter-example where none
   fire despite the product "feeling" regulated.

## 2. Worked answer

### 2.1 The four regimes at a glance

| Regime | Statute / Rule | Trigger — regulated entity | Trigger — regulated data | Primary regulator |
|---|---|---|---|---|
| HIPAA Privacy, Security, Breach Notification Rules | HIPAA (Pub. L. 104-191); regulations at 45 CFR Parts 160 and 164 | **Covered entities** (health plans, health care providers that transmit certain transactions electronically, health care clearinghouses) and **business associates** acting on their behalf (45 CFR 160.103) | **Protected Health Information (PHI)** — individually identifiable health information held or transmitted by a covered entity or business associate (45 CFR 160.103) | HHS Office for Civil Rights |
| FERPA | Family Educational Rights and Privacy Act (20 U.S.C. § 1232g); regulations at 34 CFR Part 99 | **Educational agencies or institutions** that receive US Department of Education funds (34 CFR 99.1) | **Education records** — records maintained by, or on behalf of, an educational agency or institution that are directly related to a student (34 CFR 99.3) | US Dept of Education / Student Privacy Policy Office |
| COPPA | Children's Online Privacy Protection Act (15 U.S.C. §§ 6501–6506); FTC Rule at 16 CFR Part 312 | **Operators of websites or online services directed to children**, and operators with **actual knowledge** that they are collecting personal information from a child (16 CFR 312.2) | **Personal information** from a child under 13 (16 CFR 312.2) | FTC (and state AGs with concurrent authority) |
| GLBA Privacy Rule + Safeguards Rule | Gramm-Leach-Bliley Act (15 U.S.C. §§ 6801–6809); FTC Safeguards Rule at 16 CFR Part 314; CFPB Regulation P (Privacy) at 12 CFR Part 1016 (for most banks and consumer finance entities) | **Financial institutions** significantly engaged in financial activities (16 CFR 314.2; 12 CFR 1016.3) | **Non-public personal information (NPI)** about consumers (16 CFR 313.3 / 314.2; 12 CFR 1016.3) | FTC (non-bank financials), federal banking regulators, CFPB |

### 2.2 Trigger decisions, in order

For each AI product, work through this short decision tree **before** writing
code, not after:

1. **Health trigger.** Is my customer a HIPAA covered entity (health plan,
   provider that files electronic transactions, or clearinghouse)? If yes and
   my service processes PHI on their behalf, I am a **business associate**;
   HIPAA attaches to me by contract (BAA) and by rule. If no, HIPAA does not
   attach even if my data looks medical — e.g., a consumer wellness app not
   acting for a covered entity is generally outside HIPAA, which is counter-
   intuitive and important.
2. **Education trigger.** Is my customer an educational agency or institution
   that receives US Department of Education funds? If yes and I handle
   education records on their behalf, FERPA attaches through the school's
   duties and typically through the "school official" exception in 34 CFR
   99.31(a)(1)(i)(B). If my customer is a private tutoring company with no
   ED funds, FERPA does not attach.
3. **Child trigger.** Is my service (a) directed to children under 13, or
   (b) am I collecting personal information from users I have actual
   knowledge are under 13? If yes, COPPA attaches regardless of sector.
4. **Financial trigger.** Am I a financial institution, or am I a service
   provider to one? The FTC Safeguards Rule covers non-bank financials; the
   banking regulators and CFPB cover banks and many consumer financial
   companies via Regulation P and parallel safeguards rules.

Each "yes" layers on duties. A single AI product can be in **zero, one, two,
three, or all four** regimes simultaneously.

### 2.3 Engineering duty inheritance per regime

#### HIPAA (when it attaches)

Relevant Privacy, Security, and Breach Notification Rule duties for the
engineer:

- **Business Associate Agreement.** A signed BAA with the covered entity is
  a precondition; see 45 CFR 164.504(e). Engineering artifact: the BAA
  tracked in the vendor register and the processing boundary configured to
  refuse data flows for which no BAA is on file.
- **Minimum necessary.** 45 CFR 164.502(b). Engineering artifact: field-level
  access controls, request scoping on query APIs, redaction of fields not
  needed for the task.
- **Administrative, physical, and technical safeguards** (Security Rule,
  45 CFR 164.308 / 164.310 / 164.312). Engineering artifacts: access
  control, audit controls, integrity controls, person or entity
  authentication, transmission security. Encryption is an *addressable*
  specification under § 164.312(a)(2)(iv) and § 164.312(e)(2)(ii) — the
  engineer either implements it or documents a reasoned alternative.
- **Breach notification** (45 CFR Part 164, Subpart D). Business associates
  notify the covered entity "without unreasonable delay and in no case later
  than 60 calendar days" after discovery (§ 164.410). Engineering artifact:
  incident runbook with the BA-to-CE notification clock.
- **De-identification standards** (§ 164.514(b)): either the Expert
  Determination method or the Safe Harbor method (removal of 18 enumerated
  identifier categories). This matters for training data: a BA cannot use
  PHI "for its own purposes" (§ 164.504(e)(2)) without the covered entity's
  authorisation or a valid de-identification. The engineering artifact is a
  de-identification pipeline with reproducible configuration.
- **Patient rights.** Access (§ 164.524), amendment (§ 164.526), accounting
  of disclosures (§ 164.528). A business associate typically implements
  these through the covered entity's interface; the engineering artifact is
  the BA-side request-handling adapter.

#### FERPA (when it attaches)

- **Consent for disclosure.** 20 U.S.C. § 1232g(b); 34 CFR 99.30. Prior
  written consent of a parent or eligible student unless an exception
  applies.
- **School-official exception.** 34 CFR 99.31(a)(1)(i)(B). A contractor can
  receive education records without consent if it (i) performs an
  institutional service for which the school would otherwise use employees,
  (ii) is under the school's direct control with respect to use and
  maintenance, and (iii) uses the records only for the authorised purpose
  and redisclosures within § 99.33. Engineering artifact: contract clause,
  purpose-limited access configuration, redisclosure prohibition.
- **Directory information** carve-out (§ 99.37) with opt-out — what the
  school has designated as directory information can be disclosed without
  consent subject to opt-out; engineers should treat "directory information"
  as a per-tenant configuration, not a product-wide assumption.
- **Right to inspect and review** (§ 99.10), **right to seek amendment**
  (§ 99.20), and **log of disclosures** (§ 99.32). Engineering artifacts:
  per-student access portal driven by the school's identity source, request
  routing to the school for amendment decisions, disclosure log entry on
  every eligible disclosure.
- **AI-specific consideration.** Using education records to train a model is
  "use," not "disclosure"; the school-official exception's "direct control"
  requirement makes training-on-customer-data by default a common FERPA
  failure mode. Engineering artifact: training-corpus manifest that excludes
  student-identifiable records unless the school has authorised the use.

#### COPPA (when it attaches)

- **Direct notice and verifiable parental consent** before collecting
  personal information from children under 13 (16 CFR 312.4–312.5). Methods
  include signed consent forms, payment transactions, knowledge-based
  authentication, government ID matching, video chat, and newer FTC-
  approved methods. Engineering artifact: a consent capture component
  parameterised by method and a server-side record of the method and the
  timestamp.
- **Privacy notice** with specific contents (§ 312.4(d)).
- **Parental access, review, and deletion** (§ 312.6). Engineering artifacts
  parallel the DSR service, with parent-identity resolution.
- **Data minimisation** (§ 312.7) — may not condition a child's
  participation on the collection of more PI than is reasonably necessary.
- **Confidentiality, security, and integrity** (§ 312.8) — including
  deletion of collected information once no longer necessary.
- **Safe-harbor programs** (§ 312.11) are optional industry-self-regulatory
  routes that still require an underlying implementation.
- **AI-specific consideration.** Inference-time collection counts. If a
  model receives voice, image, or free-text from a child, it is collecting
  PI; training a model on child data falls squarely under § 312. Engineering
  artifact: a "child flag" on the session that routes requests through the
  consent-verified path or refuses them.

#### GLBA Privacy Rule + Safeguards Rule (when it attaches)

- **Initial and annual privacy notices** to customers (Reg P, 12 CFR
  1016.4–1016.5; FTC Privacy Rule, 16 CFR 313.4–313.5 where it still
  applies). Engineering artifact: notice content and delivery tested for
  each covered product.
- **Opt-out of sharing with non-affiliated third parties** for information
  outside the § 1016.14 / 16 CFR 313.14 "service provider / joint marketing"
  exception and § 1016.15 / 313.15 exceptions. Engineering artifact: opt-out
  flag honoured at the data-sharing boundary.
- **FTC Safeguards Rule** (16 CFR Part 314), as amended, requires a written
  information security program with specific elements including a qualified
  individual (§ 314.4(a)), a written risk assessment (§ 314.4(b)), specified
  safeguards (§ 314.4(c)) including access controls, inventory, encryption
  of customer information in transit over external networks and at rest
  (addressable with documented equivalent), MFA for individuals accessing
  any information system, change management, continuous monitoring or
  periodic testing, secure disposal, incident response plan, and a report
  to the board. Engineering artifacts: encryption configs, MFA enforcement,
  inventory tie-in to the data catalog, access-control IaC, DR and
  destruction procedures.
- **Notification of a notification event** (16 CFR 314.5) to the FTC "as
  soon as possible, and no later than 30 days after discovery" for a
  notification event involving the information of 500 or more consumers.
  Engineering artifact: breach runbook with the 30-day clock.
- **AI-specific consideration.** NPI used in a model training set is
  "customer information" under the Safeguards Rule; the training pipeline
  inherits the Rule's controls.

### 2.4 Worked implementation — mixed trigger example (all four fire)

A startup builds a "learning companion" for K-12 schools that plugs into an
LMS (education records), offers a tele-counselling feature staffed by
licensed clinicians working through a provider partner (PHI), stores a
prepaid-card-style stipend for school supplies (financial product of a
partner bank), and markets a sub-13-friendly version for elementary schools
(children's PI).

- **FERPA** attaches through the LMS contract (school-official exception)
  with direct-control and purpose-limitation obligations. Training on
  student-identifiable records requires authorisation.
- **HIPAA** attaches through the tele-counselling provider (covered entity)
  that signs a BAA with the startup for the therapy transcripts.
- **COPPA** attaches to the sub-13 product; direct notice and VPC required
  before any PI collection.
- **GLBA Safeguards Rule** attaches to the stipend card feature if the
  startup is deemed a service provider to a financial institution (or is
  itself significantly engaged in financial activities).

Engineering plan: segregate PHI, education records, and NPI into distinct
tenancy boundaries, each with its own KMS key, access controls, and training
exclusion. Train the general product only on data the engineer can prove is
not in any of the regulated pools. Add a child-flag path for the sub-13
instance. Attach each regime's breach clock to the incident-response
playbook with a *per-regime* notification matrix.

### 2.5 Counter-example: no trigger

A direct-to-consumer "mood journal" app, operated by a non-health, non-
financial, non-education company, aimed at adults, that collects mood
check-ins and generates insights. It is **not** HIPAA-covered (no covered
entity), **not** FERPA-covered (no school), **not** COPPA-covered (not
directed to children under 13 and the operator has no actual knowledge of
child users — barring a design that invites them), and **not** GLBA-covered
(not a financial institution). It is still subject to CPRA, VCDPA, and state
equivalents, FTC Section 5, and any relevant state-health-data regimes
(e.g., Washington My Health My Data Act) — but the four sectoral regimes in
this exercise do not attach.

## 3. Validation steps

1. **Trigger-first reading.** For each regime, re-read the trigger text
   before anything else. If the trigger is not met, the rest of the regime
   does not apply.
2. **Role test.** Identify the role: covered entity vs business associate
   (HIPAA); school vs school official (FERPA); operator vs third-party
   service (COPPA); financial institution vs service provider (GLBA). Role
   determines which duties attach directly vs by contract.
3. **Negative check.** Build a counter-example product in each category and
   justify why the regime does *not* attach. If the learner cannot articulate
   a negative, they probably do not understand the trigger.
4. **Overlap check.** Build at least one mixed product where two or more
   regimes attach. Verify that duties are not double-counted and that
   conflicts are resolved by the more protective rule.
5. **Rule-text spot check.** Pick five cited sections from the map at random
   and open the current CFR text at <https://www.ecfr.gov/>. The paraphrase
   in the map must survive the open-book check.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Entity trigger correctly stated for all four regimes | 2 | Omitting "business associate" or "school official" loses a point each |
| Data trigger correctly stated for all four regimes | 1 | PHI, education records, child PI, NPI — with the real definitions |
| Duties include at least one baseline engineering control per regime | 2 | BAA/minimum necessary; school-official constraints; VPC/direct notice; Safeguards + Reg P notice |
| Breach-notification clocks listed per regime | 1 | 60 days (HIPAA BA → CE), 30 days (FTC Safeguards notification event), plus the state-law layer |
| Worked mixed example shows per-regime segregation (keys, tenancy, training exclusion) | 2 | Collapsing into a single "security posture" fails |
| Counter-example articulates why each regime does *not* attach | 1 | Must name the missing trigger |
| Each duty points at an engineering artifact a reviewer can inspect | 1 | Not a policy |

Automatic failures:
- Claiming HIPAA applies to all health data (it does not).
- Claiming FERPA applies to all K-12 tech (only when the trigger is met).
- Claiming COPPA applies to all minors (it covers under 13; teen privacy is
  largely state law).
- Claiming GLBA requires encryption absolutely (it is an addressable /
  context-specific safeguard; the Rule still requires a reasoned
  implementation).

## 5. Common mistakes

- **"Health data = HIPAA."** HIPAA is about who holds the data, not about the
  data's topic. Consumer wellness apps outside the covered-entity chain are
  generally *not* HIPAA-covered, though they may be subject to other regimes
  (state law, FTC Health Breach Notification Rule, Washington MHMDA).
- **"Schools are covered, we're not."** The school-official exception
  passes most of FERPA's substantive duties to the vendor by contract. If
  you are a school's AI vendor, you are handling education records under
  FERPA's rules even though the statute names the school.
- **Confusing COPPA with state teen-privacy laws.** COPPA sets a federal
  floor for under-13s. Teen privacy laws (e.g., CA AB-2273-era rules, UT, CT
  minor provisions) stack on top and are not COPPA.
- **Assuming GLBA only matters for banks.** The FTC's Safeguards Rule covers
  non-bank financials including many fintechs, mortgage brokers, consumer
  finance providers, and credit counsellors.
- **Training on regulated data by default.** PHI, education records, and
  child PI are the three categories where "use the data to improve the
  model" needs explicit authorisation before the training pipeline ever
  touches the data. NPI similarly inherits Safeguards Rule controls.
- **Single breach clock.** The regimes have different notification clocks
  and different recipients. Hard-code a per-regime matrix in the IR runbook.
- **Treating encryption as the whole Safeguards Rule.** The Rule has nine
  prescriptive elements in § 314.4; encryption is one.

## 6. References

- HIPAA Privacy, Security, and Breach Notification Rules — 45 CFR Parts 160
  and 164. <https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C>
- HHS Office for Civil Rights — HIPAA for professionals.
  <https://www.hhs.gov/hipaa/for-professionals/index.html>
- FERPA — 20 U.S.C. § 1232g and 34 CFR Part 99.
  <https://www.ecfr.gov/current/title-34/subtitle-A/part-99>
- US Department of Education — Student Privacy Policy Office.
  <https://studentprivacy.ed.gov/>
- COPPA — 15 U.S.C. §§ 6501–6506 and 16 CFR Part 312.
  <https://www.ecfr.gov/current/title-16/chapter-I/subchapter-C/part-312>
- FTC — Complying with COPPA.
  <https://www.ftc.gov/business-guidance/privacy-security/childrens-privacy>
- Gramm-Leach-Bliley Act — 15 U.S.C. §§ 6801–6809.
- FTC Safeguards Rule — 16 CFR Part 314.
  <https://www.ecfr.gov/current/title-16/chapter-I/subchapter-C/part-314>
- Regulation P (Consumer Financial Information) — 12 CFR Part 1016.
  <https://www.ecfr.gov/current/title-12/chapter-X/part-1016>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
