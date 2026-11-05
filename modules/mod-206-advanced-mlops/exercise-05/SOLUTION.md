# SOLUTION — Lab 05: MLOps Program Design at Senior Scale

Reference solution for
[lab-05-mlops-program-design.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-206-advanced-mlops/exercises/lab-05-mlops-program-design.md).

This is a **design exercise**. The deliverable is not code; it
is a program plan — the shape of the platform, the roles, the
SLAs, the governance, and the sequenced path a senior engineer
would walk a team through to go from "we have a model in
production" to "we run model lifecycle as a program." Per the
AICG design-exercise contract, the sections below cover a
worked example, decision rationale, and grading rubric.

## 1. Solution overview

A senior-tier MLOps program has four load-bearing pillars, each
corresponding to one of the earlier labs in this module plus one
governance pillar that spans them all:

| Pillar | Contract | Reference lab |
|---|---|---|
| Features | Feature-store governance, freshness SLAs, train-serve-skew tests | [exercise-01](../exercise-01/SOLUTION.md) |
| Experiments & registry | Lineage-complete runs, alias-based promotion, signed models | [exercise-02](../exercise-02/SOLUTION.md) |
| Experimentation | A/B framework with pre-registered exit criteria and guardrails | [exercise-03](../exercise-03/SOLUTION.md) |
| Progressive delivery | Automated quality-gated canaries with rollback | [exercise-04](../exercise-04/SOLUTION.md) |
| Governance (spans all four) | NIST AI RMF alignment, OWASP ML Top 10 controls, MITRE ATLAS threat model | — |

The design artifact the student produces is a **program plan**
that names:

- the target state for each pillar,
- the interfaces between pillars (how features feed training,
  how tracking feeds registry, how registry feeds deployment),
- the SLAs and ownership for each pillar,
- the governance mapping to NIST AI RMF functions (GOVERN, MAP,
  MEASURE, MANAGE),
- a staged rollout plan with clear exit criteria per stage.

This document is the reference against which student submissions
are graded.

## 2. Implementation — worked answer (reference program design)

### 2a. Target architecture (one picture)

```
┌───────────────────────────────────────────────────────────────────┐
│                       Governance layer                            │
│   NIST AI RMF alignment · OWASP ML Top 10 controls · MITRE ATLAS   │
│   Model inventory · Risk register · Audit trail                    │
└───────────────────────────────────────────────────────────────────┘
          │                 │                    │                 │
          ▼                 ▼                    ▼                 ▼
┌──────────────┐   ┌──────────────┐    ┌──────────────┐   ┌──────────────┐
│   Feature    │──▶│  Experiments │──▶ │Experimentation│──▶│ Progressive  │
│    store     │   │  & registry   │    │  framework    │   │   delivery   │
│   (Feast)    │   │   (MLflow)    │    │  (A/B + shadow│   │ (Argo Rollout│
│              │   │               │    │  + bandits)   │   │ + quality    │
│              │   │               │    │               │   │  gate)       │
└──────────────┘   └──────────────┘    └──────────────┘   └──────────────┘
```

Each box corresponds to one pillar; the arrows are the enforced
interfaces (see §2b).

### 2b. Interfaces between pillars (the contracts)

1. **Features → Training.** Training jobs read via
   `FeatureStore.get_historical_features(...)` using a named
   `FeatureService`. Direct SQL against raw tables is a lint
   failure.
2. **Training → Registry.** `mlflow.log_model` with a signature
   and lineage tags is the only sanctioned way to produce a
   registered model.
3. **Registry → Serving.** Serving pods resolve `models:/<name>@champion`
   and `models:/<name>@challenger` at startup; they never pin
   version numbers in configuration.
4. **Serving ↔ Experimentation.** The A/B framework assigns
   `variant` based on a sticky unit hash; the serving pod logs
   `(experiment_id, variant, unit_id, latency_ms, error,
   prediction)` to the metrics plane.
5. **Metrics plane → Progressive delivery.** Argo Rollouts'
   `AnalysisTemplate` reads from the same Prometheus series that
   the A/B framework and the production serving path emit.
6. **Progressive delivery → Registry.** A successful rollout
   triggers the alias flip on the MLflow registry.
7. **Governance → Everything.** Every model in the registry has
   a row in the model inventory keyed by name; every row names
   an owner, an intended-use statement, a risk tier, and a
   pointer to the NIST AI RMF MAP/MEASURE evidence.

### 2c. SLAs and ownership (reference numbers)

The numbers below are the reference values the grader uses; a
student can pick different numbers if they are justified by the
business context the student *writes down*.

| Area | SLA | Owner |
|---|---|---|
| Feature freshness (near-real-time features) | Materialization lag ≤ feature-view TTL (e.g., 2 h) | Feature team |
| Feature freshness (batch features) | Daily materialization completes by 02:00 local | Feature team |
| Train/serve-skew test | Runs every PR touching `feature_repo/`; green required to merge | Feature team + ML platform |
| Registry availability | 99.9% (p=lookup, p=write) | ML platform |
| Experiment A/B framework | p99 assignment latency ≤ 1 ms added at serving path | ML platform |
| Progressive-delivery quality gate | Aborts within one analysis interval on breach | ML platform |
| Model inventory accuracy | 100% of deployed models represented; audit weekly | Governance / ML lead |

