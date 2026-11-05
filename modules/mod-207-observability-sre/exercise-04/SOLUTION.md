# SOLUTION — Lab 04: Observability Strategy

Reference solution for
[lab-04-observability-strategy.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-207-observability-sre/exercises/lab-04-observability-strategy.md).

This is a **design exercise**. The deliverable is not code; it is
a written observability strategy a senior engineer would present
to a director of infrastructure and defend to each team in the
org. Per the AICG design-exercise contract, the sections below
cover a worked example, decision rationale, and grading rubric.

## 1. Solution overview

A senior-tier observability strategy answers four questions, in
order. Every question has an inherited default; a strategy
document is a set of **choices against those defaults, justified**:

1. **Signals** — what we collect (metrics, logs, traces, events,
   profiles).
2. **Pipeline** — how it flows (agents, collectors, storage,
   retention).
3. **Contract** — who owns what, which SLIs and SLOs, which
   dashboards are canonical.
4. **Discipline** — the operational practices (incident
   response, postmortems, chaos cadence) the signals feed.

The reference strategy for a mid-size AI-infra platform is below;
students may propose variants, but the rubric scores against the
four-section shape and the decisions that must appear in each.

The strategy composes the implementation-tier labs already
delivered in this module:
- Stack: [exercise-01](../exercise-01/SOLUTION.md).
- SLOs: [exercise-02](../exercise-02/SOLUTION.md).
- Chaos cadence: [exercise-03](../exercise-03/SOLUTION.md).
- On-call program: [exercise-05](../exercise-05/SOLUTION.md).

## 2. Worked answer or implementation — reference strategy

### 2a. Scope

The strategy covers an AI infrastructure platform serving (a)
online model inference, (b) feature serving, (c) batch training,
and (d) the data pipelines feeding both. It assumes the stack
from exercise-01 as the physical layer.

### 2b. Section 1 — Signals

Three primary signals, with an explicit opinion on each:

| Signal | Tool (reference) | Retention | What it's for |
|---|---|---|---|
| Metrics | Prometheus / Mimir | Hot 15 d, cold 400 d | SLIs, alerting, dashboards |
| Logs | Loki | Hot 30 d, cold 180 d | Debugging, incident forensics |
| Traces | Tempo / OTel | 7 d (sampled) | Latency root-cause, dependency maps |

Secondary signals, treated as "add if the primary signals leave
a gap":

- **Events** — Kubernetes event stream pulled via the operator;
  joined with metrics in dashboards.
- **Continuous profiling** — Pyroscope / Parca when CPU-bound
  service regressions are a recurring cause.

The strategy explicitly does **not** collect:

- Debug-level logs by default in production (toggle on per-pod,
  short-lived).
