# SOLUTION — Lab 03: Chaos Engineering Experiment

Reference solution for
[lab-03-chaos-experiment.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-207-observability-sre/exercises/lab-03-chaos-experiment.md).

The engineer-tier form of this exercise is "kill a pod with Chaos
Mesh and see what breaks." The senior-tier responsibility is to
design and run a chaos experiment as a **hypothesis-driven,
SLO-bounded, rolled-back-safely scientific experiment** — the
discipline defined in the Principles of Chaos Engineering
([principlesofchaos.org](https://principlesofchaos.org/)).

## 1. Solution overview

A senior-tier chaos experiment has five elements, all of them
required by the Principles of Chaos
([principlesofchaos.org — Advanced Principles](https://principlesofchaos.org/)):

1. **Steady-state hypothesis** — a measurable statement of
   "healthy" the system must stay in, expressed against SLIs
   already defined in [exercise-02](../exercise-02/SOLUTION.md).
2. **Perturbation** — one specific fault injected in one place
   (pod kill, network latency, DNS failure, GPU driver stall).
3. **Blast radius** — scoped as tightly as possible while still
   being meaningful; always with an abort condition tied to the
   SLO error budget.
4. **Observation** — the dashboards and alerts that would catch
   the perturbation, watched in real time.
5. **Learning + action** — what the hypothesis test produced,
   what the remediation is, and when it will be reverified.

Senior-tier additions beyond "run a chaos tool":

- The experiment lives in a document (reference template in
  §2a) committed alongside the service it tests.
- The abort condition fires *before* the SLO is actually missed —
  the budget guard, not the budget-burned alert.
- Chaos runs on a scheduled cadence (module-level
  [Decision 5](../SOLUTION.md)), not as a one-time stunt.
- The experiment runs first in a staging environment, then in
  production with explicitly reduced blast radius, not the
  other way around.

The reference uses **Chaos Mesh** running in a Kubernetes
cluster ([Chaos Mesh docs](https://chaos-mesh.org/docs/)) and
the observability stack from [exercise-01](../exercise-01/SOLUTION.md).

## 2. Worked answer or implementation

### 2a. Experiment document (reference template)

Every experiment carries a one-page document in Markdown
committed to the service's repo. The reference template:

```markdown
# Experiment: api-pod-kill-staging

## Hypothesis
Under random termination of a single `api` pod every 60s for 15
minutes, the service remains within its 99.9% availability SLO
and its 99% @ 200ms latency SLO over the experiment window.

## Steady state (SLIs)
- `slo:api_availability_error_ratio:5m` ≤ 0.001 (i.e., ≤ 0.1%).
- `slo:api_latency_error_ratio:5m` ≤ 0.01 (i.e., ≤ 1%).

## Perturbation
PodChaos action=pod-kill, mode=one, selector={app=api}, every 60s
for 15m (Chaos Mesh schedule).

## Blast radius
- Scope: one `api` pod of N≥4 replicas; others untouched.
- Scheduler: `kube-scheduler` default; pods are expected to be
  rescheduled by the Deployment controller within the readiness
  probe window.

## Abort conditions
Abort if any of the following is true for ≥ 2m:
- Availability error ratio (5m) > 0.5% (half the burn-rate
  fast page threshold).
- Latency error ratio (5m) > 3%.
- Any `page`-severity alert fires on the api SLOs.

## Observers
- Grafana dashboard: api RED + SLO burn.
- Alertmanager: filter for `service=api, severity=page`.

## Expected result (pre-registered)
Hypothesis holds; dashboards show brief dips during each
reschedule but recovery within readiness-probe + load-balancer
cycle (< 30 s).

## Actual result + learning
<Filled in post-run.>

## Next action
<Filled in post-run.>
```

Pre-registering the expected result is the chaos equivalent of
pre-registering an experiment in science: it rules out post-hoc
rationalization.

### 2b. Chaos Mesh CRD — pod kill

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: api-pod-kill
  namespace: production
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: api
  duration: "60s"
  scheduler:
    cron: "@every 60s"
```

`PodChaos` is one of Chaos Mesh's core fault types; others
include `NetworkChaos`, `IOChaos`, `StressChaos`, `HTTPChaos`,
and `DNSChaos`
([Chaos Mesh docs — Simulate chaos experiments](https://chaos-mesh.org/docs/simulate-pod-chaos-on-kubernetes/)).
`mode: one` picks a single pod per run; `mode: fixed-percent`
with a value of e.g. `25` scales the blast radius once the
experiment has proven safe at `one`.

### 2c. Abort-condition guard

Chaos Mesh does not natively abort on an external Prometheus
query, so the reference pattern pairs the experiment with a
**guard** rule that (a) alerts on the abort condition and (b)
is wired to a small operator-side killswitch — a single kubectl
delete, a Workflow stop, or a watcher Deployment that deletes
the experiment CR on alert.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: chaos-abort-api
  namespace: observability
  labels:
    release: kps
spec:
  groups:
    - name: chaos.abort
      rules:
        - alert: ChaosAbortApiAvailability
          expr: slo:api_availability_error_ratio:5m > 0.005
          for: 2m
          labels:
            severity: page
            service: api
            kind: chaos-abort
          annotations:
            summary: "chaos experiment on api hit abort threshold"
            runbook_url: "https://runbooks.example.com/chaos/abort-api"
        - alert: ChaosAbortApiLatency
          expr: slo:api_latency_error_ratio:5m > 0.03
          for: 2m
          labels:
            severity: page
            service: api
            kind: chaos-abort
          annotations:
            summary: "chaos experiment on api hit abort latency threshold"
            runbook_url: "https://runbooks.example.com/chaos/abort-api"
```

The runbook for a `chaos-abort` alert is a single command:
`kubectl delete podchaos -n production api-pod-kill` (or
`chaosctl recover`, depending on how the experiment was created).
Automating the delete via an Alertmanager webhook that targets
an operator is the next step up — out of scope for a single
lab, in scope for a production chaos program.

### 2d. Running the experiment

```bash
# 1. Baseline in staging.
kubectl apply -f experiment.yaml -n staging
# 2. Observe for duration, confirm hypothesis holds.
kubectl get events -n staging --watch
# 3. If staging holds, schedule the same experiment in production
#    during a declared window, with the abort guard already
#    deployed and on-call paged.
kubectl apply -f experiment.yaml -n production

# To stop on demand:
kubectl delete podchaos -n production api-pod-kill
```

Chaos Mesh also supports Workflows
([Chaos Mesh docs — Workflow](https://chaos-mesh.org/docs/create-chaos-mesh-workflow/))
for chained experiments; the reference uses a single-step
`PodChaos` for clarity but notes the Workflow object is the
right shape once experiments grow past one fault type.

### 2e. Example variations (choose one for the lab)

Pick one perturbation that exercises your service:

- **Pod kill** — tests pod-disruption budget, readiness probes,
  load-balancer draining.
- **Network latency** — `NetworkChaos`, 100 ms delay to a
  dependency; tests client timeouts and retries.
- **DNS failure** — `DNSChaos`, 10% random NXDOMAIN; tests
  resolver caching and backoff.
- **CPU stress** — `StressChaos`, 90% CPU on a pod; tests HPA
  scale-up thresholds.
- **IO latency** — `IOChaos`, 50 ms latency on a volume; tests
  filesystem-bound code paths.

For an AI workload, the two most informative perturbations are
(a) model-server pod kill (tests warm-up and queue behaviour)
and (b) `NetworkChaos` between the model server and the feature
store or vector DB (tests fallback behaviour and timeout
budgets).

### 2f. Post-experiment write-up

Fill in the `## Actual result + learning` and `## Next action`
sections of §2a. Action items go into the same tracker the
team uses for incident follow-ups (module-level
[mistake #5](../SOLUTION.md) — "postmortems with no
follow-through").

A reference "action" is concrete and has an owner and a
re-verification date:

> **Action:** raise the readiness probe `initialDelaySeconds`
> from 2 s to 10 s so a pod killed mid-warm-up is withdrawn
> from the LB before clients see errors.
> **Owner:** ranking-team.
> **Reverify:** replay the same experiment after the change
> is merged; expect 0 observable 5xx.

### 2g. Scheduling chaos as a discipline

Per module-level [Decision 5](../SOLUTION.md), chaos is a
cadence, not a stunt. The reference cadence:

- **Weekly** — low-blast-radius experiments in staging, one per
  service.
- **Monthly** — the same experiment set in production, during
  a declared window.
- **Quarterly** — a game day: a half-day exercise with
  multi-fault scenarios and all on-callers watching.

Experiments are checked in to the service repo so the "monthly
in production" run is the same CR a human applied last month,
not a hand-edited one.

## 3. Validation steps

1. `chaosctl version` prints a non-zero version (or Helm shows
   `chaos-mesh` released); CRDs `PodChaos`, `NetworkChaos`, etc.
   exist.
2. The experiment document (§2a) is reviewed and merged before
   any CR is applied — the grader can see the hypothesis and
   abort conditions in version control, not in Slack.
3. The abort-guard `PrometheusRule` is deployed and the runbook
   URL resolves to a one-line stop command.
4. Experiment applied in staging; `kubectl get podchaos -n staging`
   shows `Running`; pods in `app=api` are observably killed on
   schedule.
5. Grafana shows the SLI dip and recovery within the expected
   window; neither abort-guard alert fires.
6. Experiment applied in production; same behaviour; experiment
   is deleted at the end of the declared window.
7. Post-run write-up updated in the same PR; action items (if
   any) tracked in the team's issue tracker with an owner.
8. Running `kubectl apply -f experiment.yaml` is idempotent —
   the experiment can be rerun unchanged on the schedule.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Experiment document exists with hypothesis, steady state, perturbation, blast radius, abort conditions, observers | 20 |
| R2 | Steady state is expressed as SLIs from the existing SLO definitions, not ad-hoc metrics | 10 |
| R3 | Abort conditions are tighter than the SLO page threshold, and documented | 10 |
| R4 | Chaos Mesh CR (or equivalent) is version-controlled, not manually crafted | 10 |
| R5 | Abort-guard Prometheus alert deployed and connected to a one-line killswitch runbook | 10 |
| R6 | Experiment runs first in staging, then in production with declared window | 10 |
| R7 | Post-run document records actual result and at least one action with owner and reverification date | 10 |
| R8 | Experiment is on a declared cadence (weekly / monthly / quarterly), not a one-off | 5 |
| R9 | Blast radius explicitly scoped (`mode: one` or `mode: fixed-percent` with justification) | 10 |
| R10 | No perturbation that bypasses the abort path (e.g., killing the observability stack itself) | 5 |

Pass threshold: **80/100**. R1, R3, and R5 are must-haves. A
chaos experiment without an abort condition or without a
runbook is "an outage you caused on purpose."

## 5. Common mistakes

1. **No hypothesis.** The experiment becomes "let's see what
   breaks," and whatever breaks is accepted as information
   rather than evaluated against a target.
2. **Abort condition equal to the page alert.** The guard fires
   at the same threshold as the real SLO alert, which means it
   fires when the budget has already been burned.
3. **Blast radius defined by vibes.** "One pod" is fine; "a few
   pods" is not — the grader will ask what fraction and whether
   it was justified.
4. **Running chaos in prod with no staging rehearsal.** Chaos is
   a safety discipline; it is not an invitation to skip the
   safer step.
5. **Taking out the observability stack itself.** Killing
   Prometheus during a chaos run is possible (you'd want to know
   it's resilient) but it must not be in the same experiment
   that uses Prometheus to decide whether to abort. Separate
   experiments, separate blast radii.
6. **One-time stunt.** Module-level mistake #6; still the most
   common failure.
7. **No post-write-up.** A chaos experiment with no result and
   no action is a story, not a discipline.
8. **Chaos applied without on-call awareness.** If the on-caller
   finds out when their pager goes off, trust in the program
   evaporates.

## 6. References

Primary (official project documentation and canonical sources):

- Principles of Chaos Engineering —
  <https://principlesofchaos.org/>
- Chaos Mesh documentation — <https://chaos-mesh.org/docs/>
- Chaos Mesh Workflow — <https://chaos-mesh.org/docs/create-chaos-mesh-workflow/>
- LitmusChaos documentation (alternative implementation) —
  <https://litmuschaos.io/>
- Google SRE Workbook — Testing Reliability (game-day cadence,
  experiment design) —
  <https://sre.google/workbook/testing-reliability/>
- Prometheus Alertmanager (receiver webhooks used for
  automated-abort wiring) —
  <https://prometheus.io/docs/alerting/latest/alertmanager/>

Security and governance context:

- NIST AI Risk Management Framework — MANAGE 3.2 covers the
  resilience-testing function chaos engineering implements —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS — resilience testing is one of the defender-side
  disciplines that reveals which adversary techniques would
  succeed if exploited — <https://atlas.mitre.org/>

Local context:

- Module rationale: `modules/mod-207-observability-sre/SOLUTION.md`
  (Decision 5 — chaos engineering as a scheduled discipline).
- SLIs reused as steady-state: [exercise-02](../exercise-02/SOLUTION.md).
- Observability stack experiments run against: [exercise-01](../exercise-01/SOLUTION.md).
- Engineer-tier implementation reference: `engineer-solutions/mod-108`
  (per module README cross-reference).
