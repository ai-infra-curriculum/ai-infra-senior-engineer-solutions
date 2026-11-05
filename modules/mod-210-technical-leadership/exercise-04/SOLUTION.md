# SOLUTION — Lab 04: Governance Leadership for ML Platforms

Reference solution for
[lab-04-governance-leadership.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-210-technical-leadership/exercises/lab-04-governance-leadership.md).

Governance for an ML platform is the set of long-lived
agreements — policies, roles, review gates, and metrics —
that let many teams build and operate AI systems safely
without concentrating every decision in a central
governance team. This lab's deliverable is a **governance
charter + operating model** for the platform, graded
against how well it names standards, owners, intake, and
exceptions.

The senior-engineer altitude here is *tech-lead for
governance*: translating NIST AI RMF, OWASP ML Top 10, and
MITRE ATLAS into *operable* structures for the platform
team — not writing policy at a principal/staff altitude,
and not managing a governance organisation.

## 1. Solution overview

The reference governance design has six parts — the same
layering mod-209 uses for its security program, re-pointed
at the broader ML-governance surface
(see `modules/mod-209-security-compliance/exercise-04/SOLUTION.md`
§2 for the sister structure):

1. **Charter and mandate.** One page. Why this exists,
   what authority the ML governance function holds, and
   what is explicitly out of scope (e.g., corporate IT
   security, product policy).
