# SOLUTION — Lab 05: Compliance Readiness Plan

Reference solution for
[lab-05-compliance-readiness-plan.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-209-security-compliance/exercises/lab-05-compliance-readiness-plan.md).

Where exercise-03 is the *audit* (producing evidence against
claimed controls) and exercise-04 is the *program* (owning
those controls over time), this exercise is the **gap-to-ready
plan**: you have a target framework, a current state, and a
deadline; what is the written, sequenced, resource-estimated
path from today to a successful first assessment?

## 1. Solution overview

A readiness plan is three things in one document:

1. **A mapping** — the target framework's subcategories
   against the current control inventory, with a gap/risk
   rating per row.
2. **A sequenced remediation plan** — the gaps, ordered
   by dependency and risk, each with owner, estimate, and
   acceptance criteria.
3. **A dress-rehearsal schedule** — the point in the
   timeline where the organisation stops *building* and
   starts *being assessed by its own team* before the
   external assessor arrives.

The target framework for this reference is NIST AI RMF 1.0
([NIST.AI.100-1](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf))
with the AI 600-1 generative-AI profile applied to any
generative-AI systems in scope
([NIST.AI.600-1](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf)).
The same shape applies to SOC 2, ISO/IEC 42001, or
sector-specific frames; the subcategory IDs change, the
plan structure does not.

Senior additions on top of "a status spreadsheet":

- **Dependencies are modelled.** Control X depends on
  control Y; starting X before Y produces rework. The plan
  carries a directed graph, not a flat list.
- **Dress rehearsal is a milestone, not a hope.** The plan
  includes an internal mock assessment at least 6 weeks
  before the external one, with a named mock-assessor.
- **Residual-risk register is a plan output**, not a
  punt. Items the plan deliberately does not close are
  written up with accepting authority and review date.

## 2. Worked answer or implementation

### 2a. Target definition (first deliverable)

Before any mapping, the plan fixes the target.

> **Framework.** NIST AI RMF 1.0, all four functions
> (GOVERN, MAP, MEASURE, MANAGE), scoped to in-scope
> systems (see exercise-03 §2a). AI 600-1 generative-AI
> subcategories applied to the model-serving platform.
>
> **Assessment type.** External third-party assessment with
> a 6-month continuous-operation lookback.
>
> **Target date.** External kickoff 2027-05-01, report by
> 2027-07-01. Backing off: internal readiness review
> 2027-03-15; evidence collection continuous from
> 2026-11-01 (today).
>
> **Definition of ready.** Every in-scope AI RMF
> subcategory has (a) a written control, (b) evidence of
> operation for the lookback window, (c) a named owner,
> and (d) a documented risk rating for any residual gap.

### 2b. Mapping and gap matrix (core artifact)

Keyed by AI RMF subcategory. Non-exhaustive excerpt — the
real artifact enumerates every in-scope subcategory.

| Subcategory | Statement (summarised) | Current control | Evidence source | Status | Gap | Risk rating |
|---|---|---|---|---|---|---|
| GOVERN 1.1 | Legal + regulatory requirements documented | `compliance/policy-register.md` | Git | **Partial** | Not reviewed in >12 months | Medium |
| GOVERN 1.2 | Risk-tolerance characterised | Not documented | — | **Gap** | No written risk appetite | High |
| GOVERN 1.4 | Roles + responsibilities for AI risk | `compliance/raci.md` | Git | **Ready** | — | Low |
| GOVERN 2.1 | Policies for AI lifecycle governance | Partial — model-release policy only | Git | **Partial** | No training-data policy | Medium |
| GOVERN 3.1 | Processes for team diversity + skills | HR policy exists | HR | **Ready** | — | Low |
| MAP 1.1 | Context of use documented | Model cards for 80% of prod models | Registry | **Partial** | 3 models missing cards | Medium |
| MAP 2.1 | AI system categorisation | Model-card field present | Registry | **Ready** | — | Low |
| MAP 3.4 | Impact on individuals assessed | Done for high-risk only | Registry | **Partial** | Low/med risk models not assessed | Medium |
| MAP 4.1 | Dataset provenance + rights | Lineage per training run | MLMD | **Ready** | — | Low |
| MEASURE 2.3 | Evaluation metrics documented | Per-model eval report at release | CI | **Ready** | — | Low |
| MEASURE 2.5 | Robustness + safety evaluated | Robustness only for public endpoints | CI | **Partial** | Internal endpoints not tested | Medium |
| MEASURE 2.6 | Drift + degradation monitored | C-MON-1 (ex-04) | Prom | **Ready** | — | Low |
| MEASURE 2.7 | Security of AI system evaluated | Pen test last year | SIEM | **Partial** | No pen test in 12mo window | High |
| MEASURE 3.1 | Human oversight assessed | Partial — on-call playbooks only | Confluence | **Partial** | No override UI documented | Medium |
| MANAGE 1.1 | Priorities for action documented | This plan | Git | **Ready** | — | Low |
| MANAGE 2.4 | Incident response for AI | AI incident runbook exists; last tabletop 18mo ago | Confluence | **Partial** | Tabletop cadence off | High |
| MANAGE 4.3 | Control reviews ongoing | Quarterly program review (ex-04) | Git | **Ready** | — | Low |

