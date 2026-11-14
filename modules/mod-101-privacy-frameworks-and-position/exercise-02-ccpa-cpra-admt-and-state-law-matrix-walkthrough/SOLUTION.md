# Exercise 02 — CCPA / CPRA + ADMT + US State-Law Matrix Walkthrough — Reference Solution

## 1. Solution overview

This is a **design-and-interpretation exercise**. The learner is asked to
(a) locate the California-specific obligations that an AI product team must
engineer to, with special attention to the California Privacy Protection
Agency's (CPPA) **Automated Decisionmaking Technology (ADMT)** regulations;
(b) place California alongside the other "comprehensive" US state privacy
laws without flattening their differences; and (c) decide, per processing
activity, which rights and which engineering duties actually bite.

The assessable output has three parts:

1. A **state-law matrix** across (at minimum) California, Virginia, Colorado,
   Connecticut, Utah, Texas, Oregon, and Delaware — pick a stable set of
   columns that are meaningful for engineering: *applicability threshold*,
   *sensitive-data handling*, *targeted advertising / sale / sharing opt-out*,
   *profiling opt-out*, *data-protection assessment duty*, and *universal
   opt-out signal recognition*.
2. A **CCPA / CPRA deep dive** that routes specific product features to the
   right regulatory hook: sale/share, cross-context behavioural advertising
   (CCBA), sensitive-personal-information (SPI) use limitations, and the
   CPPA's ADMT rules.
3. A **one-page walkthrough** of a sample product (recommended: an AI-driven
   hiring screener) showing which rights and which assessments fire, which
   artifact owns each duty, and where the law stops and professional judgement
   starts.

## 2. Worked answer

### 2.1 State-law matrix (engineer-oriented)

Columns are the ones that most often change the shape of the pipeline. Rows
are state statutes as cited; where a state regulation (not the statute) adds a
duty, it is cited explicitly.

| State / Law | Applicability (business threshold) | Sensitive-data mechanism | Opt-out: sale / share / targeted advertising | Opt-out: profiling that produces legal or similarly significant effects | Assessment duty | Universal opt-out signal (UOOM) honoured |
|---|---|---|---|---|---|---|
| California — CCPA as amended by CPRA (Cal. Civ. Code § 1798.100 et seq.); CCPA Regs; CPPA ADMT Regs (adopted 2025) | Business ≥ $25M revenue, or buys/sells/shares PI of ≥ 100,000 CA residents/households, or ≥ 50% revenue from selling/sharing PI | Right to limit use of SPI (§ 1798.121); SPI defined in § 1798.140(ae) | Opt-out of sale and sharing (§ 1798.120); "sharing" = cross-context behavioural advertising | Access, opt-out, and pre-use notice for ADMT per CPPA Regs; see 2.2 | Risk assessments for high-risk processing per CPPA Regs | Required: Global Privacy Control via CCPA Regs § 7025 |
| Virginia — Consumer Data Protection Act (VCDPA), Va. Code § 59.1-575 et seq. | Controls or processes PD of ≥ 100,000 VA consumers, or ≥ 25,000 and > 50% gross revenue from sale | Opt-in consent for sensitive data (§ 59.1-578(A)(5)) | Opt-out of targeted advertising and sale (§ 59.1-577) | Opt-out of profiling in furtherance of decisions that produce legal or similarly significant effects (§ 59.1-577(A)(3)) | Data protection assessment (§ 59.1-580) | Not required by statute; honour recommended as good engineering |
| Colorado — Colorado Privacy Act (CPA), Colo. Rev. Stat. § 6-1-1301 et seq.; CPA Rules (4 CCR 904-3) | Same shape as VCDPA with CO thresholds | Opt-in consent for sensitive data; "sensitive data inferences" defined by Rule | Opt-out of targeted advertising, sale, profiling (§ 6-1-1306) | Yes, same hook | Data protection assessment (§ 6-1-1309) | Required: UOOM via CPA Rules; Colorado AG maintains an approved list |
| Connecticut — CTDPA, Conn. Gen. Stat. § 42-515 et seq. | Same shape as VCDPA | Opt-in consent for sensitive data | Opt-out of targeted advertising, sale, profiling | Yes | Data protection assessment | Required as of 2025 per statute |
| Utah — Utah Consumer Privacy Act (UCPA), Utah Code § 13-61-101 et seq. | Business ≥ $25M revenue + volume thresholds | Opt-out (not opt-in) for sensitive data | Opt-out of targeted advertising and sale | No profiling opt-out in statute | No general assessment duty | Not required |
| Texas — Data Privacy and Security Act (TDPSA), Tex. Bus. & Com. Code § 541.001 et seq. | Conducts business in TX; sells PD; not a small business under SBA size standards | Opt-in consent for sensitive data | Opt-out of targeted advertising, sale, profiling | Yes | Assessment required for targeted advertising, sale, sensitive data, profiling | Not required by statute |
| Oregon — Oregon Consumer Privacy Act, ORS § 646A.570 et seq. | ≥ 100,000 OR consumers, or 25,000 + 25% revenue from sale | Opt-in consent for sensitive data; broader SPI definition than many peers | Opt-out of targeted advertising, sale, profiling | Yes | Assessment required | Required: UOOM honoured |
| Delaware — Delaware Personal Data Privacy Act, 6 Del. C. § 12D-101 et seq. | ≥ 35,000 DE consumers, or 10,000 + > 20% revenue from sale | Opt-in consent for sensitive data | Opt-out of targeted advertising, sale, profiling | Yes | Assessment required | Required: UOOM honoured |

