# exercise-05 — Lab 05: Production Optimization Program

Reference solution for the learning lab
`lessons/mod-204-model-optimization/exercises/lab-05-production-optimization-program.md`.

## 1. Solution overview

This is a **design** exercise, not a code exercise. The learner must
design a *program* — the ongoing organizational practice of optimizing
models in production — rather than a single optimized artifact.

A credible program answers five questions:

1. **Who owns it?** Which team runs the optimization work, and who
   signs off on deploying an optimized model.
2. **What triggers optimization?** Cost pressure, latency SLO breach,
   new hardware, new model release.
3. **What is the pipeline?** The repeatable steps from "candidate
   model" to "shipped engine" that include quality gates.
4. **How is risk managed?** What happens when a quantized model
   regresses in production — rollback, shadow eval, canary.
5. **How is it measured?** Program-level KPIs (cost per request,
   accuracy regressions caught, lead time) that show the program is
   working.

The reference deliverable is a **program design document** that
answers these, plus a **worked example** applied to one representative
model, plus a **rubric** the grader uses.

## 2. Worked implementation (program design)

### 2.1 Ownership and RACI

- **Responsible:** ML Infrastructure team (owns the optimization
  pipeline and tooling).
- **Accountable:** Applied ML lead (owns the accuracy gate).
- **Consulted:** Product (owns the latency SLO), Security / Risk
  (owns the regulatory gates — see NIST AI RMF alignment below).
- **Informed:** On-call SRE, Finance (cost pressure signal).

### 2.2 Triggers

A program should state explicitly what causes an optimization to
start. The reference uses four:

| Trigger            | Signal                                   | Owner       |
|--------------------|------------------------------------------|-------------|
| Cost breach        | Monthly inference spend over budget      | Finance     |
| Latency SLO breach | p95 TTFT over target for 7 consecutive days | Product |
| New hardware       | GA of new GPU SKU with meaningful perf delta | Infra  |
| Model refresh      | New foundation model release of equivalent capability | Applied ML |

Optimizations started "because INT4 is cool" are a smell; the program
requires a trigger of record.

### 2.3 Pipeline

```
   ┌────────────┐   ┌────────────┐   ┌─────────────┐   ┌──────────┐
   │ Candidate  │ → │  Offline   │ → │  Shadow     │ → │  Canary  │ → Full rollout
   │ (quant / …) │   │   eval     │   │  (mirrored  │   │  (1 %)   │
   └────────────┘   └────────────┘   │   traffic)   │   └──────────┘
                                     └─────────────┘
```

Each stage has an explicit gate.

- **Offline eval gate:** fixed eval set, accuracy within budget vs
  current production. Operating conditions (hardware, software,
  versions) locked per `exercise-04` reporting rules.
- **Shadow gate:** N days of mirrored traffic; compare distributions
  of outputs, not just an aggregate score. Any new error class blocks
  promotion.
- **Canary gate:** 1 %, then 10 %, then 50 % of traffic. Rollback
  automation on SLO breach must be tested before entering canary.
- **Full rollout:** only after N canary days with no new incident
  classes.

### 2.4 Risk management

Optimization changes the model; by NIST AI RMF (GOVERN / MAP /
MEASURE / MANAGE) terms it is a change event. The program applies:

- **GOVERN:** sign-off from the Applied ML lead on each optimized
  artifact; artifact registered with training provenance and
  calibration-dataset reference.
- **MAP:** model card updated with the new precision ladder and the
  categories of input the optimization was *not* calibrated for.
- **MEASURE:** regression probes run on protected categories defined
  by the product's fairness policy, not only on the primary eval set.
- **MANAGE:** rollback is a single configuration change; pre-canary
  rehearsal is required.

Where the optimization touches a model exposed to adversarial input
(e.g., user prompts can shape what the model sees), the program
cross-references the OWASP ML Top 10 and MITRE ATLAS. For this lab
the relevant threats are:

- OWASP ML03 *Model Inversion* — quantized models can leak training
  data differently; re-run the privacy eval.
- OWASP ML05 *Model Theft* — a smaller / quantized model may be
  easier to exfiltrate and replicate; revisit API rate limits and
  authn.
- MITRE ATLAS *AML.T0024 Evade ML Model* — quantized models may
  respond differently to adversarial examples than the baseline; the
  robustness probe is part of the offline eval gate.

The reference program does not claim the exercise covers every
control in these standards; it names the ones the optimization
pipeline must re-run on every change.

### 2.5 Program KPIs

The program is measured on:

- **Cost per request** (primary cost KPI).
- **Accuracy regressions caught before canary** (effectiveness of
  the eval gate).
- **Incident rate per deploy** (safety of the pipeline).
- **Lead time from trigger to full rollout** (program speed).
- **Fraction of production models on a current precision ladder**
  (coverage).

### 2.6 Worked example — one model through the pipeline

