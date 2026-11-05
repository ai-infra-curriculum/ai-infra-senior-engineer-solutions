# SOLUTION — Lab 03: A/B Testing Framework for ML Models

Reference solution for
[lab-03-ab-testing.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-206-advanced-mlops/exercises/lab-03-ab-testing.md).

The engineer-tier version is "route 50% of traffic to v2." The
senior-tier version is a framework that answers four questions
before touching traffic: *how do we assign?*, *what do we
measure?*, *when do we stop?*, and *how do we not poison the
test?*

## 1. Solution overview

The reference framework has three layers:

1. **Assignment** — deterministic hashing of a sticky unit
   (user id, session id) to a variant, so the same unit sees
   the same variant across requests.
2. **Metrics plane** — per-request logging of `(experiment_id,
   variant, unit_id, decision, outcome)` to a metrics store.
3. **Decision plane** — a scheduled job that computes the
   per-variant statistics and either (a) reports significance,
   (b) continues, or (c) triggers an early stop. The module-level
   SOLUTION flags that for ML-*quality* routing, multi-armed
   bandits (Thompson sampling / epsilon-greedy) can replace
   fixed splits; classical A/B is still correct for high-stakes
   metrics.

Key senior-tier choices (and the rationale is in
`modules/mod-206-advanced-mlops/SOLUTION.md`, Decision 2):

- **Deterministic sticky assignment** — not per-request RNG.
- **Guardrail metrics** — latency, error rate, cost — alongside
  the win metric, so a "winning" variant that doubles latency
  loses anyway.
- **Pre-registered exit criteria** — a minimum detectable effect
  (MDE), an alpha, and a max duration, decided *before* the test
  starts. Open-ended tests are the most common anti-pattern.
- **Shadow mode as a staging step**, so production traffic sees
  both predictions but only acts on the control; used to catch
  errors and latency regressions before any real exposure.

## 2. Worked answer or implementation

### 2a. Experiment config (pre-registered)

```python
# experiments/driver_ranker_v2.yaml
experiment_id: driver_ranker_v2_vs_v1
owner: ranking-team
unit: user_id
variants:
  control:
    weight: 0.5
    model_alias: champion
  treatment:
    weight: 0.5
    model_alias: challenger
primary_metric:
  name: conversion_rate
  direction: increase
  mde: 0.01              # 1 pp lift to be worth shipping
guardrails:
  - name: p95_latency_ms
    direction: not_worse_than
    threshold: 150
  - name: error_rate
    direction: not_worse_than
    threshold: 0.005
alpha: 0.05
power: 0.8
max_duration_days: 14
start: "2026-11-05T00:00:00Z"
```