Additional columns the learner may add (not scored as required): breach
notification trigger, private right of action scope, children's / minor
provisions (COPPA still sets the floor), employee/B2B coverage (California
removed the employee exemption after Jan 1, 2023; most peers exempt
employee/B2B data), cure period.

### 2.2 CCPA / CPRA deep dive: which hook does your product hit?

**Step 1 — Classify the data.**
- Is any element in the § 1798.140(ae) **SPI** list (SSN, DL, financial
  account + credentials, precise geolocation, racial/ethnic origin, religion,
  mail/email/text contents not addressed to you, genetic data, biometric data
  for identification, health, sex-life/orientation)? If yes, the **right to
  limit use of SPI** (§ 1798.121) applies and the business must present a
  "Limit the Use of My Sensitive Personal Information" link unless it falls
  into the narrow enumerated purposes in § 1798.121(a).
- "Inferences drawn from PI to create a profile reflecting preferences,
  characteristics, psychological trends, predispositions, behaviour,
  attitudes, intelligence, abilities, and aptitudes" (§ 1798.140(v)(1)(K)) is
  **Personal Information**. A derived embedding or score is covered the same
  way the raw input is.

**Step 2 — Classify the processing.**
- Does the business **sell** or **share** PI? CPRA's "share" (§ 1798.140(ah))
  covers disclosing PI to a third party for *cross-context behavioural
  advertising*, even without monetary exchange. If yes, the "Do Not Sell or
  Share My Personal Information" opt-out + UOOM (GPC) honour is required.
- Does the business operate **ADMT**? CPPA's ADMT regulations (adopted 2025;
  phased in on the schedule set by the CPPA) define ADMT and require a
  pre-use notice, an opt-out from ADMT in specified contexts, a right of
  access to ADMT outputs, and a risk assessment for significant decisions
  using ADMT. The exact scope — including which decisions count as
  "significant decisions" and which training activities are in scope — is
  set by the CPPA's regulatory text; engineers should confirm the operative
  version and the current phase-in date at <https://cppa.ca.gov/regulations/>
  at build time, since the CPPA finalised the ADMT, risk-assessment, and
  cybersecurity-audit rules in 2025 with staggered compliance dates rather
  than a single effective date.

**Step 3 — Pick the artifact.** Each California hook maps to one engineering
artifact, consistent with mod-101 ex-01:

| CA hook | Engineering artifact |
|---|---|
| Right to know / access (§ 1798.110) | DSR access endpoint; data inventory with the categories in § 1798.110(c) |
| Right to delete (§ 1798.105) | DSR deletion endpoint with downstream cascading delete, including feature store, vector index, and training-set membership |
| Right to correct (§ 1798.106) | DSR correction endpoint and downstream recompute |
| Opt-out of sale and sharing (§ 1798.120) | GPC listener + do-not-sell/share flag in the ad-tech integration |
| Right to limit use of SPI (§ 1798.121) | Purpose-limiting flag that filters SPI from any processing outside § 1798.121(a) purposes |
| ADMT opt-out, access, pre-use notice | Pre-use notice served at the decision point; ADMT inventory entry; access endpoint returns the output, the "key parameters" explanation, and (where required) an alternative or appeal path |
| Risk assessment (CPPA Regs) | Completed assessment in the DPIA / privacy-review register (see mod-111) |
| Universal opt-out signal (GPC) | Server-side listener for `Sec-GPC: 1`; integration test that asserts the sale/share flag flips on GPC-present requests |