Apply the program to a single representative model (e.g., an 8B
instruct LLM used by a summarization feature):

- **Trigger:** cost breach. Finance signals a 20 % overrun.
- **Candidate:** INT4 AWQ (per `exercise-02` recipe).
- **Offline eval:** task metric on the summarization eval set,
  perplexity on an in-domain corpus, and the privacy probe.
- **Shadow:** 72 hours of mirrored prompts; compare summary length
  distributions, refusal rate, and a sample of 200 outputs rated by
  humans.
- **Canary:** 1 % → 10 % → 50 % → 100 % over a week, with automatic
  rollback on refusal-rate spike or TTFT regression.
- **Post-rollout:** model card and runbook updated; monitors added.

(Numbers for cost reduction, accuracy delta, and lead time are
produced by the learner on their own stack — see the no-fabricated-
numbers policy in `exercise-04`.)

## 3. Validation steps

1. The design document names the owner, the trigger set, the pipeline
   stages, the risk-management hooks, and the KPIs.
2. The worked example walks one model through every stage with a
   stated decision at each gate.
3. The document cross-references the module decision framework
   (`SOLUTION.md`) and the per-exercise implementations
   (`exercise-01` through `exercise-04`).
4. The rubric in §4 is reproduced verbatim in the deliverable so the
   grader can tick each row.
5. No invented metrics, outages, or vendor case studies appear in the
   write-up.

## 4. Rubric / review checklist

| Dimension                           | Weight | Pass condition |
|-------------------------------------|--------|----------------|
| Ownership named (RACI)              | 10%    | Who is R / A / C / I for optimization decisions |
| Explicit triggers                   | 10%    | At least cost, latency, hardware, model refresh |
| Pipeline with gates                 | 20%    | Each stage has an exit criterion and an owner |
| Risk management                     | 20%    | References NIST AI RMF; names rollback mechanism |
| Security framing                    | 10%    | References OWASP ML Top 10 and MITRE ATLAS where the optimization affects an exposed surface |
| Program KPIs                        | 10%    | KPIs are measurable and tied to the triggers |
| Worked example end-to-end           | 15%    | A real-shape model walked through every gate |
| Honesty                             | 5%     | No invented outages, case studies, or metrics |

Automatic fail: a program design that ships INT8 to production
without an accuracy gate; or that names rollback as "page the on-call"
rather than a tested mechanism.

## 5. Common mistakes

1. **Treating optimization as a one-off project.** The exercise asks
   for a *program*. A one-shot plan that ends at "we shipped INT4"
   has no answer for the next trigger.
2. **No rollback rehearsal.** A rollback that has never been tested
   isn't a rollback; it's a hope. The program must rehearse it before
   first canary.
3. **Aggregate accuracy as the only gate.** A quantized model can
   preserve aggregate accuracy and still regress on a subpopulation.
   The program needs sub-slice evals.
4. **Treating security standards as checkbox references.** Dropping
   "aligned with NIST AI RMF" into a doc without naming the specific
   controls the pipeline runs is a smell.
5. **No definition of done.** Without KPIs, the program cannot say
   whether it is working. Define the KPIs first, iterate the pipeline
   to meet them.
6. **Confusing the program with the tooling.** The pipeline and the
   tooling can be excellent; if nobody owns the decision to ship, the
   program still fails.
7. **Fabricated "case studies".** Referencing an incident that did
   not happen or a vendor result that is not cited is grounds for
   rejection; flag the claim as unsourced and leave it to the
   learner's organization to produce real numbers.

## 6. References

Official standards and project documentation:

- NIST AI Risk Management Framework (AI RMF 1.0) — GOVERN / MAP /
  MEASURE / MANAGE functions
  — https://www.nist.gov/itl/ai-risk-management-framework
- OWASP Machine Learning Security Top 10
  — https://owasp.org/www-project-machine-learning-security-top-10/
- MITRE ATLAS (Adversarial Threat Landscape for AI Systems)
  — https://atlas.mitre.org/
- MLPerf Inference benchmarking methodology (used as the honest-
  reporting reference inside the pipeline's eval gate)
  — https://mlcommons.org/en/inference-datacenter/

Local module context:

- `modules/mod-204-model-optimization/SOLUTION.md` — the decision
  framework the program operationalizes.
- `modules/mod-204-model-optimization/exercise-01/SOLUTION.md` —
  TensorRT pipeline; a candidate type the program must support.
- `modules/mod-204-model-optimization/exercise-02/SOLUTION.md` —
  quantization recipe; also a candidate type.
- `modules/mod-204-model-optimization/exercise-03/SOLUTION.md` —
  vLLM serving; the surface the canary stage deploys against.
- `modules/mod-204-model-optimization/exercise-04/SOLUTION.md` —
  the benchmarking harness that produces the numbers the offline
  gate evaluates.