2. **Framework mapping.** The platform's controls mapped
   against **NIST AI RMF 1.0** functions (GOVERN, MAP,
   MEASURE, MANAGE —
   [NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework)),
   with cross-references to **OWASP ML Top 10** risks
   ([OWASP ML Top 10](https://owasp.org/www-project-machine-learning-security-top-10/))
   and **MITRE ATLAS** techniques
   ([MITRE ATLAS](https://atlas.mitre.org/)).
3. **Control catalogue.** The structural controls the
   platform enforces — model registry, admission policy,
   lineage, eval gating, red-team gate, incident
   playbook — each with an owner and the enforcement
   surface (admission vs. CI vs. runtime).
4. **Operating model.** RACI; intake for new models and
   datasets; the exception process; the escalation path.
5. **Metrics and reporting.** The small number of
   platform-level indicators the function reports,
   including coverage of the controls and age of open
   exceptions.
6. **Roadmap and revisit cadence.** A 12-month rolling
   plan and the quarterly review where the above gets
   refreshed.

Senior additions on top of a governance slide deck:

- **Governance is a product.** It has a backlog, SLOs, a
  release cadence, and a quarterly review — not a one-off
  sign-off.
- **Exceptions are first-class.** Most governance programs
  fail when exceptions quietly become the norm; the
  operating model names how exceptions are granted,
  logged, and expired.
- **Framework anchors, local shape.** The design uses NIST
  AI RMF subcategory IDs, OWASP ML risk IDs, and ATLAS
  technique IDs by name, not paraphrase. Successors can
  audit against the same anchors.

## 2. Worked answer or implementation

### 2a. Charter (one page, signed)

> **Mission.** Reduce the probability and impact of
> harm (safety, security, privacy, reliability) from the
> organisation's ML systems to a level the business has
> explicitly accepted, through structural controls owned
> by the ML platform team and operated by the teams that
> build and run those systems.
>
> **Scope.** ML training and evaluation pipelines; the
> model registry; inference serving; feature stores; the
> development paths that feed them. Out of scope: user
> account security (IT Security), physical security,
> non-ML corporate systems, product-level policy (e.g.,
> which features ship).
>
> **Framework alignment.** NIST AI RMF 1.0 (all four
> functions) is the primary frame; AI 600-1 generative-AI
> profile applies to the generative subset
> ([NIST.AI.600-1](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf)).
> OWASP ML Top 10 and MITRE ATLAS are the primary threat
> references.
>
> **Authority.** The ML Platform tech-lead owns the
> control catalogue (§2c) and the admission policies that
> enforce it. Changes to the catalogue require review by
> the Governance Working Group (§2d). The Governance
> Working Group does **not** sign off on individual model
> launches — that is the owning team's decision against
> the control catalogue.
>
> **Review.** Reviewed quarterly by the Governance Working
> Group. Charter versioned in the platform repo.

### 2b. NIST AI RMF function mapping

The platform maps controls to the four NIST AI RMF
functions. Each entry names the subcategory the control
addresses; every control catalogue entry in §2c cites back
here.

| Function | Platform expression |
|---|---|
| **GOVERN** — structures, policies, roles | Charter (§2a), control catalogue (§2c), operating model (§2d), incident process (§2e). |
| **MAP** — context and intended use | Model card (per model) with intended-use, data-source, and known-limitation sections; populated before registry promotion to Staging. |
| **MEASURE** — instrumentation | Eval harness run at promotion; serving-time drift and abuse metrics; lineage completeness metric (§2f). |
| **MANAGE** — response over time | Incident playbook (§2e); exceptions register with expiry; quarterly control review. |

The mapping is 1:N — a single control (e.g., the model
registry) touches MAP (lineage), MEASURE (eval
completeness), and MANAGE (promotion revocation). The
rubric in §4 does not require one-to-one.

### 2c. Control catalogue (the structural half)

Each entry: control name, risk addressed
(NIST subcategory / OWASP ML / ATLAS technique),
enforcement surface, owner, revisit cadence.

```markdown
C01 — Model registry as system of record
  Risk:      NIST AI RMF MAP-4.1 (lineage), MANAGE-4.1
             (reproducibility), OWASP ML06 (supply chain)
  Enforced:  KServe admission; manifests reject non-registry URIs
  Owner:     ML platform (tech-lead = Rivera)
  Revisit:   Annually, or on registry vendor change

C02 — Model artefact signature verification
  Risk:      OWASP ML06 (supply chain),
             ATLAS AML.T0019 (Model Replication / Tampering)
  Enforced:  Cosign signature check at serving admission
             (see mod-209-security-compliance SOLUTION §2f)
  Owner:     ML platform jointly with Security
  Revisit:   On key rotation; after any cosign CVE

C03 — Required model card at promotion to Staging/Prod
  Risk:      NIST AI RMF MAP-3.3 (intended use),
             MAP-1.1 (context)
  Enforced:  Registry promotion hook rejects missing/stale card
  Owner:     ML platform
  Revisit:   Semi-annually

C04 — Eval harness gate on promotion to Production
  Risk:      NIST AI RMF MEASURE-2.1/2.3 (eval + test),
             OWASP ML08 (model skewing)
  Enforced:  Registry promotion hook; eval scores + dataset
             fingerprint stamped on model version
  Owner:     ML platform; eval framework owned by MLOps
  Revisit:   Quarterly (eval suite); annually (gate design)

C05 — Red-team review gate for generative models
  Risk:      NIST AI 600-1 (GAI profile),
             OWASP ML01 (input manipulation),
             ATLAS AML.T0051 (LLM Prompt Injection)
  Enforced:  Procedural gate at Staging->Prod for models
             flagged `category: generative`
  Owner:     Security + ML platform
  Revisit:   Quarterly

C06 — Default-deny network egress from training clusters
  Risk:      OWASP ML07 (transfer learning),
             ATLAS AML.T0010 (ML Data Access Exfiltration)
  Enforced:  NetworkPolicy; see mod-209 §2d
  Owner:     Platform + Security
  Revisit:   Annually

C07 — Incident classification and timeline
  Risk:      NIST AI RMF MANAGE-4.3 (response)
  Enforced:  On-call playbook (§2e); SIEM-backed timeline
  Owner:     ML platform on-call rotation
  Revisit:   After every Sev1/2

C08 — Exceptions register
  Risk:      NIST AI RMF GOVERN-4.1 (change control)
  Enforced:  Exceptions file in platform repo; expiry stamp
             checked by CI; exception renewal is a PR
  Owner:     ML platform tech-lead
  Revisit:   Every exception reviewed at or before expiry
```

A real catalogue grows to ~15-25 controls; this reference
set is the minimum to demonstrate the shape and the four
required ingredients per control: risk anchor, enforcement
surface, owner, revisit cadence.

### 2d. Operating model (the human half)

**Governance Working Group.** Monthly. Membership: ML
platform tech-lead (chair), two product-team ML leads on
rotation, Security representative, Legal/Privacy liaison
when agenda requires. Charter: review intake of new model
categories, exception requests, catalogue changes. **Not**
a per-model sign-off body.

**RACI (abbreviated).**

| Activity | Platform tech-lead | Owning team | GWG | Security |
|---|---|---|---|---|
| Register model version (Dev) | I | R/A | - | - |
| Promote to Staging (card + eval present) | C | R/A | - | I |
| Promote to Prod (eval gate + signature) | C | R/A | I | C |
| Add generative category | R | A | C | C |
| Approve C05 red-team exemption | I | R | A | C |
| Change control catalogue (C01-C08) | A | C | C | C |
| Incident response (Sev1/2 model) | C | R/A | I | C |

R = responsible, A = accountable, C = consulted,
I = informed.

**Intake.** A new model category (e.g., "we want to serve
a multimodal retrieval model") enters via a one-page
intake form in the platform repo. The form names the
NIST AI RMF MAP-1.1 context, OWASP ML risks the owning
team sees as in-scope, and any known ATLAS adversary
techniques. The platform tech-lead and GWG review in the
next monthly meeting.

**Exception process.** An exception is a *time-bounded*
waiver from a control. Submitted as a PR to
`governance/exceptions.yaml`:

```yaml
- id: EX-2026-004
  control: C04
  reason: >
    Model `search-rerank-v2` promoted to Prod with 1
    missing eval metric (fairness). Owning team committed
    to backfill by 2026-05-01.
  requested_by: team-search (lead: K. Osei)
  approved_by: platform-tech-lead (J. Rivera)
  risk_accepting_authority: director-of-mlops
  opened: 2026-02-28
  expires: 2026-05-01
  review: GWG 2026-04-20
```

CI rejects any exception that is expired, lacks a named
risk-accepting authority at director level or above, or
exceeds 180 days without re-approval. This is the single
design element that keeps "temporary" from becoming
permanent.

**Escalation.** Deadlock between platform and an owning
team on a control call goes to the GWG. Director-level
escalation is for risk acceptance on exceptions only; the
GWG is not a tie-breaker for engineering preferences.

### 2e. Incident playbook (abridged)

Severity ladder (model-specific):

- **Sev1** — model is producing incorrect, harmful, or
  policy-violating outputs at scale; paging.
- **Sev2** — control is bypassed (unsigned model in prod,
  stale card, failed eval gate); paging during business
  hours.
- **Sev3** — individual complaint or single-request
  anomaly; ticketed.

On page:

1. **Isolate.** Freeze promotion for the affected model;
   do not roll back immediately — gather state first.
2. **Timeline.** SIEM-backed, from first signal.
3. **Root cause with the five-whys shape.** Avoid
   "the engineer should have known"-type findings (see
   `modules/mod-210-technical-leadership/SOLUTION.md`
   §"Common mistakes graders see" #4).
4. **Postmortem.** Owned by platform on-call; published
   within 10 business days; action items with owners and
   dates; linked from the model card.
5. **Governance review.** Any Sev1 or repeat Sev2
   triggers a control-catalogue review item in the next
   GWG.

### 2f. Metrics and reporting

Four indicators, reported monthly:

- **Control coverage.** % of production model versions
  satisfying every enabled control (C01-C06). Target 100%
  with named exceptions covering the gap.
- **Exception age.** Median and P90 age of open
  exceptions; any > 90 days flagged on the dashboard.
- **Eval-gate hit rate.** % of Staging->Prod promotions
  blocked by C04 in the last 30 days (healthy: non-zero;
  zero means the gate is loose).
- **Incident count by severity.** Trailing 90-day Sev1
  and Sev2 counts, with per-incident links to the
  postmortem.

Reported into the same quarterly review the charter is
reviewed in. The same metrics drive the executive-level
narrative; no separate deck.

### 2g. Roadmap (12-month rolling)

Example first quarter:

- **Q1.** Close the "no cosign on model artefacts" gap
  (C02). Adopt generative-AI red-team gate (C05) for the
  one generative model in scope.
- **Q2.** Automate exception-expiry enforcement in CI
  (§2d). Roll out drift metrics for all prod models.
- **Q3.** External review against NIST AI RMF; address
  the two to three highest-rated gaps.
- **Q4.** Review and refresh catalogue; retire any
  controls superseded by platform changes.

Roadmap items get owners and quarter commits; the GWG
reviews slippage monthly.

## 3. Validation steps

1. **Charter test.** A new engineer joins the ML platform
   team and reads the charter. Can they answer: what's in
   scope, who owns catalogue changes, and who signs off on
   a Prod promotion? If not, the charter is thin.
2. **Framework-anchor test.** For every control in §2c,
   the risk line names at least one of a NIST AI RMF
   subcategory ID, an OWASP ML risk ID, or an ATLAS
   technique ID. No control cites the framework by
   paraphrase only.
3. **Enforcement-surface test.** Each control names a
   concrete enforcement surface (admission policy, CI
   gate, registry hook, SIEM rule), not "team discipline".
4. **Exception-process test.** The exception YAML in §2d
   includes expiry, risk-accepting authority at
   director+, and a review date. Attempt to add one
   without expiry; the CI check (described in §2d)
   rejects it.
5. **RACI test.** For each row in §2d, exactly one R and
   one A. Rows with two As are ambiguous ownership.
6. **Metrics test.** Each metric in §2f is backed by a
   query against a system of record (registry, CI,
   SIEM); none are "surveys".
7. **Review-cadence test.** Charter, catalogue, and
   roadmap all name a review period. No section is
   "whenever we remember."

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Charter answers scope, out-of-scope, authority, and review cadence on a single page | 10 |
| R2 | Framework mapping cites NIST AI RMF 1.0 functions by name and the AI 600-1 profile where generative-AI systems are in scope | 15 |
| R3 | Every control in the catalogue has (a) risk anchor citing NIST / OWASP ML / ATLAS, (b) concrete enforcement surface, (c) named owner, (d) revisit cadence | 20 |
| R4 | RACI covers at least the common lifecycle events (register, promote, incident, catalogue change) with exactly one R and one A per row | 10 |
| R5 | Exception process is first-class: expiry, risk-accepting authority at director+, CI enforcement of expiry, review date | 15 |
| R6 | Metrics are quantitative, system-of-record-backed, and tied to the governance review cadence; no survey-based or anecdotal metrics | 10 |
| R7 | Incident playbook names severity ladder, timeline sourcing, and blame-free root-cause framing | 10 |
| R8 | Roadmap is time-bound (12 months rolling), itemised by quarter, with owners and slippage review | 5 |
| R9 | Document is maintained as a product: versioned in-repo, quarterly review, successor-reader test passed | 5 |

Pass threshold: **80/100**. R2, R3, and R5 are must-haves
— a governance document that does not anchor to the
standards, does not make controls concretely enforceable,
or does not bound exceptions is not a governance program.

## 5. Common mistakes

1. **Governance as slide deck.** A deck exists; the
   backing repo, enforcement, and metrics do not.
   Governance that lives only in slides collapses the
   moment the author leaves the team.
2. **Paraphrased framework references.** "We align to
   NIST AI RMF" without naming a single subcategory. An
   auditor cannot verify alignment to a paraphrase.
3. **Controls without enforcement surface.** "Teams must
   add a model card" with no admission hook. The control
   becomes a norm, then a suggestion, then nothing.
4. **No exceptions process.** Everything is a must; nothing
   is bounded. Every real org has exceptions. Design them
   in, or they will happen outside the design.
5. **Exceptions without expiry.** The number-one failure
   mode. Exceptions become permanent; the catalogue
   becomes advisory. CI-enforced expiry is the simplest
   mitigation.
6. **Per-model sign-off by the GWG.** The GWG is a
   governance body, not a release train. Per-model
   decisions belong with the owning team against the
   catalogue; bringing them to the GWG creates
   bottlenecks that drive owners to go around.
7. **RACI with multiple accountable parties.** "Platform
   *and* Security are accountable for C02." In an
   incident, one of them is actually accountable; name
   it.
8. **No metric on exceptions.** Control coverage looks
   great; exception age is quietly growing. Both metrics
   have to be in the same report.
9. **Blame-shaped incident findings.** "The engineer
   should have known" cost many governance programs
   their legitimacy. Senior-engineer postmortems and the
   GWG should reject blame-shaped findings in review.
10. **Governance as annual exercise.** Written once, read
    never. A quarterly review and a visible metric are
    what keep the document load-bearing.

## 6. References

Primary (official standards and frameworks):

- NIST AI Risk Management Framework 1.0
  (GOVERN / MAP / MEASURE / MANAGE; subcategory IDs used
  throughout the control catalogue) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST AI 600-1 — Generative AI Profile (applied to the
  generative subset of in-scope systems) —
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
- OWASP Machine Learning Security Top 10 (ML01 input
  manipulation, ML06 supply chain, ML07 transfer
  learning, ML08 model skewing cited by control) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS (ML adversary technique catalogue; cited by
  technique ID in C02 and C05) —
  <https://atlas.mitre.org/>

Local context:

- Module-level framing on senior-vs-staff leadership
  altitude for governance:
  `modules/mod-210-technical-leadership/SOLUTION.md`
  §"Tech-lead-without-people-management is the focus".
- Sister structure for security program design (shares the
  six-layer shape and exceptions pattern):
  `modules/mod-209-security-compliance/exercise-04/SOLUTION.md`.
- Admission-side enforcement referenced by C01, C02:
  `modules/mod-209-security-compliance/SOLUTION.md` §2f.
- Related ADR for the registry choice referenced in C01:
  `modules/mod-210-technical-leadership/exercise-02/SOLUTION.md`
  §2a.
- Engineer-tier implementation references:
  `engineer-solutions/mod-106 ex-14` +
  `mlops-learning/projects/project-4-governance`
  (per module README cross-reference).
