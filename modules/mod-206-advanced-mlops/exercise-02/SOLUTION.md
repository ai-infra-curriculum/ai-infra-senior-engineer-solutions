# SOLUTION — Lab 02: Advanced Experiment Tracking with MLflow

Reference solution for
[lab-02-experiment-tracking.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-206-advanced-mlops/exercises/lab-02-experiment-tracking.md).

The engineer-tier form of this exercise is "log parameters and
metrics to MLflow and register a model." The senior-tier
responsibility is to turn that into **lineage** — the chain from
training data version to code commit to registered model to
deployment — and to run the registry as a production system with
staged transitions and signed artifacts.

## 1. Solution overview

MLflow provides four components
([MLflow docs — Concepts](https://mlflow.org/docs/latest/introduction/index.html)):

- **Tracking** — runs, parameters, metrics, artifacts, tags.
- **Model Registry** — versioned models with stage transitions
  (via `aliases` and `tags` in modern MLflow; `Staging/Production`
  stages are deprecated as of MLflow 2.9 —
  [MLflow docs — Model Registry: Deprecated Model Stages](https://mlflow.org/docs/latest/model-registry.html#deprecated-model-stages)).
- **MLflow Models** — a packaging format with a `model signature`
  and `pyfunc` flavor.
- **Projects** — a reproducible run spec (less used in senior
  practice than SDK-driven runs).

Senior additions beyond the baseline:

1. **Deterministic lineage tags** on every run — data version,
   code commit, feature service version, dataset hash.
2. **Alias-based promotion** (`@champion`, `@challenger`) instead
   of deprecated stage transitions.
3. **Signed models** — a model signature captured at log time
   and validated at load time to catch schema drift.
4. **Nested runs** for hyperparameter sweeps so parent/child
   traceability survives.
5. **Server-backed tracking** (not local `mlruns/`), with
   artifacts in object storage and metadata in a SQL backend —
   the only shape that scales past one engineer.

## 2. Worked answer or implementation

### 2a. Tracking server shape

Local-dev `mlflow server` suffices for the lab; production
requires explicit backing stores
([MLflow docs — Tracking Server](https://mlflow.org/docs/latest/tracking/server.html)):

```bash
export MLFLOW_BACKEND_STORE_URI="postgresql+psycopg2://mlflow:***@postgres:5432/mlflow"
export MLFLOW_ARTIFACTS_DESTINATION="s3://mlflow-artifacts/"

mlflow server \
  --host 0.0.0.0 --port 5000 \
  --backend-store-uri "$MLFLOW_BACKEND_STORE_URI" \
  --artifacts-destination "$MLFLOW_ARTIFACTS_DESTINATION" \
  --serve-artifacts
```

Clients point at it with `MLFLOW_TRACKING_URI`.

### 2b. Lineage-rich training run

```python
import hashlib
import json
import subprocess
from pathlib import Path

import mlflow
import mlflow.sklearn
import pandas as pd
from mlflow.models import infer_signature
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import roc_auc_score
from sklearn.model_selection import train_test_split

mlflow.set_tracking_uri("http://mlflow:5000")
mlflow.set_experiment("driver-ranker")


def git_sha() -> str:
    return subprocess.check_output(
        ["git", "rev-parse", "HEAD"], text=True
    ).strip()


def dataset_hash(path: Path) -> str:
    h = hashlib.sha256()
    with path.open("rb") as f:
        for chunk in iter(lambda: f.read(1 << 20), b""):
            h.update(chunk)
    return h.hexdigest()


def train(data_path: Path, params: dict, feature_service: str):
    df = pd.read_parquet(data_path)
    X = df.drop(columns=["label", "event_timestamp"])
    y = df["label"]
    X_tr, X_te, y_tr, y_te = train_test_split(
        X, y, test_size=0.2, random_state=42
    )

    with mlflow.start_run() as run:
        # --- lineage tags -------------------------------------------------
        mlflow.set_tags({
            "code.commit": git_sha(),
            "data.path": str(data_path),
            "data.sha256": dataset_hash(data_path),
            "data.rows": len(df),
            "feature_service": feature_service,
            "owner": "ranking-team",
        })

        mlflow.log_params(params)

        # --- modern MLflow dataset lineage ---------------------------------
        dataset = mlflow.data.from_pandas(
            df, source=str(data_path), name="driver_ranker_train"
        )
        mlflow.log_input(dataset, context="training")

        model = GradientBoostingClassifier(**params).fit(X_tr, y_tr)
        preds = model.predict_proba(X_te)[:, 1]

        mlflow.log_metric("roc_auc", roc_auc_score(y_te, preds))

        # --- signed artifact -----------------------------------------------
        signature = infer_signature(X_tr, model.predict_proba(X_tr))
        mlflow.sklearn.log_model(
            sk_model=model,
            artifact_path="model",
            signature=signature,
            input_example=X_tr.head(5),
            registered_model_name="driver_ranker",
        )

        return run.info.run_id
```

`mlflow.data.from_pandas` + `log_input` registers the dataset as
a first-class lineage object
([MLflow docs — Tracking datasets](https://mlflow.org/docs/latest/tracking/data-api.html)).
`infer_signature` captures the input/output schema
([MLflow docs — Model Signatures](https://mlflow.org/docs/latest/model/signatures.html)),
and `log_model` auto-registers under `registered_model_name`.

### 2c. Nested runs for sweeps

```python
grid = [{"n_estimators": n, "max_depth": d}
        for n in (100, 200) for d in (3, 5, 7)]

with mlflow.start_run(run_name="sweep") as parent:
    mlflow.set_tag("sweep.strategy", "grid")
    best = None
    for params in grid:
        with mlflow.start_run(nested=True):
            run_id = train(Path("data/train.parquet"), params,
                           feature_service="driver_activity_v1")
            # parent sees children via mlflow search_runs(parent_run_id=...)
```

Nested runs are the documented pattern for sweeps
([MLflow docs — Parent and child runs](https://mlflow.org/docs/latest/tracking/tracking-api.html#parent-and-child-runs)).

### 2d. Alias-based registry promotion

```python
from mlflow.tracking import MlflowClient

client = MlflowClient()

def promote_to_challenger(model_name: str, version: int):
    client.set_registered_model_alias(model_name, "challenger", version)

def promote_to_champion(model_name: str, version: int):
    client.set_registered_model_alias(model_name, "champion", version)

def load_champion(model_name: str):
    return mlflow.pyfunc.load_model(f"models:/{model_name}@champion")
```

Aliases (`@champion`, `@challenger`) replace the deprecated
`Staging`/`Production` stages
([MLflow docs — Model Registry: Deploy and organize models with aliases and tags](https://mlflow.org/docs/latest/model-registry.html#deploy-and-organize-models-with-aliases-and-tags)).
A promotion is a two-step dance: challenger first (serving-side
canary pulls from `@challenger`), champion only after the canary
passes — which is the subject of
[exercise-04](../exercise-04/SOLUTION.md).

### 2e. Loader with signature enforcement

```python
import mlflow.pyfunc
from mlflow.models import validate_schema

def load_and_validate(model_name: str, alias: str, sample_df):
    uri = f"models:/{model_name}@{alias}"
    model = mlflow.pyfunc.load_model(uri)
    validate_schema(sample_df, model.metadata.get_input_schema())
    return model
```

`validate_schema` raises on a schema mismatch
([MLflow docs — Model Signatures: validate_schema](https://mlflow.org/docs/latest/model/signatures.html#validating-the-signature-of-a-model)).
Calling it at load time turns schema drift into a startup failure
instead of a runtime prediction bug.

## 3. Validation steps

1. `mlflow server --help` runs; starting with explicit
   `--backend-store-uri` and `--artifacts-destination` succeeds.
2. `python train.py` creates a run visible at the tracking URI;
   the run has all five lineage tags and a logged dataset.
3. The registered model has `version=1` and a signature with the
   expected columns and types.
4. `client.get_registered_model_alias("driver_ranker", "champion")`
   returns the version set by `promote_to_champion`.
5. `mlflow.pyfunc.load_model("models:/driver_ranker@champion")`
   loads without error; `load_and_validate` raises on a
   deliberately-dropped column.
6. A sweep run lists N child runs when queried with
   `search_runs(filter_string=f"tags.mlflow.parentRunId = '{parent}'")`
   ([MLflow docs — Search Runs](https://mlflow.org/docs/latest/search-runs.html)).
7. Shutting the tracking server down and restarting it with the
   same URIs preserves all runs, metrics, and aliases — i.e.
   state really is in the backing stores.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Tracking server uses explicit SQL backend and object-store artifacts; no local `mlruns/` in the reference | 10 |
| R2 | Every run logs `code.commit`, `data.sha256`, and `feature_service` tags | 15 |
| R3 | `mlflow.log_input` is used — datasets appear as first-class lineage, not just params | 10 |
| R4 | `log_model` is called with `signature=` and `input_example=` | 15 |
| R5 | Promotion uses aliases (`@champion`, `@challenger`); deprecated stages are not used | 15 |
| R6 | Sweeps use nested runs (parent + children), not flat runs with magic tags | 10 |
| R7 | A loader validates the input DataFrame against `model.metadata.get_input_schema()` before serving | 15 |
| R8 | Tracking URI is injected via env var; no hard-coded localhost in library code | 5 |
| R9 | The run can be reproduced end-to-end from the logged tags (commit + dataset hash + params) | 5 |

Pass threshold: **80/100**. R2 and R4 are must-haves — a run
without lineage tags or without a signature has failed the
senior bar regardless of overall score.

## 5. Common mistakes

1. **Logging to the default local tracking URI.** Everyone's runs
   go to `./mlruns/` and are invisible to the rest of the team.
   Set `MLFLOW_TRACKING_URI` in the training image.
2. **Using deprecated stage transitions.** `transition_model_version_stage`
   still works but is deprecated. Writing new code against it
   locks the team into migration work.
3. **No model signature.** Ten versions in, nobody remembers the
   exact feature order; a mis-ordered call silently returns
   garbage probabilities.
4. **Tags that drift from the code.** `code.commit` hard-coded or
   read from `os.environ` in a way that isn't populated in CI →
   tag is literally the string `"unknown"` for every run. Set
   it at module import time with a subprocess call.
5. **Hyperparameter sweeps as flat runs.** No parent, N siblings;
   the "sweep" is unrecoverable once the notebook is closed. Use
   `mlflow.start_run(nested=True)`.
6. **Artifacts bigger than Postgres likes.** Model pickles going
   through the SQL backend instead of object storage. Always set
   `--artifacts-destination` + `--serve-artifacts` (or configure
   per-experiment artifact locations).
7. **Treating the registry as write-only.** Senior-tier use means
   *reading from* the registry at serve time, keyed on an alias —
   not re-downloading a specific version by number.

## 6. References

Primary (official project documentation):

- MLflow documentation — <https://mlflow.org/docs/latest/>
- MLflow Tracking — <https://mlflow.org/docs/latest/tracking.html>
- MLflow Tracking Server — <https://mlflow.org/docs/latest/tracking/server.html>
- MLflow Model Registry — <https://mlflow.org/docs/latest/model-registry.html>
- MLflow Model Signatures — <https://mlflow.org/docs/latest/model/signatures.html>
- MLflow dataset tracking (`mlflow.data`) — <https://mlflow.org/docs/latest/tracking/data-api.html>
- MLflow search runs (filter syntax, parent/child) — <https://mlflow.org/docs/latest/search-runs.html>

Security and governance context:

- NIST AI Risk Management Framework — MANAGE 4.1 covers
  documenting the training data, model version, and deployment
  context, which is what lineage tags implement —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- MITRE ATLAS — the ML Supply Chain Compromise technique
  motivates signed artifacts and auditable lineage —
  <https://atlas.mitre.org/>

Local context:

- Module rationale: `modules/mod-206-advanced-mlops/SOLUTION.md`
  (Decision 5 — "Model lineage tracked, not just experiments").
- Serving-side consumer of this registry: [exercise-04](../exercise-04/SOLUTION.md).
- Engineer-tier implementation reference: `engineer-solutions/mod-106`.