For generative-AI systems, the matching rows from AI 600-1
are added — in particular MP-2.3, MS-2.3-003 (robustness in
generative context), and the GV-6 vendor subcategories.

### 2c. Dependency graph (what blocks what)

Flat gap lists hide rework. Model remediation as a DAG and
sequence by topological order.

```
GOVERN 1.2 (risk appetite) ──► decides ──► MAP 3.4 (impact threshold)
                               │
                               └─► MANAGE 2.4 (IR severity matrix)

GOVERN 2.1 (training-data policy) ──► MAP 4.1 is already Ready, but
                                       MAP 1.1 model cards reference this policy
                                       so cards must update after the policy lands.

MEASURE 2.7 (pen test) ──► depends on exercise-01/02 controls being in Enforce
                           (otherwise the test finds the configuration gaps,
                           not the harder vulnerabilities)
```

Rule the plan follows: **ground-level policy before
process, process before measurement, measurement before
rehearsal.** A pen test scheduled before admission policy
is in Enforce mode finds admission-policy gaps and nothing
else.

### 2d. Sequenced remediation plan

Each gap becomes a work item with owner, estimate, acceptance
criteria, and dependencies. Representative shape:

```
WI-001  Draft risk-appetite statement
  Addresses: GOVERN 1.2
  Owner: Program owner, drafted; VP Eng signs
  Estimate: 2 weeks (mostly stakeholder interviews)
  Dependencies: none
  Acceptance:
    - 1-page document approved by VP Eng and CTO
    - Published in compliance/ repo
  Due: 2026-11-30

WI-002  Author training-data policy
  Addresses: GOVERN 2.1
  Owner: ML Platform lead
  Estimate: 3 weeks
  Dependencies: WI-001 (appetite informs retention / minimisation)
  Acceptance:
    - Policy document covering acquisition, classification,
      retention, deletion, and consent basis
    - Referenced from every in-scope model card by 2027-01-15
  Due: 2026-12-20

WI-003  Complete missing model cards (3)
  Addresses: MAP 1.1
  Owner: Model owners (named per model)
  Estimate: 1 week per card
  Dependencies: WI-002 (cards reference the data policy)
  Due: 2027-01-31

WI-004  Impact assessment for low/med-risk models
  Addresses: MAP 3.4
  Owner: Program owner
  Estimate: 4 weeks
  Dependencies: WI-001
  Due: 2027-02-15

WI-005  Robustness eval on internal endpoints
  Addresses: MEASURE 2.5
  Owner: ML Platform
  Estimate: 6 weeks (new harness work)
  Dependencies: WI-003 (eval scope follows card)
  Due: 2027-03-01

WI-006  Scheduled pen test
  Addresses: MEASURE 2.7
  Owner: Security (external vendor)
  Estimate: 4 weeks scope + 4 weeks engagement
  Dependencies: exercise-01/02 controls in Enforce;
                 WI-001 (risk appetite defines "high")
  Due: 2027-03-15 (report back)

WI-007  Tabletop exercise (AI incident)
  Addresses: MANAGE 2.4
  Owner: Security + Program owner
  Estimate: 2 weeks prep + 1 day exercise
  Dependencies: WI-006 (findings inform scenarios)
  Due: 2027-03-28

WI-008  Annual policy register review
  Addresses: GOVERN 1.1
  Owner: Legal + Program owner
  Estimate: 2 weeks
  Dependencies: none
  Due: 2026-12-15 (easy; do early to buy calendar)
```

### 2e. Evidence-collection ramp-up

