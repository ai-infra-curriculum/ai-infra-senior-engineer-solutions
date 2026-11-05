# SOLUTION — Lab 01: Complete Observability Stack Setup

Reference solution for
[lab-01-prometheus-grafana-setup.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-207-observability-sre/exercises/lab-01-prometheus-grafana-setup.md).

The engineer-tier form of this exercise is "deploy Prometheus and
Grafana and scrape an app." The senior-tier responsibility is to
stand up the stack as **three signals plus alerting** — metrics,
logs, traces, and an Alertmanager routing tree — so that every
subsequent lab in this module (SLOs, chaos, incident response) has
real data to operate on.

## 1. Solution overview

A complete observability stack at senior tier has four
cooperating subsystems, not one:

- **Metrics** — Prometheus scraping metric endpoints, with
  recording and alerting rules
  ([Prometheus docs — Overview](https://prometheus.io/docs/introduction/overview/)).
- **Logs** — Loki (or an equivalent) collecting structured
  application logs tagged with the same labels used in
  Prometheus ([Grafana Loki docs](https://grafana.com/docs/loki/latest/)).
- **Traces** — OpenTelemetry Collector + Tempo (or Jaeger)
  receiving OTLP from the applications
  ([OpenTelemetry docs](https://opentelemetry.io/docs/)).
- **Alerting** — Alertmanager handling deduplication, grouping,
  silencing, and routing to pager and chat channels
  ([Prometheus docs — Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)).

Grafana is the UI that unifies all four signals via datasources
([Grafana docs — Data sources](https://grafana.com/docs/grafana/latest/datasources/)).

The reference deployment is Kubernetes-based and uses the
`kube-prometheus-stack` Helm chart as the metrics/alerting
foundation
([kube-prometheus-stack chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)),
with Loki and Tempo installed alongside and wired into Grafana as
pre-configured datasources.

## 2. Worked answer or implementation

### 2a. Namespace layout

```
observability/
  prometheus (kube-prometheus-stack)
  alertmanager
  grafana
  loki
  tempo
  otel-collector
```

Everything lives in a single `observability` namespace so RBAC,
network policy, and PodMonitor/ServiceMonitor selectors can be
scoped uniformly. The chart defaults are acceptable for the lab;
production shapes (dedicated nodes, external storage) are called
out in §2g.

### 2b. Install kube-prometheus-stack

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace observability

helm install kps prometheus-community/kube-prometheus-stack \
  -n observability \
  -f values.yaml
```

`values.yaml` turns on the pieces we need and sets retention:

```yaml
# values.yaml
prometheus:
  prometheusSpec:
    retention: 15d
    retentionSize: "40GiB"
    scrapeInterval: 15s
    evaluationInterval: 15s
    # Discover PodMonitor / ServiceMonitor in any namespace
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

alertmanager:
  config:
    route:
      receiver: "default"
      group_by: ["alertname", "cluster", "service"]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      routes:
        - matchers: ['severity="page"']
          receiver: "pager"
          continue: true
        - matchers: ['severity="ticket"']
          receiver: "ticket"
    receivers:
      - name: "default"
      - name: "pager"
        # webhook_configs / pagerduty_configs here
      - name: "ticket"
        # slack_configs / webhook_configs here
grafana:
  adminPassword: "changeme-in-ci"
  defaultDashboardsEnabled: true
  sidecar:
    datasources:
      enabled: true
    dashboards:
      enabled: true
      searchNamespace: ALL
```

`ServiceMonitor` and `PodMonitor` are the Prometheus Operator
custom resources that replace raw scrape configs
([Prometheus Operator docs — ServiceMonitor](https://prometheus-operator.dev/docs/user-guides/getting-started/)).
Setting `*SelectorNilUsesHelmValues: false` lets Prometheus pick
up CRs the chart did not install (i.e., your application's own).

### 2c. Scrape an application

Your application exposes Prometheus metrics on `:8080/metrics`
(any Prometheus client library works —
[Prometheus client libraries](https://prometheus.io/docs/instrumenting/clientlibs/)).
A `ServiceMonitor` tells the Operator to scrape it:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api
  namespace: production
  labels:
    release: kps   # must match kube-prometheus-stack release
spec:
  selector:
    matchLabels:
      app: api
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
      scrapeTimeout: 10s
```

The pod's `Service` must expose a port named `http` matching
this selector. The chart's Prometheus will discover the
ServiceMonitor within one reload cycle.

### 2d. Recording and alerting rules

Recording rules pre-compute common expressions so dashboards and
alerts are cheap
([Prometheus docs — Recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)).
Alerting rules are the paging inputs
([Prometheus docs — Alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)).
Deploy them as `PrometheusRule` CRs:

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
    - name: api.rules
      interval: 30s
      rules:
        - record: job:http_requests:rate5m
          expr: sum by (job) (rate(http_requests_total[5m]))
        - record: job:http_requests_errors:rate5m
          expr: sum by (job) (rate(http_requests_total{code=~"5.."}[5m]))
        - record: job:http_request_error_ratio:5m
          expr: job:http_requests_errors:rate5m
                / job:http_requests:rate5m

    - name: api.alerts
      rules:
        - alert: ApiHighErrorRate
          expr: job:http_request_error_ratio:5m > 0.05
          for: 10m
          labels:
            severity: ticket
            service: api
          annotations:
            summary: "API error rate > 5% for 10m"
            runbook_url: "https://runbooks.example.com/api/high-error-rate"
```

The real burn-rate alerts live in
[exercise-02](../exercise-02/SOLUTION.md) — this rule only
demonstrates that the plumbing works end-to-end.

### 2e. Logs with Loki + Promtail

Install Loki alongside:

```bash
helm install loki grafana/loki-stack \
  -n observability \
  --set promtail.enabled=true \
  --set grafana.enabled=false
```

Promtail ships Kubernetes pod logs into Loki with labels
`namespace`, `pod`, `container`, and any `__meta_kubernetes_*`
labels you relabel in. Loki's key constraint is "labels cheap,
log lines unlimited" — do **not** promote high-cardinality
fields (user id, trace id) to labels; keep them in the log line
and extract with LogQL at query time
([Loki docs — Labels best practices](https://grafana.com/docs/loki/latest/get-started/labels/)).

### 2f. Traces with OpenTelemetry + Tempo

Install the OpenTelemetry Collector and Tempo:

```bash
helm install otel open-telemetry/opentelemetry-collector \
  -n observability \
  -f otel-values.yaml

helm install tempo grafana/tempo -n observability
```

`otel-values.yaml`:

```yaml
mode: deployment
config:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
  processors:
    batch: {}
    memory_limiter:
      check_interval: 1s
      limit_percentage: 75
  exporters:
    otlp:
      endpoint: tempo:4317
      tls:
        insecure: true
  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [otlp]
```

Applications emit OTLP to the collector; the collector batches
and forwards to Tempo. See
[OpenTelemetry Collector docs](https://opentelemetry.io/docs/collector/)
and [Grafana Tempo docs](https://grafana.com/docs/tempo/latest/).

### 2g. Grafana datasources and dashboards

Pre-configure datasources via the sidecar — any `ConfigMap`
labelled `grafana_datasource: "1"` is picked up
([Grafana docs — Provisioning datasources](https://grafana.com/docs/grafana/latest/administration/provisioning/#data-sources)):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: observability
  labels:
    grafana_datasource: "1"
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        url: http://kps-prometheus:9090
        isDefault: true
      - name: Loki
        type: loki
        access: proxy
        url: http://loki:3100
      - name: Tempo
        type: tempo
        access: proxy
        url: http://tempo:3100
        jsonData:
          tracesToLogs:
            datasourceUid: loki
```

`tracesToLogs` wires the three signals together: clicking a span
in Tempo jumps to the matching Loki query for the same pod and
time range ([Grafana docs — Trace to logs](https://grafana.com/docs/grafana/latest/datasources/tempo/configure-tempo-data-source/#trace-to-logs)).

Dashboards as `ConfigMap`s with `grafana_dashboard: "1"` are
similarly auto-loaded. For the lab, the chart's built-in
Kubernetes dashboards plus a single application-level RED
dashboard (rate / errors / duration —
[The RED Method, Grafana Labs](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/))
are sufficient.

### 2h. Production hardening (noted, not fully implemented in the lab)

The lab uses the chart defaults; production requires:

- **External storage** for Prometheus: remote-write to Mimir,
  Thanos, or VictoriaMetrics for long-term retention
  ([Prometheus docs — Remote write](https://prometheus.io/docs/practices/remote_write/)).
- **HA Alertmanager**: three replicas with gossip; chart supports
  this via `alertmanager.alertmanagerSpec.replicas: 3`.
- **Separate node pool** for observability workloads to prevent
  tenant apps from starving the monitoring plane.
- **RBAC + network policy** restricting scrape, write, and
  Alertmanager API access.
- **Secrets** (admin passwords, pager integration keys) managed
  via a secret manager, not Helm values.

## 3. Validation steps

1. `kubectl get pods -n observability` — Prometheus, Alertmanager,
   Grafana, Loki/Promtail, Tempo, OTel collector all `Running`.
2. `kubectl port-forward -n observability svc/kps-prometheus 9090:9090`
   → open `http://localhost:9090/targets` → the application
   `ServiceMonitor` is listed and `State=UP`.
3. In Prometheus, run `up{job="api"} == 1` and get at least one
   series; run `job:http_request_error_ratio:5m` and get the
   recording-rule series.
4. `kubectl exec -it <pager-test-pod> -- curl kps-alertmanager:9093/api/v2/alerts`
   shows the active alerts; trigger `ApiHighErrorRate` with a
   synthetic error load and verify Alertmanager routes to the
   configured receiver.
5. `kubectl port-forward -n observability svc/grafana 3000:80` →
   log in → Prometheus / Loki / Tempo datasources all show
   green under *Settings → Data sources*.
6. In Grafana Explore: a LogQL query
   `{namespace="production", pod=~"api-.*"}` returns the app's
   logs; a TraceQL query returns traces for the same pod.
7. From a span in Tempo, "View logs" jumps to the Loki query at
   the same time range for the same pod.
8. Restart the Prometheus pod; after it comes back, data for the
   last minutes before the restart is still present (the PVC
   survived) and new data continues to arrive.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | kube-prometheus-stack installed with ServiceMonitor discovery enabled across namespaces | 10 |
| R2 | At least one application `ServiceMonitor` is scraped (target UP) | 10 |
| R3 | At least one recording rule and one alerting rule deployed via `PrometheusRule` CR | 10 |
| R4 | Alertmanager configured with severity-based routing and group/repeat intervals set (not defaults) | 10 |
| R5 | Loki + Promtail installed; log labels do not include high-cardinality fields | 10 |
| R6 | OTel Collector receives OTLP and forwards to Tempo; application sends traces | 10 |
| R7 | Grafana has Prometheus, Loki, and Tempo datasources provisioned via ConfigMap (not clicked in by hand) | 10 |
| R8 | Traces-to-logs wiring works in Grafana Explore | 10 |
| R9 | Prometheus has a PVC and retention is explicitly set (`15d` reference, not the 2h default) | 5 |
| R10 | Runbook URL annotation on each alerting rule | 5 |
| R11 | At least one RED dashboard for an application service (rate / errors / duration) | 10 |

Pass threshold: **80/100**. R3, R4, and R7 are must-haves —
metrics with no rules, alerts with no routing, or datasources
only configured by hand are the three failure modes the grader
most commonly sees.

## 5. Common mistakes

1. **Scraping without `ServiceMonitor`/`PodMonitor`.** Writing
   raw `scrape_configs` into Prometheus values defeats the
   operator; configs drift and new services don't get picked up.
2. **Labelling logs with trace IDs or user IDs in Loki.** Label
   cardinality explodes Loki's index; keep those in the log line
   and use `| json | line_format` at query time.
3. **Default Alertmanager routing.** Everything lands in one
   receiver, everyone mutes everything, and the pager is
   worthless by week three. Severity routing is table stakes.
4. **No `for:` on alerting rules.** Single-sample spikes page
   on-call for noise. The `for:` clause is the difference
   between an alert and a smoke detector.
5. **No runbook URL.** On-call gets paged at 3am with "API error
   rate high." Without the runbook link, the next action is
   "wake someone else up." The annotation is a 10-second fix.
6. **Prometheus storage on `emptyDir`.** Data evaporates on pod
   restart. The chart defaults to a PVC when `storageSpec` is
   set; this is non-optional.
7. **One Grafana admin password shared across the team.** Use
   the Grafana auth integration (OIDC, SAML) and treat admin as
   break-glass.
8. **Running the whole stack on the same nodes as the
   workloads it monitors.** When nodes are under pressure, the
   observability stack dies at exactly the moment you needed to
   see it. Isolate on its own node pool in production.

## 6. References

Primary (official project documentation):

- Prometheus — <https://prometheus.io/docs/introduction/overview/>
- Prometheus Alertmanager — <https://prometheus.io/docs/alerting/latest/alertmanager/>
- Prometheus Operator — <https://prometheus-operator.dev/>
- kube-prometheus-stack Helm chart —
  <https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack>
- Grafana — <https://grafana.com/docs/grafana/latest/>
- Grafana Loki — <https://grafana.com/docs/loki/latest/>
- Grafana Tempo — <https://grafana.com/docs/tempo/latest/>
- OpenTelemetry — <https://opentelemetry.io/docs/>
- The RED Method (Grafana Labs) —
  <https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/>

Security and governance context:

- NIST AI Risk Management Framework — MEASURE 2.6 (TEVV of
  monitoring systems) is the framework slot that an observability
  stack occupies for an AI workload —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS — detection capabilities rely on the metrics and
  logs this stack produces — <https://atlas.mitre.org/>

Local context:

- Module rationale: `modules/mod-207-observability-sre/SOLUTION.md`.
- Engineer-tier implementation reference: `engineer-solutions/mod-108`
  (per module README cross-reference).
- Downstream labs that consume this stack:
  [exercise-02](../exercise-02/SOLUTION.md) (SLOs),
  [exercise-03](../exercise-03/SOLUTION.md) (chaos).