- Full 100% trace sampling (expensive, informationally
  redundant — see [OpenTelemetry sampling docs](https://opentelemetry.io/docs/concepts/sampling/)).

The three-signal model (metrics, logs, traces) is the one
documented in the OpenTelemetry Observability Primer
([OpenTelemetry docs — Observability primer](https://opentelemetry.io/docs/concepts/observability-primer/)).

### 2c. Section 2 — Pipeline

Reference pipeline (same as the stack in exercise-01, with a
long-term tier added):

```
          applications
               │
               ▼  OTLP / Prom scrape / log stdout
┌──────────────────────────────────────────────┐
│   OTel Collector  ·  Promtail  ·  cAdvisor   │
└──────────────────────────────────────────────┘
               │
               ▼
┌──────────────┬──────────────┬───────────────┐
│  Prometheus  │     Loki     │    Tempo      │
│  (15d hot)   │   (30d hot)  │  (7d hot)     │
└──────┬───────┴──────┬───────┴───────┬───────┘
       │ remote_write │ shipper       │ shipper
       ▼              ▼               ▼
      Mimir          Loki-S3         Tempo-S3
     (cold)          (cold)          (cold, sampled)
                    All backed by object storage
```

Key decisions in the pipeline:

- **Remote write for metrics**, not a federated Prometheus. A
  horizontally scalable backend (Mimir / Thanos / VictoriaMetrics)
  is the only shape that scales past one cluster
  ([Prometheus docs — Remote write](https://prometheus.io/docs/practices/remote_write/),
  [Grafana Mimir docs](https://grafana.com/docs/mimir/latest/)).
- **Object storage as the cold-tier substrate**, not a second
  database. One retention policy, one compaction story.
- **OTel Collector as the trace boundary**, not direct exporters
  from applications. The Collector is where sampling, PII
  redaction, and vendor swapping happen.
- **No cross-cutting log parsing in the collector.** Keep logs
  structured at source; query-time parsing in LogQL is cheaper
  than collector CPU.

### 2d. Section 3 — Contract

The strategy names four owner groups and their deliverables:

| Owner | Owns | Delivers |
|---|---|---|
| Observability platform team | The pipeline (§2c) | Uptime of Prometheus / Loki / Tempo / Grafana; datasource contract; retention |
| Service teams | Instrumentation of their own services | A ServiceMonitor, a RED dashboard, SLOs, alerting rules, runbooks |
| ML platform team | Instrumentation of the inference and training platforms | Model-serving SLIs; training-job SLIs; GPU utilisation dashboards |
| Governance | The audit trail | NIST AI RMF evidence, ATLAS threat-model entries, model-inventory links to dashboards |

Canonical dashboard rules:

- Each service owns exactly one canonical "service health"
  dashboard with a fixed URL; everything else is exploratory.
- Service health dashboards follow **RED** for request-driven
  services ([Grafana Labs — The RED Method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/))
  and **USE** for resource-bound components
  ([Brendan Gregg — The USE Method](https://www.brendangregg.com/usemethod.html)).
- Every alerting rule links back to a runbook URL (already a
  requirement from exercise-01 and exercise-02).

Canonical SLOs:

- Every "primary user path" has at least two SLOs (availability
  and latency), defined per the shape in exercise-02.
- Every data pipeline has a freshness SLO.
- Every model inference endpoint has an availability, latency,
  and output-schema-validity SLO. The last one is the AI-specific
  slot that engineer-tier observability leaves out and that is
  the first-line signal for OWASP ML09 Output Integrity
  ([OWASP ML Top 10](https://owasp.org/www-project-machine-learning-security-top-10/)).

### 2e. Section 4 — Discipline

Operational practices the signals feed:

- **Incident response.** Severity levels are defined by SLO
  impact (module-level [Decision 3](../SOLUTION.md)); severity
  determines pager routing and status-page cadence.
- **Postmortems.** Blameless template (module-level
  [Decision 4](../SOLUTION.md)); action items tracked in the
  same issue tracker as feature work and visible on the service
  health dashboard until closed.
- **Chaos cadence.** Weekly staging, monthly production,
  quarterly game days (see
  [exercise-03](../exercise-03/SOLUTION.md)).
- **On-call program.** See [exercise-05](../exercise-05/SOLUTION.md).
- **SLO review.** Quarterly; chair rotates across service teams.
- **Access review.** Grafana / Alertmanager admin rights
  reviewed quarterly; service accounts separated from humans.

### 2f. AI-specific overlays

AI workloads add specific signals the baseline strategy must
cover:

- **Model-output schema validation metric** on the serving path
  (count of predictions whose output fails the registered
  schema). Alert on any non-zero rate. Ties to OWASP ML09.
- **Prediction distribution drift metric** (JS divergence or
  equivalent against a reference window). Ties to OWASP ML08
  Model Skewing.
- **Training-data provenance logging** — every training run
  emits `dataset.sha256`, `code.commit`, `feature_service` into
  the metrics plane as an event. Ties to NIST AI RMF MAP 3.
- **Inference-cost SLI** — tokens-per-request or
  dollars-per-1k-requests for model endpoints; the strategy
  explicitly names it because cost incidents in AI workloads
  are often the first symptom of a regression.

AI-specific signals are gated through the same pipeline and are
owned by the ML platform team per §2d.

### 2g. Rollout plan (90-day)

Written as exit-criteria-per-stage, per module-level
convention:

- **Weeks 1–3.** Install the stack (exercise-01) in a single
  cluster. Exit: every service in that cluster has a
  ServiceMonitor and a scraped `up{}` series.
- **Weeks 4–6.** Each service team writes SLOs for its primary
  user paths (exercise-02). Exit: at least one SLO per service,
  with burn-rate alerts routing to pager and ticket.
- **Weeks 7–9.** Chaos cadence starts in staging
  (exercise-03). Exit: every service has at least one checked-in
  chaos experiment with an abort guard.
- **Weeks 10–12.** On-call program formalised
  (exercise-05). Exit: rotation calendar, runbook index,
  escalation policy, and blameless postmortem template all
  live.
- **Continuous (after week 4).** Governance (§2d) collects
  evidence against NIST AI RMF, OWASP ML Top 10, and MITRE
  ATLAS entries for AI workloads; model inventory links each
  model to its SLOs and runbooks.

### 2h. Trade-offs we accept

Explicit, honest trade-offs. A strategy with no trade-offs is
hiding them.

- **Cost ceiling.** Hot retention of 15 d for metrics and 30 d
  for logs is a cost compromise; most incident forensics happens
  within 24 h anyway.
- **Not Datadog.** Self-hosted stack is operational work for the
  platform team; the strategy document names this and names a
  trigger (team size + cost) at which to reconsider.
- **Trace sampling.** 100% sampling makes complete
  dependency-map builds easier; the strategy chooses a sampled
  default and names "turn up sampling during an incident" as
  the operational control.
- **One canonical dashboard per service.** Teams want many;
  limit is intentional. The strategy names an exception path
  (an exploratory dashboard can graduate to canonical with the
  service-team's approval).

### 2i. One-paragraph pitch

> Three signals — metrics, logs, traces — through one collector
> boundary into a horizontally scalable hot tier backed by
> object storage for the cold tier. Each service owns its own
> instrumentation and SLOs; the platform team owns the pipeline.
> AI workloads add schema-validity and drift SLIs on top. The
> signals feed four disciplines: incident response, blameless
> postmortems, scheduled chaos, and on-call. We land it in 90
> days starting with the stack install and the SLO writing in
> parallel.

## 3. Validation steps

This is a design review, not a test run. A submission is
validated by checking that the student deliverable:

1. Names the four questions (signals, pipeline, contract,
   discipline) and answers each with explicit choices (§2a–§2e).
2. Specifies retention for each signal with a rationale (§2b).
3. Includes a pipeline diagram with hot-tier *and* cold-tier
   components; the cold tier is backed by object storage (§2c).
4. Assigns ownership per deliverable; no deliverable is
   ownerless (§2d).
5. Names at least one AI-specific signal and the standard it
   maps to (§2f).
6. Lays out a staged rollout with exit criteria per stage, not
   just durations (§2g).
7. States at least three trade-offs explicitly (§2h).
8. Ends with an executive-ready one-paragraph pitch (§2i).

A successful submission reads like a strategy a senior engineer
could hand to a director of infrastructure and walk a service
team through in one meeting.

## 4. Rubric / review checklist

Weightings sum to 100. Below 70 → rewrite; 70–84 →
pass-with-comments; 85+ → ship.

| # | Criterion | Weight |
|---|---|---|
| R1 | Four-section shape (signals, pipeline, contract, discipline) present and scoped | 10 |
| R2 | Each signal has a retention policy with a stated rationale | 10 |
| R3 | Pipeline includes a hot tier *and* a cold tier (object storage), with explicit decisions on remote-write and OTel collector boundary | 15 |
| R4 | Ownership matrix names at least three groups and their deliverables | 10 |
| R5 | Canonical-dashboard and canonical-SLO rules named | 10 |
| R6 | AI-specific signals called out with mapping to OWASP ML Top 10 or NIST AI RMF | 10 |
| R7 | Rollout plan has exit criteria per stage, not just durations | 10 |
| R8 | At least three trade-offs stated explicitly and defensibly | 10 |
| R9 | Strategy integrates with the on-call program and chaos cadence (references the respective labs) | 5 |
| R10 | Executive one-paragraph pitch present and defensible | 5 |
| R11 | No invented facts, metrics, or incidents — claims are tied to official standards or to clearly labeled local context | 5 |

R11 is a must-have. A strategy with fabricated statistics
(e.g., "studies show logging costs scale 3× yearly") fails
regardless of total score.

## 5. Common mistakes

1. **Strategy as a toolchain list.** "We use Prometheus, Loki,
   Tempo" is a shopping list, not a strategy. The strategy is
   the contracts and the trade-offs.
2. **Retention by inertia.** "Default 15 d because the chart
   does" — without a stated rationale, the retention can't be
   defended at cost review.
3. **Everything owned by platform team.** Instrumentation
   ownership belongs with service teams; treating it as
   "platform problem" produces vendor-shaped systems.
4. **No cold tier.** 15 d hot with nothing behind it fails every
   quarterly cost incident and every compliance audit that asks
   for 180-day retention.
5. **No AI-specific signals.** An AI-infra observability
   strategy that could be used unchanged for a plain web service
   has missed the point.
6. **Rollout plan as a Gantt chart.** Durations without exit
   criteria become indefinite (module-level mistake).
7. **Trade-offs hidden.** A document with no trade-offs is
   either lying or hasn't been reviewed by the people who will
   bear the costs.
8. **Fabricated benchmarks.** Any specific vendor benchmark or
   cost comparison that lacks a source is an R11 failure.

## 6. References

Primary (official standards and project documentation):

- Google SRE book —
  <https://sre.google/sre-book/table-of-contents/>
- Google SRE Workbook —
  <https://sre.google/workbook/table-of-contents/>
- OpenTelemetry documentation —
  <https://opentelemetry.io/docs/>
- OpenTelemetry sampling —
  <https://opentelemetry.io/docs/concepts/sampling/>
- Prometheus remote-write —
  <https://prometheus.io/docs/practices/remote_write/>
- Grafana Mimir — <https://grafana.com/docs/mimir/latest/>
- Grafana Loki — <https://grafana.com/docs/loki/latest/>
- Grafana Tempo — <https://grafana.com/docs/tempo/latest/>
- The RED Method (Grafana Labs) —
  <https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/>
- The USE Method (Brendan Gregg) —
  <https://www.brendangregg.com/usemethod.html>

Governance and AI-specific context:

- NIST AI Risk Management Framework (AI RMF 1.0) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — <https://atlas.mitre.org/>

Local context:

- Module rationale: `modules/mod-207-observability-sre/SOLUTION.md`.
- Stack this strategy stands on: [exercise-01](../exercise-01/SOLUTION.md).
- SLO shape: [exercise-02](../exercise-02/SOLUTION.md).
- Chaos discipline: [exercise-03](../exercise-03/SOLUTION.md).
- On-call program the strategy feeds: [exercise-05](../exercise-05/SOLUTION.md).
- Engineer-tier implementation reference: `engineer-solutions/mod-108`
  (per module README cross-reference).