The lookback window is what makes compliance readiness hard:
you cannot retroactively collect evidence you weren't storing.
The plan includes an explicit evidence ramp-up:

| Starting date | Minimum retention | Store | Owner |
|---|---|---|---|
| 2026-11-01 | Kubernetes audit (metadata+ on writes) | Immutable object store, 400 d | SRE |
| 2026-11-01 | Kyverno policy reports | Object store, 400 d | Platform |
| 2026-11-01 | MLMD lineage records | DB + nightly export, 400 d | ML Platform |
| 2026-11-01 | SIEM security events | SIEM, 1 y | Security |
| 2026-11-01 | Cloud audit logs (CloudTrail etc.) | Logging bucket w/ object-lock, 400 d | SRE |
| 2026-11-15 | Signed commit log in platform repos | GitHub + archive, 400 d | Program owner |
| 2026-12-01 | Exception register snapshots (nightly) | Git, 400 d | Program owner |

400 days ≥ the 365-day lookback window plus buffer for the
assessment itself; no evidence should rotate while the
assessor is reading it.

### 2f. Internal readiness review (dress rehearsal)

Scheduled **2027-03-15** — six weeks before external
kickoff. Conducted like the real assessment by a team that
did not build the system:

1. A mock-assessor (internal audit or an unrelated senior
   engineer) is given the control-to-evidence map
   (exercise-03 §2b) and the subcategory list; nothing else.
2. For each subcategory, the mock-assessor requests
   evidence through the same channel the real assessor will
   use (not Slack-DM to the engineer who built it).
3. Each failure to produce evidence within 1 business day
   is a finding against *readiness*, not against the control.
4. Findings are logged with the same template as exercise-03
   §2e and triaged: Severity High → fix before external
   kickoff; Severity Medium → fix before mid-engagement;
   Severity Low → document as residual risk.

### 2g. Residual-risk register

The plan does not promise 100% closure. Items deliberately
left open go into the residual-risk register with:

- Subcategory and gap description.
- Risk rating.
- Compensating controls.
- Accepting authority (name of role, not person).
- Review date (no later than the next annual review).
- Expiry if applicable.

Example:

```
RR-001
  Subcategory: AI 600-1 MS-2.3-003 (generative robustness)
  Gap: Adversarial-prompt eval runs weekly, not per-release
  Rating: Medium
  Compensating controls:
    - Rate limiting on public endpoint
    - Content safety filter in gateway
  Accepted by: VP Engineering
  Review: 2027-05 (post-external-assessment)
  Expiry: Must close by 2027-12-31 (per-release eval)
```

### 2h. Communications plan

A one-page plan keyed to audience, owner, and cadence:

| Audience | Message | Owner | Cadence | Channel |
|---|---|---|---|---|
| Executive sponsor | On-track / at-risk status by WI | Program owner | Biweekly | 1-pager |
| Board / risk committee | Posture + residual-risk summary | CISO + Program owner | Quarterly | Slide |
| Engineering org | What's changing, by when, how it affects them | Program owner | Weekly digest | Email + wiki |
| External assessor | Scope, timeline, point of contact | Program owner | Pre-kickoff | Formal letter |
| Legal / Privacy | Policy changes requiring review | Program owner | Per WI | PR + sync |

### 2i. Budget and staffing estimate

Readiness plans with no cost line get re-scoped halfway
through. The plan includes:

- **People-weeks** per work item (summed from §2d).
- **External spend** — pen-test vendor, tabletop
  facilitator, any tooling licenses.
- **Platform cost** — long-term retention of audit sinks
  (object-lock storage × 400 d per source).

The exact numbers are organisational; the discipline is
*having the numbers* so the plan survives a budget review.

## 3. Validation steps

(Design artifact; validation is a readiness check against
rubric plus mechanical tests.)

1. Every in-scope AI RMF subcategory appears in the
   mapping with a status (Ready / Partial / Gap). No blanks.
2. Every Partial or Gap row has a corresponding work item
   in the remediation plan.
3. Every work item has owner, estimate, dependencies,
   acceptance criteria, due date.
4. The dependency DAG has no cycles.
5. Dress-rehearsal date is at least 6 weeks before external
   kickoff.
6. Evidence-collection ramp-up starts ≥ lookback window
   before the external kickoff, with margin.
7. Residual-risk register items each have an accepting
   authority (role) and a review date.
8. Communications plan names an owner and cadence for every
   audience row.
