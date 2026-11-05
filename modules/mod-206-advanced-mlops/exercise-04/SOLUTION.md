# SOLUTION — Lab 04: Canary Deployment with Automated Rollback

Reference solution for
[lab-04-canary-deployment.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-206-advanced-mlops/exercises/lab-04-canary-deployment.md).

Engineer-tier canary: "shift 10% traffic, wait, then 100%."
Senior-tier canary: progressive delivery driven by a **quality
gate on real metrics**, with automated rollback and an explicit
manual-override escape hatch. The rationale for the automation
sits in `modules/mod-206-advanced-mlops/SOLUTION.md` (Decision 4
— "Automated rollback on quality regression").

## 1. Solution overview

The reference uses Kubernetes + Argo Rollouts
([Argo Rollouts docs](https://argoproj.github.io/argo-rollouts/))
for progressive delivery, with a Prometheus-backed **analysis
template** that reads the same metrics logged by the A/B
framework in [exercise-03](../exercise-03/SOLUTION.md).

- **Progressive steps** — e.g. 10% → 25% → 50% → 100%, with a
  pause at each step for the analysis window.
- **Quality gate** — per step, the challenger must satisfy:
  - error rate ≤ control × 1.1
  - p95 latency ≤ control × 1.1
  - model-quality metric (e.g. AUC on labeled sample) ≥ control
    threshold
- **Automated rollback** — Argo Rollouts aborts the rollout and
  restores the stable `ReplicaSet` on analysis failure
  ([Argo Rollouts — Analysis & Progressive Delivery](https://argoproj.github.io/argo-rollouts/features/analysis/)).
- **Manual override** — `kubectl argo rollouts promote --full`
  and `kubectl argo rollouts abort` are retained as human
  escape hatches; alert when either fires.
- **Model source** — the pod loads `models:/<name>@challenger`
  from the MLflow registry on startup, so the "deploy a new
  model" event is "update the alias and ship a Rollout." See
  [exercise-02 §2d](../exercise-02/SOLUTION.md).

## 2. Worked answer or implementation

### 2a. Rollout manifest

```yaml
# rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: driver-ranker
spec:
  replicas: 10
  selector:
    matchLabels:
      app: driver-ranker
  template:
    metadata:
      labels:
        app: driver-ranker
    spec:
      containers:
        - name: serving
          image: registry.example.com/driver-ranker:2026.11.04
          env:
            - name: MLFLOW_TRACKING_URI
              value: http://mlflow:5000
            - name: MODEL_URI
              value: "models:/driver_ranker@challenger"
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            periodSeconds: 5
  strategy:
    canary:
      canaryService: driver-ranker-canary
      stableService: driver-ranker-stable
      trafficRouting:
        istio:
          virtualService:
            name: driver-ranker
            routes: [primary]
      steps:
        - setWeight: 10
        - pause: { duration: 10m }
        - analysis:
            templates:
              - templateName: quality-gate
            args:
              - name: service
                value: driver-ranker
        - setWeight: 25
        - pause: { duration: 10m }
        - analysis: { templates: [{ templateName: quality-gate }],
                      args: [{ name: service, value: driver-ranker }] }
        - setWeight: 50
        - pause: { duration: 15m }
        - analysis: { templates: [{ templateName: quality-gate }],
                      args: [{ name: service, value: driver-ranker }] }
        - setWeight: 100
```

Progressive canary with inline analysis is the documented
pattern ([Argo Rollouts — Canary strategy](https://argoproj.github.io/argo-rollouts/features/canary/)).
We pair it with the `AnalysisTemplate` below.

### 2b. Analysis template (quality gate)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: quality-gate
spec:
  args:
    - name: service
  metrics:
    - name: error-rate
      interval: 1m
      count: 10                    # 10 samples over the pause window
      failureLimit: 2              # abort after 2 failed samples
      successCondition: result[0] <= result[1] * 1.10
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            (
              sum(rate(http_requests_total{service="{{args.service}}",
                                          variant="canary",status=~"5.."}[1m]))
              /
              sum(rate(http_requests_total{service="{{args.service}}",
                                          variant="canary"}[1m]))
            ) or vector(0)
            and on()
            (
              sum(rate(http_requests_total{service="{{args.service}}",
                                          variant="stable",status=~"5.."}[1m]))
              /
              sum(rate(http_requests_total{service="{{args.service}}",
                                          variant="stable"}[1m]))
            ) or vector(0)
    - name: latency-p95
      interval: 1m
      count: 10
      failureLimit: 2
      successCondition: result[0] <= result[1] * 1.10
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket{
                service="{{args.service}}",variant="canary"}[1m])) by (le))
            and on()
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket{
                service="{{args.service}}",variant="stable"}[1m])) by (le))
    - name: model-quality
      interval: 5m
      count: 2                     # labeled sample is slow; fewer, larger
      failureLimit: 1
      successCondition: result >= 0.78
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            avg_over_time(model_auc{service="{{args.service}}",
                                    variant="canary"}[5m])
