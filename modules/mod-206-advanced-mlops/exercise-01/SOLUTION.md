# SOLUTION — Lab 01: Feature Store Implementation with Feast

Reference solution for
[lab-01-feature-store-implementation.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-206-advanced-mlops/exercises/lab-01-feature-store-implementation.md).

This senior-tier lab adds governance, freshness SLAs, and
train-serve-skew protection on top of the engineer-tier
feature-store exercise referenced in
`engineer-solutions/mod-106`. The underlying Feast mechanics are
the same; the senior responsibility is the contract around them.

## 1. Solution overview

Feast is an open-source feature store that separates three
concerns ([Feast docs — Concepts](https://docs.feast.dev/getting-started/concepts)):

- **Entities** — the join keys (e.g. `user_id`, `driver_id`).
- **Data sources** — where raw feature data lives (warehouse table,
  Parquet, stream).
- **Feature views** — the contract: a named group of features
  attached to an entity, with a TTL and a source.

The *same* feature view definition is used by:

- **Offline retrieval** (`get_historical_features`) during training,
  producing point-in-time-correct training data.
- **Online retrieval** (`get_online_features`) during serving,
  reading from a low-latency store (Redis, DynamoDB, etc.).

This dual-read is Feast's reason for existing: it makes
train-serve skew structurally difficult, because training and
serving are reading from the same declared feature view
([Feast docs — Overview](https://docs.feast.dev/)).

Senior-tier additions beyond "make Feast run":

- A **freshness SLA** per feature view with an alert when
  materialization lag exceeds it.
- A **governance contract** — PR-reviewed feature definitions,
  owner metadata, no ad-hoc feature creation at serving time.
- **Train-serve skew checks** as an explicit CI step, not a hope.

## 2. Worked answer or implementation

### 2a. Repository layout

```
feature_repo/
├── feature_store.yaml        # project config: registry, online/offline stores
├── entities.py               # Entity definitions
├── data_sources.py           # FileSource / BigQuerySource / KafkaSource
├── feature_views.py          # FeatureView definitions with TTLs
├── feature_services.py       # Named bundles for model versions
└── tests/
    ├── test_definitions.py   # repo validates (feast apply --dry-run)
    └── test_skew.py          # offline vs online retrieval parity
```

Layout follows the structure shown in
[Feast docs — Create a feature repository](https://docs.feast.dev/getting-started/quickstart).

### 2b. Minimal feature store config

`feature_store.yaml` — local-dev form, swap `online_store` and
`offline_store` for the production backends (Redis + BigQuery /
DynamoDB + Snowflake / etc.):

```yaml
project: mlops_senior
registry: data/registry.db
provider: local
online_store:
  type: sqlite
  path: data/online_store.db
offline_store:
  type: file
entity_key_serialization_version: 2
```

Supported providers and store backends are enumerated in
[Feast docs — feature_store.yaml](https://docs.feast.dev/reference/feature-repository/feature-store-yaml).

### 2c. Entity and feature view

```python
# entities.py
from feast import Entity, ValueType

driver = Entity(
    name="driver",
    join_keys=["driver_id"],
    value_type=ValueType.INT64,
    description="Delivery driver identified by driver_id.",
)
```

```python
# data_sources.py
from feast import FileSource
from feast.data_format import ParquetFormat

driver_stats_source = FileSource(
    name="driver_stats_source",
    path="data/driver_stats.parquet",
    timestamp_field="event_timestamp",
    created_timestamp_column="created",
)
```

```python
# feature_views.py
from datetime import timedelta
from feast import FeatureView, Field
from feast.types import Float32, Int64
from .entities import driver
from .data_sources import driver_stats_source

driver_stats_fv = FeatureView(
    name="driver_hourly_stats",
    entities=[driver],
    ttl=timedelta(hours=2),           # freshness SLA — see §2e
    schema=[
        Field(name="conv_rate", dtype=Float32),
        Field(name="acc_rate", dtype=Float32),
        Field(name="avg_daily_trips", dtype=Int64),
    ],
    source=driver_stats_source,
    online=True,
    tags={"owner": "ranking-team", "pii": "false"},
)
```

The `ttl` on a feature view is the point-in-time join window:
rows older than TTL will not be joined onto training labels
([Feast docs — Feature views](https://docs.feast.dev/getting-started/concepts/feature-view)).
We reuse `ttl` as the freshness SLA for serving.

### 2d. Feature service — the model's contract

A `FeatureService` names the exact set of features a given
model version consumes. This is the artifact you pin alongside
the model binary.

```python
# feature_services.py
from feast import FeatureService
from .feature_views import driver_stats_fv

driver_activity_v1 = FeatureService(
    name="driver_activity_v1",
    features=[driver_stats_fv],
    tags={"model": "driver-ranker", "version": "v1"},
)
```

`FeatureService` is documented in
[Feast docs — Feature services](https://docs.feast.dev/getting-started/concepts/feature-retrieval#feature-services).

### 2e. Offline (training) retrieval — point-in-time join

```python
from feast import FeatureStore
import pandas as pd

store = FeatureStore(repo_path="feature_repo")

entity_df = pd.DataFrame({
    "driver_id": [1001, 1002, 1003],
    "event_timestamp": pd.to_datetime([
        "2026-11-01 10:00:00",
        "2026-11-01 10:05:00",
        "2026-11-01 10:10:00",
    ]),
    "label": [1, 0, 1],
})

training_df = store.get_historical_features(
    entity_df=entity_df,
    features=store.get_feature_service("driver_activity_v1"),
).to_df()
```

This produces an as-of join: for each entity row, Feast returns
the latest feature value whose event timestamp is `<= event_timestamp`
and within the view's TTL ([Feast docs — Point-in-time joins](https://docs.feast.dev/getting-started/concepts/feature-retrieval#point-in-time-joins)).

### 2f. Materialize + online (serving) retrieval

Push the latest window of offline data into the online store:

```bash
feast apply
feast materialize-incremental $(date -u +"%Y-%m-%dT%H:%M:%S")
```

Then at serving time:

```python
features = store.get_online_features(
    features=store.get_feature_service("driver_activity_v1"),
    entity_rows=[{"driver_id": 1001}],
).to_dict()
```

The same `FeatureService` is used in both paths. That is the
train-serve-skew guard.

### 2g. Freshness-SLA alert

`feast materialize-incremental` records the last materialized
timestamp per view. A simple exporter reads that and emits a
Prometheus gauge of lag:

```python
# freshness_exporter.py
import time
from datetime import datetime, timezone
from feast import FeatureStore
from prometheus_client import Gauge, start_http_server

store = FeatureStore(repo_path="feature_repo")
lag = Gauge("feast_materialization_lag_seconds",
            "Seconds since last materialization", ["feature_view"])

def tick():
    now = datetime.now(tz=timezone.utc)
    for fv in store.list_feature_views():
        last = store.get_feature_view(fv.name).most_recent_end_time
        if last is not None:
            lag.labels(feature_view=fv.name).set((now - last).total_seconds())

if __name__ == "__main__":
    start_http_server(9000)
    while True:
        tick()
        time.sleep(30)
```

Alert when `feast_materialization_lag_seconds{feature_view="X"}`
exceeds the view's declared TTL. `most_recent_end_time` is a
property on the applied feature view
([Feast FeatureView API](https://docs.feast.dev/reference/feast-python-api-documentation#feast.feature_view.FeatureView)).

### 2h. Skew test (CI gate)

```python
# tests/test_skew.py
import pandas as pd
from feast import FeatureStore

def test_offline_online_parity():
    store = FeatureStore(repo_path="feature_repo")
    now = pd.Timestamp.utcnow()
    entity_df = pd.DataFrame({
        "driver_id": [1001],
        "event_timestamp": [now],
    })
    offline = store.get_historical_features(
        entity_df=entity_df,
        features=store.get_feature_service("driver_activity_v1"),
    ).to_df().iloc[0].to_dict()
    online = store.get_online_features(
        features=store.get_feature_service("driver_activity_v1"),
        entity_rows=[{"driver_id": 1001}],
    ).to_dict()
    for col in ["conv_rate", "acc_rate", "avg_daily_trips"]:
        assert offline[col] == online[col][0], f"skew on {col}"
```

The test can only pass when offline and online stores have both
been loaded with the same data — exactly the invariant we want
to protect.

## 3. Validation steps

1. `feast apply` — succeeds, prints the feature views and services
   it registered.
2. `feast feature-views list` — `driver_hourly_stats` appears with
   the expected TTL and owner tag.
3. `feast materialize-incremental <now>` — reports rows written to
   the online store.
4. Offline retrieval returns a training frame whose row count
   matches `len(entity_df)`.
5. Online retrieval returns the same per-entity values as the
   offline retrieval for the latest timestamp (skew test passes).
6. Freshness exporter publishes a gauge; stopping materialization
   causes the gauge to exceed `ttl` within one TTL window.
7. `pytest tests/` — passes, including the skew test, on a clean
   checkout.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Entities, data sources, feature views, and feature services are declared in separate modules, not inline | 10 |
| R2 | Each feature view has an explicit TTL justified by the business freshness need | 10 |
| R3 | Each feature view carries owner / PII / cost tags | 5 |
| R4 | The model's features are referenced via `FeatureService`, not by passing raw feature view lists at call sites | 10 |
| R5 | Offline retrieval uses `get_historical_features` with an `entity_df` carrying `event_timestamp` (point-in-time correct) | 15 |
| R6 | Online and offline paths use the same `FeatureService` name | 15 |
| R7 | Freshness SLA is observable (metric exported) and alerted when lag > TTL | 15 |
| R8 | A train/serve parity test exists and runs in CI | 15 |
| R9 | `feature_store.yaml` provider matches the deployment target; secrets are not committed | 5 |

Pass threshold: **80/100**. R5, R6, and R8 are must-haves — a
submission missing any of them is an auto-fail regardless of
total.

## 5. Common mistakes

1. **Feature logic in the serving layer, bypassing Feast.** Any
   transformation that happens at serve time but not at train
   time is a skew bug. The feature view / on-demand feature view
   is the only sanctioned place.
2. **TTL set so long it hides a broken pipeline.** A 30-day TTL
   on a feature that should refresh hourly means stale values
   still "satisfy" the join, silently.
3. **Different feature set between training and serving.** The
   symptom: offline accuracy good, online accuracy bad.
   Enforcing a single `FeatureService` prevents this.
4. **`feast apply` only in dev.** Registry drift between dev and
   prod means prod silently serves last month's feature definitions.
   `feast apply` belongs in CI/CD.
5. **Materializing by cron without monitoring.** The cron can
   fail and nobody notices until a model quality incident.
   Freshness SLA alerting is non-optional in production.
6. **Secrets in `feature_store.yaml`.** Database credentials go
   through environment variables / secret manager, not the
   registry config file.

## 6. References

Primary (official project documentation):

- Feast documentation — <https://docs.feast.dev/>
- Feast concepts (entities, feature views, feature services) —
  <https://docs.feast.dev/getting-started/concepts>
- Feast feature retrieval (point-in-time joins, services) —
  <https://docs.feast.dev/getting-started/concepts/feature-retrieval>
- Feast `feature_store.yaml` reference —
  <https://docs.feast.dev/reference/feature-repository/feature-store-yaml>
- Feast Python API — <https://docs.feast.dev/reference/feast-python-api-documentation>

Security and governance context:

- NIST AI Risk Management Framework (data governance / mapping
  of data sources is a MAP-function activity) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 (ML02: Data Poisoning
  motivates tight control of the write path into the feature
  store) — <https://owasp.org/www-project-machine-learning-security-top-10/>

Local context:

- Module-level rationale: `modules/mod-206-advanced-mlops/SOLUTION.md`
  (Decision 3 — "Feature store as the contract layer").
- Engineer-tier implementation reference: `engineer-solutions/mod-106`
  (per module README cross-reference).