### 2.3 Worked implementation — AI-driven hiring screener walkthrough

Facts (local exercise context; the product described here is hypothetical):
A SaaS company operates a résumé-scoring service used by employers. The model
scores candidates, surfaces a top-N list, and provides a written rationale.
Candidates' résumés are collected through the employer's portal. The company
also trains model updates on aggregated scoring data. The service is sold to
employers in CA, VA, CO, CT, UT, TX, OR, and DE.

Walkthrough answers the question *"what duties fire, in which state?"*:

- **Role.** The screener company is a **business** / **controller** in the
  consumer-facing sense only when it is making decisions about candidates on
  its own account. When it is scoring candidates on an employer's
  instructions, in most states it is a **service provider** / **processor** —
  with contractual duties and reduced direct consumer-facing duties, but
  significant residual engineering obligations (security, deletion on
  request, instructions-only processing).
- **Applicability.** Assume thresholds are met in each of the eight states.
- **Lawful hooks that fire regardless of state:**
  - Access, deletion, correction rights to candidates whose data is stored on
    the controller side.
  - Security and breach notification.
  - Service-provider / processor contractual terms (CCPA § 1798.140(ag) and
    equivalent).
- **California-specific hooks:**
  - **ADMT pre-use notice** at the point where the score materially drives a
    hiring decision (shortlist, screen-out). The notice names what the system
    does, how it is used, and how to opt out *where opt-out is available*.
  - **Right to access** the ADMT output for the candidate, with "meaningful
    information about the logic" at an engineer-usable granularity.
  - **Risk assessment** filed internally (not a submission requirement in CA,
    but required to be produced on request). The assessment covers accuracy,
    discrimination risk, data minimisation, and the alternative to ADMT.
  - **SPI right to limit** if the model inadvertently infers or uses any SPI
    category (e.g., inferred national origin from a name — the engineer must
    affirmatively prevent this).
  - **Employee / applicant coverage** — CPRA removed the HR exemption in
    2023, so applicant and employee data are in scope. In most peer states
    they are not.
- **Peer-state hooks that fire differently:**
  - **CO, CT, OR, DE, TX, VA:** profiling opt-out for decisions with legal or
    similarly significant effects — hiring is squarely in scope. Engineering
    artifact: a profiling opt-out signal in the candidate flow that, if
    exercised, routes the candidate out of ADMT. If the employer is the
    controller, the service must *propagate* the opt-out from employer to
    processor.
  - **CO, OR, DE (and CA via GPC):** honour the universal opt-out signal —
    engineering artifact is a server-side listener on the controller surface;
    a processor-only deployment can rely on the controller passing the
    signal through.
  - **UT:** no profiling opt-out, no UOOM — but sensitive-data opt-out still
    applies.
  - **TX:** assessment required for profiling; UOOM not required by statute.
- **Engineering plan:**
  1. Add an ADMT inventory entry describing the scoring pipeline, the
     training set, the key features, the decision point, and the owner.
  2. Add a pre-use notice component to the candidate portal and parameterise
     it by the controller's jurisdiction.
  3. Add a profiling / ADMT opt-out endpoint and wire it into the scoring
     service as a hard filter.
  4. Add the GPC listener on the controller surface.
  5. Add the risk assessment to the DPIA register and link it to the model
     card.
  6. Add a periodic accuracy and disparate-impact evaluation whose output is
     part of the assessment evidence.

## 3. Validation steps

1. **Statute check.** Open each cited section in the official source:
   California: <https://leginfo.legislature.ca.gov/>. Other states' chapter
   pages on the state legislature's site. The learner's matrix must survive
   a spot-check on at least five random cells.
2. **CPPA regulatory check.** Open the CPPA's regulations page and locate
   the most recent operative text covering ADMT, risk assessments, and
   cybersecurity audits: <https://cppa.ca.gov/regulations/>. If any claim in
   the matrix depends on regulatory (not statutory) text, the cite must point
   at the regulation, not the statute.
3. **UOOM check.** Visit the Global Privacy Control project page at
   <https://globalprivacycontrol.org/>. The matrix row that lists GPC must
   describe a server-side behaviour, not a browser setting alone.