```

The `AnalysisTemplate` + `provider.prometheus` pattern is from
the Argo Rollouts docs
([Argo Rollouts — Prometheus analysis](https://argoproj.github.io/argo-rollouts/features/analysis/#prometheus)).
Every metric has a `failureLimit` — a single flake does not
abort the rollout, but a trend does.

### 2c. Metrics source

The serving pod exposes `/metrics` via a Prometheus client, and
every request logs `variant={canary|stable}`:

```python
from prometheus_client import Counter, Histogram, Gauge

REQS = Counter("http_requests_total", "", ["service", "variant", "status"])
LAT  = Histogram("http_request_duration_seconds", "",
                 ["service", "variant"])
AUC  = Gauge("model_auc", "Rolling AUC on labeled sample",
             ["service", "variant"])
```

`variant` is derived from a pod env var set by the canary and
stable Deployments created by the Rollout (`argoproj.io/rollout`
labels). The labeled-sample AUC is written by a sidecar that
scores recent labeled events; without a labeled-sample stream,
`model_auc` falls back to a proxy metric (e.g., confidence
calibration drift), and that substitution must be documented in
the manifest.

### 2d. Manual override

```bash
# Promote stuck-but-healthy rollout
kubectl argo rollouts promote driver-ranker

# Full promote (skip remaining steps)
kubectl argo rollouts promote driver-ranker --full

# Abort and roll back now
kubectl argo rollouts abort driver-ranker
kubectl argo rollouts undo driver-ranker
```

Alert on either command firing — the humans-ran-the-override
path must be auditable.

### 2e. Registry alias flip on success

After Rollout reaches 100% and stays healthy for the
`stable-grace-period`, a post-promotion job flips the champion
alias:

```python
from mlflow.tracking import MlflowClient
client = MlflowClient()

def flip_champion(model: str, challenger_version: int):
    client.set_registered_model_alias(model, "champion", challenger_version)
    client.delete_registered_model_alias(model, "challenger")