Alpha, power, and MDE are the inputs to a frequentist sample-size
calculation for a two-proportion z-test; the sample size is the
*exit sample size*, not a target. Pre-registering it prevents the
"keep looking until it's significant" failure mode (a
well-documented statistical hazard; see e.g.
[NIST/SEMATECH e-Handbook § 7.2.4 — Sample sizes for comparing two proportions](https://www.itl.nist.gov/div898/handbook/prc/section2/prc242.htm)).

### 2b. Deterministic sticky assignment

```python
# assignment.py
import hashlib

def assign(experiment_id: str, unit_id: str, variants: dict[str, float]) -> str:
    key = f"{experiment_id}:{unit_id}".encode()
    bucket = int(hashlib.sha256(key).hexdigest(), 16) % 10_000 / 10_000.0
    cumulative = 0.0
    for name, weight in variants.items():
        cumulative += weight
        if bucket < cumulative:
            return name
    return name  # rounding slack
```

Properties the hash gives us for free:

- **Stickiness** — the same `(experiment, unit)` → same variant.
- **Independence across experiments** — the experiment id is in
  the hash, so experiment A's assignment doesn't correlate with
  experiment B's.
- **No server state** — any replica can assign without a lookup.

### 2c. Router + logging

```python
# router.py
import time
from dataclasses import dataclass

import mlflow.pyfunc

from .assignment import assign
from .metrics_sink import log_event

@dataclass
class Variant:
    name: str
    model: mlflow.pyfunc.PyFuncModel
    weight: float

class ABRouter:
    def __init__(self, experiment_id: str, variants: list[Variant]):
        self.experiment_id = experiment_id
        self.by_name = {v.name: v for v in variants}
        self.weights = {v.name: v.weight for v in variants}

    def predict(self, unit_id: str, features: dict):
        variant = assign(self.experiment_id, unit_id, self.weights)
        start = time.perf_counter()
        try:
            y = self.by_name[variant].model.predict([features])[0]
            error = False
        except Exception:
            y, error = None, True
            raise
        finally:
            log_event({
                "experiment_id": self.experiment_id,
                "variant": variant,
                "unit_id": unit_id,
                "latency_ms": (time.perf_counter() - start) * 1000,
                "error": error,
                "prediction": y,
            })
        return y
```

Three invariants here:

- The logged variant is the one that actually served the
  request, not the one we *intended* to serve.
- Latency and error are captured in `finally`, so even failed
  calls go into the guardrail metrics.
- No per-request RNG — if `log_event` were based on
  `random.random()` call-site assignment, the same user could
  flip variants across pods.

### 2d. Shadow stage (optional pre-rollout)

Before any real split, run the treatment in shadow:

```python
class ShadowRouter:
    def __init__(self, control, shadow):
        self.control, self.shadow = control, shadow

    def predict(self, unit_id, features):
        y = self.control.predict([features])[0]
        try:
            y_shadow = self.shadow.predict([features])[0]
            log_event({"variant": "shadow", "prediction": y_shadow, ...})
        except Exception:
            log_event({"variant": "shadow", "error": True, ...})
        return y
```

Shadow catches crashes and latency regressions with zero user
exposure. Promotion from shadow → A/B is the gate before any
real split.

### 2e. Analysis job

Two-proportion z-test for a binary primary metric; guardrails
checked independently.

```python
# analyze.py
from math import sqrt
from scipy.stats import norm
import pandas as pd

def analyze(events: pd.DataFrame, cfg: dict) -> dict:
    g = events.groupby("variant")
    successes = g["outcome"].sum()
    trials = g["outcome"].count()
    p_ctrl = successes["control"] / trials["control"]
    p_trt  = successes["treatment"] / trials["treatment"]
    p_pool = (successes.sum()) / (trials.sum())
    se = sqrt(p_pool * (1 - p_pool) * (1 / trials["control"] + 1 / trials["treatment"]))
    z = (p_trt - p_ctrl) / se if se > 0 else 0.0
    p_value = 2 * (1 - norm.cdf(abs(z)))

    guardrails = {}
    for gr in cfg["guardrails"]:
        if gr["name"] == "p95_latency_ms":
            p95 = g["latency_ms"].quantile(0.95)
            guardrails[gr["name"]] = {
                "control": float(p95["control"]),
                "treatment": float(p95["treatment"]),
                "violated": p95["treatment"] > gr["threshold"],
            }
        if gr["name"] == "error_rate":
            er = g["error"].mean()
            guardrails[gr["name"]] = {
                "control": float(er["control"]),
                "treatment": float(er["treatment"]),
                "violated": er["treatment"] > gr["threshold"],
            }

    return {
        "lift_abs": float(p_trt - p_ctrl),
        "p_value": float(p_value),
        "significant": p_value < cfg["alpha"],
        "guardrails": guardrails,
    }
```

The decision rule is **not** "significant → ship." It is:

```
significant and lift ≥ MDE and no guardrail violation → recommend promote
not significant and sample ≥ planned N → recommend keep control
any guardrail violation → recommend kill
otherwise → keep running until planned N or max_duration
```

### 2f. (Optional) Multi-armed bandit mode

For ML-quality routing with low per-decision stakes, swap fixed
weights for Thompson sampling over a Beta prior on the per-arm
success rate — the module-level SOLUTION calls this out
explicitly as the appropriate choice for routing, *not* for
revenue-impacting A/B tests
(`modules/mod-206-advanced-mlops/SOLUTION.md`, Decision 2).

```python
import numpy as np

def thompson_pick(successes: dict[str, int], trials: dict[str, int]) -> str:
    draws = {
        arm: np.random.beta(1 + successes[arm],
                            1 + trials[arm] - successes[arm])
        for arm in successes
    }
    return max(draws, key=draws.get)
```

Bandits still log `(experiment_id, variant, unit_id, outcome)`
events — the metrics plane does not change; only the assignment
plane does.

## 3. Validation steps

1. `python -m assignment_test` — for 10k synthetic ids, variant
   proportions match configured weights within ±1%, and the same
   id returns the same variant on every call.
2. Route traffic via `ABRouter` to two models that are *identical
   copies*; after N requests, the analysis job returns
   `significant: false` (A/A test — the sanity check that the
   framework doesn't fabricate signal).
3. Introduce a deliberate +2 pp lift on treatment; the analysis
   job returns `significant: true` with a lift close to 2 pp.
4. Introduce a deliberate 2× latency regression on treatment;
   the analysis job flags `guardrails.p95_latency_ms.violated=true`
   regardless of primary-metric outcome.
5. In shadow mode, zero user-facing predictions come from the
   shadow model; shadow error rate still gets logged.
6. Stopping the experiment at `max_duration_days` with
   insignificant result produces a `keep control` recommendation,
   not an indefinite continuation.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Experiment config is pre-registered: owner, unit, variants with weights, primary metric with direction and MDE, guardrails, alpha, max duration | 15 |
| R2 | Assignment is deterministic on `(experiment_id, unit_id)` — no per-request RNG | 15 |
| R3 | Assignment is stable when the service scales horizontally (no server-side state) | 10 |
| R4 | Every served request logs the variant that actually served it (captured in `finally`) | 10 |
| R5 | Guardrail metrics (latency, error rate, cost) are logged and checked independently of the primary metric | 15 |
| R6 | Analysis job is a scheduled batch, not computed inside the hot path | 5 |
| R7 | Decision rule is encoded (significant + MDE + no guardrail violation) and does not reduce to "significant → ship" | 15 |
| R8 | Shadow mode exists as a pre-split stage with zero user exposure | 10 |
| R9 | If bandits are used, they are scoped to ML-quality routing; high-stakes revenue tests use fixed splits | 5 |

Pass threshold: **80/100**. R2, R5, and R7 are must-haves.

## 5. Common mistakes

1. **Running until significant.** The sample-size calculation
   exists to prevent exactly this. Enforce `max_duration_days`.
2. **Non-sticky assignment.** Per-request `random.random()` or
   per-pod state → the same user bounces between variants,
   making the "variant" column meaningless and inflating variance.
3. **Logging the *intended* variant, not the served one.** When
   a fallback kicks in and treatment silently serves control,
   the events still say "treatment." Capture after the call.
4. **No guardrails.** A model that doubles latency but nudges
   the primary metric looks like a win until oncall gets paged.
5. **Treating the A/B framework as the launch gate.** The gate
   is the *decision rule*; the framework is the measurement.
   Reviewers must see both artifacts.
6. **Bandits for revenue decisions.** The module SOLUTION
   explicitly rules this out. Bandits trade statistical rigor
   for faster convergence, which is wrong for decisions that
   need defensible significance.
7. **No A/A sanity test.** Without one, nobody has evidence that
   the pipeline doesn't manufacture significance from noise.

## 6. References

Primary (official standards and documentation):

- NIST/SEMATECH e-Handbook of Statistical Methods —
  Comparing two proportions / sample-size calculations —
  <https://www.itl.nist.gov/div898/handbook/prc/section2/prc242.htm>
- NIST AI Risk Management Framework — MEASURE 2.3 (empirical
  evaluation of deployed systems covers controlled rollouts) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- SciPy `scipy.stats.norm` — <https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.norm.html>

Security context:

- MITRE ATLAS — ML attack patterns include manipulation of
  evaluation signal; a tamper-resistant metrics sink matters —
  <https://atlas.mitre.org/>

Practitioner implementation examples (illustrative only, not the
source of truth):

- VeriSwarm model-routing patterns — treat as implementation
  examples, not a standard. Follow the pre-registration and
  guardrail rules even if the practitioner reference omits them.

Local context:

- Module rationale: `modules/mod-206-advanced-mlops/SOLUTION.md`
  (Decision 2 — "Multi-armed bandit instead of static A/B
  splits" + the carve-out that high-stakes tests stay classical).
- Serving / rollback consumer: [exercise-04](../exercise-04/SOLUTION.md).
- Model source: [exercise-02](../exercise-02/SOLUTION.md) (aliases
  `@champion`, `@challenger`).