### 2d. Staged rollout plan

The reference plan assumes a team that already has "model in
production" (engineer tier) and needs to move to senior tier.

- **Stage 0 — Assessment (Week 1).**
  - Inventory current models: name, owner, runtime, how it was
    last deployed, where its training data came from. Any model
    whose owner cannot answer these three in one sentence is
    added to the risk register.
  - Map current practice to NIST AI RMF GOVERN function:
    policies, roles, accountability. Record gaps.
- **Stage 1 — Feature store (Weeks 2–4).**
  - Stand up Feast against existing feature sources
    (exercise-01). No new ML work; this stage pays off train-
    serve skew for *existing* models.
  - Exit criteria: every production model's features are
    expressible as a `FeatureService`; CI runs the parity test.
- **Stage 2 — Lineage-complete registry (Weeks 3–5, overlaps).**
  - Migrate experiment tracking to a server-backed MLflow
    (exercise-02). Enforce lineage tags via a CI check.
  - Exit criteria: no training job merges without lineage tags
    and a signature.
- **Stage 3 — Progressive delivery + quality gate (Weeks 5–8).**
  - Introduce Argo Rollouts and a Prometheus-backed analysis
    template (exercise-04) for the lowest-risk model.
  - Expand to the rest of the fleet one model per week; each
    migration is itself gated by a shadow run.
  - Exit criteria: all production models deploy via Rollouts;
    champion alias flip is automated.
- **Stage 4 — Experimentation framework (Weeks 7–10, overlaps).**
  - Introduce the A/B framework (exercise-03). Start with
    shadow mode for every candidate before any live split.
  - Exit criteria: every new model release has a pre-registered
    experiment config or an explicit "no A/B" waiver.
- **Stage 5 — Governance hardening (continuous after Week 4).**
  - Model inventory, risk register, and audit trail live from
    Stage 1 onward and are reviewed weekly.
  - Formal mapping to NIST AI RMF (§2e) completed by Week 10.

### 2e. Governance mapping (NIST AI RMF alignment)