```

This closes the loop with [exercise-02](../exercise-02/SOLUTION.md):
"deployment stable" means "the registry now reflects it."

## 3. Validation steps

1. `kubectl argo rollouts get rollout driver-ranker` — shows
   the step sequence and current weight.
2. Deploy a known-good challenger — rollout progresses through
   all steps and reaches 100% without aborting.
3. Deploy a challenger that returns HTTP 500 on 5% of requests
   — rollout aborts at the first analysis step; stable retains
   100% of traffic; `kubectl rollout status` reports Degraded.
4. Deploy a challenger with doubled p95 latency — rollout aborts
   on the latency metric even if error rate is clean.
5. Rollout aborted → `models:/driver_ranker@champion` is
   unchanged in MLflow; no post-promotion job ran.
6. `kubectl argo rollouts promote --full` on a paused rollout
   completes in seconds; the override is logged (audit webhook /
   Rollout event stream).
7. Deliberately deploy with `successCondition` so permissive
   that any drift passes — code review should flag this as an
   R4 violation (see rubric).

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Progressive steps with pauses; no single-step 0→100 cutover | 10 |
| R2 | Analysis runs at every step, not only at the final step | 10 |
| R3 | Analysis includes error rate, latency, and a model-quality signal — not latency alone | 15 |
| R4 | Success conditions compare *canary to stable*, not to a static threshold (so a slow-moving regression doesn't silently pass) | 15 |
| R5 | `failureLimit` is > 0 and tuned — single flakes don't abort, trends do | 5 |
| R6 | `variant` label is set from pod identity (Rollout label), not from a hardcoded env | 5 |
| R7 | Manual promote / abort commands exist, are documented, and firing them raises an alert | 10 |
| R8 | On success, the MLflow champion alias is flipped automatically | 10 |
| R9 | On failure, the rollout aborts within one analysis interval and stable traffic is preserved | 15 |
| R10 | Runbook names what happens if the model-quality signal stalls (e.g., labeled stream outage) — the rollout must not silently pass | 5 |

Pass threshold: **80/100**. R3, R4, and R9 are must-haves — a
rollout that aborts only on HTTP 5xx is not a quality gate.

## 5. Common mistakes

1. **Quality gate on latency alone.** A worse model with
   unchanged latency sails through. Model-quality signal must
   be part of the gate — or the gate is a deployment ritual,
   not a safeguard.
2. **Static thresholds instead of canary-vs-stable comparison.**
   The whole fleet slowly degrades and "passes" because the
   absolute number is still below the threshold. Compare to
   stable.
3. **No `failureLimit`.** A single noisy minute aborts every
   rollout; team learns to raise thresholds; gate is now toothless.
4. **No manual escape hatch.** When the rollback target is also
   broken (yes, it happens), a human needs a lever. Keep
   `promote --full` and `abort` documented.
5. **Alias flip on `Deployment` success instead of `Rollout`
   stable.** The alias then reflects a model whose steps all
   passed individually but which never finished progressive
   delivery.
6. **Model-quality metric with no uptime guarantee.** If the
   labeled-sample stream is down, `model_auc` goes to `NaN` and
   the `successCondition` evaluates to false-or-true depending
   on Prometheus semantics; write the query so stale data is a
   *failure*, not a *pass*.
7. **Wiring Rollouts at the Deployment layer only.** Without
   `trafficRouting` (Istio / NGINX / SMI), Rollouts shift *pod
   counts*, which is not the same as *traffic split* for
   long-lived connections. Use an explicit traffic router.

## 6. References

Primary (official project documentation):

- Argo Rollouts documentation — <https://argoproj.github.io/argo-rollouts/>
- Argo Rollouts — Canary strategy — <https://argoproj.github.io/argo-rollouts/features/canary/>
- Argo Rollouts — Analysis & Progressive Delivery — <https://argoproj.github.io/argo-rollouts/features/analysis/>
- Argo Rollouts — Prometheus metric provider — <https://argoproj.github.io/argo-rollouts/features/analysis/#prometheus>
- Argo Rollouts — Traffic management (Istio) — <https://argoproj.github.io/argo-rollouts/features/traffic-management/>
- MLflow Model Registry aliases — <https://mlflow.org/docs/latest/model-registry.html#deploy-and-organize-models-with-aliases-and-tags>

Security and governance context:

- OWASP Machine Learning Security Top 10 — ML06 "AI Supply
  Chain Attacks" motivates signing + automated rollback on
  anomalous behavior post-deploy —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- NIST AI Risk Management Framework — MANAGE 2.3 covers safe
  deployment practices including rollback —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS — ML model manipulation techniques motivate
  canary-vs-stable comparison as a tamper-detection signal —
  <https://atlas.mitre.org/>

Local context:

- Module rationale: `modules/mod-206-advanced-mlops/SOLUTION.md`
  (Decision 4 — "Automated rollback on quality regression").
- Producer of `@challenger`: [exercise-02](../exercise-02/SOLUTION.md).
- Metrics plane consumer: [exercise-03](../exercise-03/SOLUTION.md).
- Project 202 (`projects/project-202-model-serving`) covers the
  serving-platform implementation that these Rollouts deploy
  into.