4. **Service-provider vs business check.** Identify, for each row of the
   walkthrough, which entity is the controller and which is the processor /
   service provider. If the learner has the screener company as both
   simultaneously for the same activity, rework.
5. **Threshold check.** Re-run the applicability threshold for each state.
   States differ on whether revenue alone is sufficient (CA, UT) or whether a
   volume threshold also applies (VA, CO, CT, OR, DE, TX).

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Matrix covers ≥ 8 states and includes the required engineer-oriented columns | 2 | One point lost per missing column |
| Each row cites a specific statute or regulation, not just the state name | 1 | Deduct for "California says…" |
| CCPA / CPRA deep dive separates sale, share (CCBA), SPI limit, and ADMT | 2 | Collapsing these loses points |
| ADMT section identifies that the duty flows from CPPA *regulations*, not the statute alone | 1 | A common failure |
| Walkthrough correctly distinguishes controller from processor / service provider | 1 | Role must be assigned per activity |
| Walkthrough maps each fired duty to an engineering artifact a reviewer can inspect | 2 | Not a policy |
| Walkthrough names at least one duty that fires in peer states but not CA, and one that fires in CA but not peers | 1 | Prevents "copy the CA answer" |

Automatic failures:
- A matrix that treats "US state privacy laws" as a single thing.
- Omitting the UOOM / GPC row.
- Claiming ADMT opt-out applies to all uses of any model; the regulation
  targets significant decisions and specified contexts, not all inference.
- Any reference to "CPRA section 22" without a sub-cite — § 1798.185 and
  the CPPA regulations are the right targets.

## 5. Common mistakes

- **Treating California as the ceiling.** It is often the ceiling for
  consumer-facing rights, but other states have broader sensitive-data
  opt-in regimes and separate assessment duties that CA does not mirror
  line-for-line.
- **Conflating "sale" and "sharing" under CPRA.** Share is defined
  specifically for cross-context behavioural advertising and has no
  consideration requirement — most ad-tech integrations trigger "share"
  even when nothing is being "sold."
- **Treating SPI as a stricter version of PI.** SPI opens a distinct *right
  to limit*, not a stricter version of access/deletion. SPI inferences count,
  which surprises engineers used to thinking only about raw inputs.
- **Confusing ADMT with "all automated processing."** The CPPA's rules
  target defined ADMT uses — most prominently significant decisions and the
  specified training and extensive-profiling contexts. The engineer still
  owes the GDPR Art. 22 baseline, but ADMT specifics must be read from the
  regulation, not transcribed from GDPR.
- **Pretending UOOM is optional in states where it isn't.** California,
  Colorado, Connecticut, Oregon, and Delaware require it in some form; the
  engineering artifact is a server-side listener, not a client-side banner.
- **Forgetting that CA covers employee and applicant data.** Hiring tools
  that rely on an HR exemption are relying on an exemption California no
  longer grants.
- **Writing the matrix once and letting it rot.** States amend these laws
  almost yearly. Attach a "last-validated" date to the matrix and treat it as
  code.

## 6. References

- California Consumer Privacy Act as amended by the California Privacy
  Rights Act — Cal. Civ. Code § 1798.100 et seq.
  <https://leginfo.legislature.ca.gov/faces/codes_displayText.xhtml?division=3.&part=4.&lawCode=CIV&title=1.81.5>
- California Privacy Protection Agency — Regulations.
  <https://cppa.ca.gov/regulations/>
- Virginia Consumer Data Protection Act — Va. Code § 59.1-575 et seq.
  <https://law.lis.virginia.gov/vacodefull/title59.1/chapter53/>
- Colorado Privacy Act — Colo. Rev. Stat. § 6-1-1301 et seq., and Colorado
  Attorney General CPA Rules (4 CCR 904-3).
  <https://coag.gov/office-sections/consumer-protection/colorado-privacy-act/>
- Connecticut Data Privacy Act — Conn. Gen. Stat. § 42-515 et seq.
- Utah Consumer Privacy Act — Utah Code § 13-61-101 et seq.
- Texas Data Privacy and Security Act — Tex. Bus. & Com. Code § 541.001 et
  seq.
- Oregon Consumer Privacy Act — ORS § 646A.570 et seq.
- Delaware Personal Data Privacy Act — 6 Del. C. § 12D-101 et seq.
- Global Privacy Control project. <https://globalprivacycontrol.org/>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework> — used for the
  "significant decisions" vocabulary and for the governance role of
  ADMT impact assessments.