The four NIST AI RMF functions
([NIST AI RMF 1.0](https://www.nist.gov/itl/ai-risk-management-framework))
map to concrete artifacts in the program:

- **GOVERN** — the model inventory, ownership matrix, and this
  program document. GOVERN 1 (policies and procedures) is
  satisfied by having the staged rollout plan as a living
  document.
- **MAP** — the per-model intended-use statement and risk tier.
  MAP 3 (data sources documented) is satisfied by the feature
  store contract (exercise-01) and the dataset lineage in
  MLflow (exercise-02).
- **MEASURE** — the A/B framework (exercise-03) and the
  quality-gate analysis in the canary (exercise-04). MEASURE 2
  (empirical evaluation) is served by both.
- **MANAGE** — the automated rollback path (exercise-04),
  manual override runbook, and incident response. MANAGE 2
  (risk treatment) is served by progressive delivery.

### 2f. Security mapping (OWASP ML Top 10 + MITRE ATLAS)

For each OWASP ML Top 10 category
([OWASP ML Top 10](https://owasp.org/www-project-machine-learning-security-top-10/)),
the program names the control pillar that mitigates it. (Below
are the dominant alignments; many OWASP categories are
multi-pillar.)

| OWASP ML risk | Primary control | Pillar |
|---|---|---|
| ML01 Input Manipulation / Adversarial | Guardrail metrics; anomaly detection on prediction distribution | Experimentation + Delivery |
| ML02 Data Poisoning | Feature-store write-path governance; dataset hash in run lineage | Features + Registry |
| ML03 Model Inversion | Access control on registry; serving-side rate limits | Registry + Serving |
| ML04 Membership Inference | Same as ML03 | Registry + Serving |
| ML05 Model Theft | Signed artifacts at load time; registry audit log | Registry |
| ML06 AI Supply Chain Attacks | Model signatures; alias flip gated by rollout success | Registry + Delivery |
| ML07 Transfer-Learning Attack | Lineage: base model + fine-tune dataset tagged on every run | Registry |
| ML08 Model Skewing | Train-serve parity test; drift monitoring | Features + Delivery |
| ML09 Output Integrity | Quality gate on canary; rollback on regression | Delivery |
| ML10 Model Poisoning | Lineage; signed artifacts; auditable registry | Registry |

MITRE ATLAS ([https://atlas.mitre.org/](https://atlas.mitre.org/))
provides the adversary-perspective complement: the program's
threat model should name which ATLAS techniques (e.g., ML Model
Access, ML Supply Chain Compromise, Backdoor ML Model) the
controls above actually detect or block, and which are accepted
residual risks.

### 2g. The one-paragraph escalator pitch

> We're moving from "we have models in production" to "we run
> model lifecycle as a program." Four pillars — features,
> registry, experimentation, delivery — connected by enforced
> interfaces, with a governance layer that maps every control
> to NIST AI RMF and OWASP ML Top 10. We land it in 10 weeks,
> starting with the feature store because that buys down the
> largest silent risk (train-serve skew) first.

A senior engineer should be able to deliver that pitch to an
exec audience and defend each sentence to their team.

## 3. Validation steps

A submission is validated against the reference by checking
that the student deliverable:

1. Names all four pillars and a governance layer, and places
   them on a diagram with explicit interfaces (§2a, §2b).
2. Specifies at least one SLA per pillar with a named owner
   (§2c).
3. Lays out a staged rollout plan with exit criteria per stage
   (§2d).
4. Maps the program to the four NIST AI RMF functions with at
   least one concrete artifact per function (§2e).
5. Maps the program to at least eight of the ten OWASP ML Top
   10 categories, naming the responsible pillar (§2f).
6. Includes either an executive pitch or an equivalent
   one-pager (§2g) — i.e., does not require a reader to walk
   the whole document to understand the approach.

Validation is a design review, not a test run. A successful
submission reads like a program plan a senior engineer could
hand to a director of ML.

## 4. Rubric / review checklist

Weightings sum to 100. A submission below 70 needs a rewrite;
70–84 is pass-with-comments; 85+ is ship.

| # | Criterion | Weight |
|---|---|---|
| R1 | Four pillars + governance layer present and correctly scoped | 10 |
| R2 | Pillar interfaces are explicit (not implied by a picture alone) and each interface has a contract sentence | 15 |
| R3 | Each pillar has at least one SLA with a named owner | 10 |
| R4 | Staged rollout plan has concrete exit criteria per stage, not just durations | 15 |
| R5 | NIST AI RMF mapping covers all four functions (GOVERN / MAP / MEASURE / MANAGE) with a concrete artifact each | 10 |
| R6 | OWASP ML Top 10 mapping covers ≥ 8 of 10 categories and names the responsible pillar | 10 |
| R7 | MITRE ATLAS threat model referenced; at least three techniques named with the corresponding control | 5 |
| R8 | Reference acknowledges the module's trade-offs (Feast vs managed; bandits vs classical A/B — see `../SOLUTION.md`) | 5 |
| R9 | Manual override / human escape hatch is explicitly called out, not just "automated everything" | 5 |
| R10 | Proposal is defensible to a non-ML executive — one-pager / escalator pitch included | 5 |
| R11 | No invented facts, metrics, or incidents — claims are tied to official standards or to clearly labeled local context | 10 |

R11 is a must-have. A submission containing fabricated incident
statistics or vendor benchmark numbers fails regardless of
other scores.

## 5. Common mistakes

1. **Program as a toolchain list.** "We use Feast, MLflow, Argo
   Rollouts" is a shopping list, not a program. The program is
   the contracts between them and the ownership model.
2. **Governance bolted on at the end.** Governance is the layer
   that spans all four pillars; it must be in the design from
   Stage 0, not a Stage-5 appendix.
3. **No exit criteria.** Stages with durations but no exit
   criteria become indefinite. Senior-tier plans always name
   "we move on when X is true."
4. **No manual override in delivery.** The module-level
   SOLUTION flags this as a common mistake — automated rollback
   must coexist with a human lever.
5. **NIST AI RMF cited but not mapped.** Writing "we follow NIST
   AI RMF" without naming the artifact that satisfies each
   function is theatre. Mapping must be concrete.
6. **Bandits applied to revenue-impacting experiments.** The
   module SOLUTION explicitly rules this out; students who
   propose bandits for billing-adjacent models have missed the
   carve-out.
7. **Over-scoping for the team.** A 20-engineer program plan
   handed to a 3-engineer team will not land. Senior-tier
   design acknowledges capacity. A plan that assumes infinite
   headcount is not a plan.
8. **Fabricated statistics.** Any specific percentage uplift,
   incident count, or vendor benchmark that lacks a source is
   an R11 failure.

## 6. References

Primary (official standards and project documentation):

- NIST AI Risk Management Framework (AI RMF 1.0) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — <https://atlas.mitre.org/>
- Feast documentation — <https://docs.feast.dev/>
- MLflow documentation — <https://mlflow.org/docs/latest/>
- Argo Rollouts documentation — <https://argoproj.github.io/argo-rollouts/>

Local context (the four pillars in detail):

- Features — [exercise-01](../exercise-01/SOLUTION.md).
- Experiments & registry — [exercise-02](../exercise-02/SOLUTION.md).
- Experimentation framework — [exercise-03](../exercise-03/SOLUTION.md).
- Progressive delivery — [exercise-04](../exercise-04/SOLUTION.md).
- Module rationale — `modules/mod-206-advanced-mlops/SOLUTION.md`.
- Engineer-tier implementations referenced throughout —
  `engineer-solutions/mod-106` (per module README cross-reference).
