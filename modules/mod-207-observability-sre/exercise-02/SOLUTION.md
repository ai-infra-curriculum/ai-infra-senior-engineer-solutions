# SOLUTION — Lab 02: SLO Implementation and Monitoring

Reference solution for
[lab-02-slo-implementation.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-207-observability-sre/exercises/lab-02-slo-implementation.md).

The engineer-tier form of this exercise is "put a threshold on
error rate and alert when it crosses." The senior-tier
responsibility is **SLOs with error budgets and multi-window
multi-burn-rate alerts** — the alerting shape Google's SRE
Workbook defines as the canonical form
([SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)).

## 1. Solution overview

An SLO is a target over a window. The reference uses the
definitions from the Google SRE book
([Site Reliability Engineering — Chapter 4: Service Level
Objectives](https://sre.google/sre-book/service-level-objectives/)):

- **SLI** — Service Level Indicator: the measured quantity
  (e.g., ratio of successful requests, p99 latency).
- **SLO** — Service Level Objective: the target the SLI should
  meet over a window (e.g., 99.9% of requests succeed over 30
  days).
- **Error budget** — the complement of the SLO. A 99.9% SLO
  means a 0.1% error budget; burning it faster than linearly is
  the alerting signal.

Senior-tier additions beyond "define an SLO":

1. **SLI selection** grounded in user-visible outcomes, not
   infrastructure counters.
2. **Multi-window multi-burn-rate alerts**, as documented in the
   SRE Workbook. Fast-burn pages; slow-burn tickets.
3. **Recording rules** that pre-compute the SLI and the burn
   rate so dashboards and alerts share the same series.
4. **Error-budget-aware feature pacing** — when the budget is
   exhausted, feature work pauses for reliability work (the
   module-level [Decision 6](../SOLUTION.md)).
5. **An SLO document per service** with the SLI definition, the
   target, the window, and the review cadence.

## 2. Worked answer or implementation

### 2a. Pick the SLIs

The reference service is the `api` HTTP service from
[exercise-01](../exercise-01/SOLUTION.md). Two SLIs cover the
primary user experience:

- **Availability** — fraction of requests that succeed:
  `success / total` where success is `code !~ "5.."`.
- **Latency** — fraction of requests fast enough:
  `fast / total` where fast is `duration <= 200ms`.

Both are *ratio* SLIs — a count over a count — which is the
shape the SRE Workbook recommends for burn-rate math
([SRE Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/)).

### 2b. Pick SLOs and window

Reference values for a lab service (students should pick targets
justified by business context, not these numbers verbatim):

| SLI | SLO | Window | Error budget |
|---|---|---|---|
| Availability | 99.9% successful responses | 30 days rolling | 0.1% |
| Latency | 99% of requests ≤ 200ms | 30 days rolling | 1% |

The SRE book cautions against "unachievable SLOs" — if
dependencies collectively run at 99.9%, a 99.99% SLO is
impossible without architectural changes
([SRE book — Service Level Objectives: Choosing Targets](https://sre.google/sre-book/service-level-objectives/)).
See also module-level mistake #1.

### 2c. Recording rules for SLI and burn rate

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-slo
  namespace: observability
  labels:
    release: kps
spec:
  groups:
    - name: api.sli
      interval: 30s
      rules:
        # Totals and bad events
        - record: slo:api_requests:rate5m
          expr: sum(rate(http_requests_total{job="api"}[5m]))
        - record: slo:api_requests_errors:rate5m
          expr: sum(rate(http_requests_total{job="api",code=~"5.."}[5m]))
        - record: slo:api_requests_slow:rate5m
          expr: |
            sum(rate(http_requests_total{job="api"}[5m]))
            - sum(rate(http_request_duration_seconds_bucket{job="api",le="0.2"}[5m]))

        # Error ratios across multiple windows
        - record: slo:api_availability_error_ratio:5m
          expr: slo:api_requests_errors:rate5m / slo:api_requests:rate5m
        - record: slo:api_latency_error_ratio:5m
          expr: slo:api_requests_slow:rate5m / slo:api_requests:rate5m

    - name: api.sli.long
      interval: 1m
      rules:
        - record: slo:api_availability_error_ratio:1h
          expr: |
            sum(rate(http_requests_total{job="api",code=~"5.."}[1h]))
            / sum(rate(http_requests_total{job="api"}[1h]))
        - record: slo:api_availability_error_ratio:6h
          expr: |
            sum(rate(http_requests_total{job="api",code=~"5.."}[6h]))
            / sum(rate(http_requests_total{job="api"}[6h]))
        - record: slo:api_availability_error_ratio:3d
          expr: |
            sum(rate(http_requests_total{job="api",code=~"5.."}[3d]))
            / sum(rate(http_requests_total{job="api"}[3d]))
```

The latency SLI uses histogram buckets — your application must
record `http_request_duration_seconds` with a bucket at `0.2`.
Prometheus histograms are documented in
[Prometheus docs — Metric types: Histograms](https://prometheus.io/docs/concepts/metric_types/#histogram).

### 2d. Multi-window multi-burn-rate alerts

The SRE Workbook defines two alert tiers with two-window
confirmation each
([SRE Workbook — Alerting on SLOs: Multiwindow, multi-burn-rate alerts](https://sre.google/workbook/alerting-on-slos/#6-multiwindow-multi-burn-rate-alerts)):

| Severity | Long window | Short window | Burn rate | Budget consumed if sustained |
|---|---|---|---|---|
| Page (fast burn) | 1 h | 5 m | 14.4× | 2% in 1 h |
| Ticket (slow burn) | 6 h | 30 m | 6× | 5% in 6 h |

For a 99.9% SLO (0.1% error budget) these translate to error
ratios of **1.44%** (14.4 × 0.001) and **0.6%** (6 × 0.001).

```yaml
    - name: api.alerts
      rules:
        - alert: ApiAvailabilityFastBurn
          expr: |
            (slo:api_availability_error_ratio:1h > (14.4 * 0.001))
            and
            (slo:api_availability_error_ratio:5m > (14.4 * 0.001))
          for: 2m
          labels:
            severity: page
            service: api
            slo: availability
          annotations:
            summary: "api: availability error budget burning at ≥14.4× (1h)"
            description: |
              Error ratio over the last hour is above the 14.4× burn-rate
              threshold for the 99.9% availability SLO, and the 5m window
              confirms. At this rate the 30d error budget is exhausted in
              ~2 days.
            runbook_url: "https://runbooks.example.com/api/availability-fast-burn"

        - alert: ApiAvailabilitySlowBurn
          expr: |
            (slo:api_availability_error_ratio:6h > (6 * 0.001))
            and
            (slo:api_availability_error_ratio:30m > (6 * 0.001))
          for: 15m
          labels:
            severity: ticket
            service: api
            slo: availability
          annotations:
            summary: "api: availability error budget burning at ≥6× (6h)"
            runbook_url: "https://runbooks.example.com/api/availability-slow-burn"
```

Two parallel rules cover the latency SLO using
`slo:api_latency_error_ratio:*`. The **`and` between windows is
the point** — single-window alerts either page on 60-second
spikes (short window alone) or miss creeping degradation (long
window alone). The two-window form is what the Workbook pages 2
("Issues" and "Alerting on SLOs") call out as the canonical
shape.

### 2e. Error budget remaining — a dashboard panel

```promql
# Error budget consumed (fraction of allowed error used) over 30d:
(
  sum(increase(http_requests_total{job="api",code=~"5.."}[30d]))
  /
  sum(increase(http_requests_total{job="api"}[30d]))
)
/ 0.001
```

Values < 1 mean budget remains; ≥ 1 means the SLO has been
missed for the window. A Grafana *stat* panel with thresholds
at `0.5`, `0.8`, `1.0` is the reference visualization. The
underlying math is from
[SRE Workbook — Implementing SLOs: Error budget](https://sre.google/workbook/implementing-slos/).

### 2f. SLO document (per service)

Every service with an SLO has a short, reviewed document. The
reference template:

```markdown
# SLOs — api

## Owner
ranking-team

## SLIs
- Availability: fraction of HTTP responses not in 5xx.
- Latency: fraction of HTTP responses with duration <= 200ms.

## SLOs
- Availability: 99.9% over 30d rolling.
- Latency: 99.0% ≤ 200ms over 30d rolling.

## Why these numbers
- Availability: upstream dependencies combined availability
  observed ≥ 99.95%. A 99.9% target leaves headroom for our
  own failure domain and matches the user-perceived outage
  tolerance stated by product (`<link>`).
- Latency: measured p99 of successful transactions on the
  reference dataset was X ms (`<link>`); 200 ms is the
  user-perceived budget.

## Alerting
- Fast burn: 14.4×, 1h ∩ 5m, severity=page.
- Slow burn: 6×, 6h ∩ 30m, severity=ticket.

## Error-budget policy
- Budget consumed ≥ 100% for the window → feature freeze until
  reliability work restores ≥ 50% headroom.
- Budget consumed ≥ 50% → weekly review in team standup.

## Review cadence
- SLOs reviewed quarterly.
- Error-budget policy reviewed when the team composition or
  product commitments change.
```

The explicit "why these numbers" and the error-budget policy
are the parts most engineer-tier SLO definitions leave out;
they are the reason senior-tier SLOs survive first contact with
product pressure.

### 2g. Burn-rate alert routing

Alertmanager (configured in [exercise-01](../exercise-01/SOLUTION.md))
routes on `severity`:

- `severity=page` → PagerDuty / Opsgenie / on-call pager.
- `severity=ticket` → Slack / Jira creation.

An SLO burn-rate alert always carries `slo=<name>` so the
on-caller can filter and so the retrospective tool can tie the
incident to the SLO.

## 3. Validation steps

1. All six recording rules return series in Prometheus
   (`slo:api_requests:rate5m` etc.) within one evaluation
   interval of `PrometheusRule` apply.
2. Inject a synthetic error load that pushes the error ratio
   above 1.44%; after both the 1h and 5m windows reflect it,
   `ApiAvailabilityFastBurn` fires, and Alertmanager routes to
   the `pager` receiver.
3. Stop the error load; the alert resolves within two
   evaluation intervals.
4. Inject a lower-grade error load (around 0.7%); only
   `ApiAvailabilitySlowBurn` fires, after the 30m window catches
   up; the pager is not touched.
5. The Grafana SLO dashboard shows the correct remaining budget
   fraction for the 30-day window.
6. `promtool check rules api-slo.yaml` returns success.
7. `promtool test rules` exercises a hand-authored timeseries
   fixture and confirms both burn-rate alerts fire at the
   intended thresholds
   ([Prometheus docs — Unit testing for rules](https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/)).

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | At least two SLIs defined; each is a ratio and is grounded in user-visible outcomes | 10 |
| R2 | SLOs stated with explicit target, window, and justification text | 10 |
| R3 | Recording rules produce short- and long-window error ratios reused by alerts and dashboards | 10 |
| R4 | Multi-window multi-burn-rate alerts present for every SLO: page and ticket tiers, two windows each | 20 |
| R5 | Burn-rate thresholds match the Workbook table (14.4× fast, 6× slow — or values the student justifies against it) | 10 |
| R6 | Each alert has a `runbook_url` annotation and a `severity` label used by Alertmanager routing | 10 |
| R7 | SLO document names the owner, SLIs, targets, justification, alerting shape, and the error-budget policy | 10 |
| R8 | A Grafana panel shows remaining error budget for the 30-day window | 5 |
| R9 | `promtool` rule-unit tests exercise the burn-rate alerts | 10 |
| R10 | No "aspirational" SLOs tighter than achievable given dependency availability | 5 |

Pass threshold: **80/100**. R4 is a must-have — a single-window
alert is the engineer-tier shape and fails the senior bar
regardless of total score.

## 5. Common mistakes

1. **SLI that measures the infrastructure, not the user.** CPU,
   memory, or pod-restart counts are not SLIs — they are causes.
   The SLI is whatever the user perceives.
2. **Unachievable SLOs.** 99.99% behind dependencies at 99.9%.
   The error budget is burnt before the week begins.
3. **Single-window alerts.** Either page on noise or miss slow
   creep; see §2d.
4. **Burn-rate thresholds not tied to the SLO.** Hard-coding
   "error ratio > 0.01" instead of "burn rate > 14.4× for a
   99.9% SLO" means re-tuning every time the SLO changes.
5. **Error-budget policy left implicit.** "We'll slow down if
   it's bad" is not a policy. Senior-tier policies state
   "budget consumed ≥ X → action Y" in writing.
6. **SLOs for every endpoint.** The module-level trade-off
   explicitly calls this out — start with 3–5 SLOs per service.
7. **Latency SLO without histogram buckets.** Averaged latency
   hides tail behavior; the SLI needs bucket data (`_bucket`).
8. **Alerts without runbook URLs.** Same mistake as
   exercise-01; still common.
9. **Changing SLOs after the fact to make dashboards green.**
   The point of the SLO is to force a conversation when the
   service misses it, not to redefine success at week's end.

## 6. References

Primary (official project documentation and canonical standards):

- Google SRE book — Chapter 4, Service Level Objectives —
  <https://sre.google/sre-book/service-level-objectives/>
- Google SRE Workbook — Implementing SLOs —
  <https://sre.google/workbook/implementing-slos/>
- Google SRE Workbook — Alerting on SLOs (multi-window
  multi-burn-rate alerts) —
  <https://sre.google/workbook/alerting-on-slos/>
- Prometheus documentation — <https://prometheus.io/docs/>
- Prometheus recording rules — <https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/>
- Prometheus alerting rules — <https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/>
- Prometheus histograms — <https://prometheus.io/docs/concepts/metric_types/#histogram>
- Prometheus rule unit testing — <https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/>

Security and governance context:

- NIST AI Risk Management Framework — MEASURE 2.7 explicitly
  covers performance monitoring thresholds; SLOs are the
  operational form of those thresholds —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — ML09 Output Integrity
  is detected by latency and error SLIs on the prediction path
  — <https://owasp.org/www-project-machine-learning-security-top-10/>

Local context:

- Module rationale: `modules/mod-207-observability-sre/SOLUTION.md`
  (Decisions 1, 2, 6 — SLOs, multi-burn-rate alerts, per-team
  error budgets).
- Upstream stack: [exercise-01](../exercise-01/SOLUTION.md).
- Consumer of the error-budget policy:
  [exercise-04](../exercise-04/SOLUTION.md) and
  [exercise-05](../exercise-05/SOLUTION.md).
- Engineer-tier implementation reference: `engineer-solutions/mod-108`
  (per module README cross-reference).