9. Running the mapping through "if the external assessor
   asks for evidence of GOVERN X.Y today, can we produce
   it?" returns Yes for every Ready row and a concrete plan
   for every Partial/Gap row.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Target definition fixes framework, assessment type, lookback window, external date | 10 |
| R2 | Mapping covers every in-scope AI RMF subcategory (and AI 600-1 subcategories for genAI), with status | 15 |
| R3 | Each Partial/Gap row has a matching work item in the remediation plan | 10 |
| R4 | Work items have owner, estimate, acceptance criteria, due date, and explicit dependencies | 10 |
| R5 | Dependency graph sequences policy → process → measurement → rehearsal; no cycles | 10 |
| R6 | Evidence-collection ramp-up begins before the lookback window and uses tamper-evident stores | 15 |
| R7 | Internal readiness review scheduled ≥6 weeks before external kickoff, with a non-builder mock assessor | 10 |
| R8 | Residual-risk register with role-level accepting authority and review dates | 10 |
| R9 | Communications plan addresses exec, engineering, and external assessor | 5 |
| R10 | Budget and staffing estimate present — people-weeks, external spend, platform cost | 5 |

Pass threshold: **80/100**. R2, R5, and R6 are must-haves —
a readiness plan without complete subcategory mapping, without
a sequenced dependency model, or without early evidence
collection will fail its first external assessment regardless
of effort elsewhere.

## 5. Common mistakes

1. **Starting evidence collection when the plan is written.**
   If the lookback is 6 months and collection starts at the
   kickoff, 5 of those 6 months have no evidence. Start the
   ramp-up as the first act of the plan.
2. **Mapping to a framework by title only.** "We cover NIST
   AI RMF" with no subcategory decomposition hides every
   gap. The subcategory IDs are the planning unit.
3. **Flat gap list, no dependencies.** Teams start at the
   top of the list and discover halfway through that half
   of their work needs to be redone because a policy upstream
   wasn't set yet.
4. **Pen test or tabletop before controls are in Enforce.**
   The findings are "configuration gaps the assessor already
   knew about". Spend the budget when the controls are
   actually running.
5. **No dress rehearsal.** The first time anyone tries to
   produce evidence through the formal channel is during
   the external engagement; everything is a fire drill.
6. **Residual risk accepted by the program owner alone.**
   If the risk clears the organisation's appetite, the
   accepting authority is above the program owner. The
   register names the actual role.
7. **Communications plan addresses engineers but not
   executives.** Executives surprised by findings pull the
   plug on the program. Biweekly 1-pager is cheap and
   survives budget reviews.
8. **Ignoring the generative-AI profile for generative
   systems.** AI 600-1 subcategories are mandatory extras,
   not optional. Omitting them produces a predictable
   finding.
9. **"We'll handle retention when we need it."** By the
   time you need it, the audit log from 10 months ago has
   rotated. Retention policy is a day-one artifact.

## 6. References

Primary (official standards and project documentation):

- NIST — AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST AI 100-1 — AI RMF 1.0 —
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf>
- NIST AI 600-1 — Generative AI Profile —
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
- OWASP Machine Learning Security Top 10 (threat framing
  for ML-specific subcategories) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS (adversary-technique reference for
  MEASURE 2.7 and tabletop scenarios) —
  <https://atlas.mitre.org/>

Practitioner reference (programs that publish their
readiness artifacts can be read as examples, not as
authority):

- VeriSwarm Trust Center — practitioner-grade AI-compliance
  artifacts (control summaries, evidence shapes) —
  <https://veriswarm.ai/trust>

Local context:

- Module-level rationale:
  `modules/mod-209-security-compliance/SOLUTION.md`
  (Decision 5 — data classification and lineage drive
  MAP 4.1; Decision 6 — compliance gates in CI feed
  MEASURE 2.5/2.6 evidence).
- Prior labs supplying the pieces:
  - exercise-01 produces the Kubernetes-side evidence for
    MEASURE 2.7 and MANAGE 4.3.
  - exercise-02 produces the zero-trust architecture that
    MEASURE 2.7 pen tests.
  - exercise-03 supplies the control-to-evidence map that
    this readiness plan refines and sequences.
  - exercise-04 defines the program that owns the
    remediation plan's work items.
- Engineer-tier implementation references:
  `engineer-solutions/mod-109 ex-07/08`,
  `engineer-solutions/mod-103 ex-10`,
  `security-solutions/projects` (per module README
  cross-reference).
