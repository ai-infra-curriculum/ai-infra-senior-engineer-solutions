# SOLUTION — Lab 02: GitOps with Argo CD

Reference solution for
[lab-02-gitops-argocd.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-208-iac-gitops/exercises/lab-02-gitops-argocd.md).

Engineer-tier GitOps mechanics mirror `engineer-solutions/mod-109`.
Senior scope is the *program*: bootstrapping clusters from a
single root application, promoting changes safely across
environments, and making drift observable instead of invisible.

## 1. Solution overview

GitOps treats Git as the source of truth for cluster desired
state. Argo CD is a declarative continuous-delivery agent that
runs *inside* the cluster, pulls manifests from Git, and
reconciles live state against the declared state
([Argo CD — What Is Argo CD](https://argo-cd.readthedocs.io/en/stable/)
and [Core Concepts](https://argo-cd.readthedocs.io/en/stable/core_concepts/)).

Four Argo CD primitives cover the reference design:

- **Application** — a desired-state tracker for a set of
  manifests from a Git path.
- **ApplicationSet** — a template that expands into many
  Applications from a generator (list / Git directory /
  cluster list) so fleet growth doesn't mean handwriting YAML
  ([ApplicationSet Controller](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)).
- **Project (`AppProject`)** — a boundary of what sources,
  destinations, and resources a set of Applications may use
  ([Projects](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#projects)).
- **Sync-waves and hooks** — ordered reconciliation when
  resources depend on each other
  ([Sync Phases and Waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)).

Senior additions beyond "a working Application":

- **App-of-apps** bootstrap so cluster recreation is one
  `kubectl apply` of a root Application
  ([Cluster Bootstrapping — App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)).
- **Environment promotion** from `dev` → `staging` → `prod` by
  Git commit (and PR), not `argocd app sync --revision`.
- **Different sync policies per environment** (auto in dev,
  manual + approval for prod)
  ([Automated Sync Policy](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)).
- **Drift observability** — treat `OutOfSync` duration as a
  monitored metric, not a UI color.

## 2. Worked answer or implementation

### 2a. Repository layout — one env dir per environment

Follow the layout in
[Argo CD declarative setup — Applications](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications)
and the app-of-apps bootstrap pattern:

```
gitops/
├── bootstrap/
│   └── root-app.yaml            # the one Application you apply by hand
├── projects/
│   └── ml-platform.yaml         # AppProject: boundaries
├── apps/                        # one Application manifest per workload x env
│   ├── dev/
│   │   ├── model-serving.yaml
│   │   └── feature-store.yaml
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
└── manifests/                   # the actual K8s YAML / Helm / Kustomize
    ├── model-serving/
    │   ├── base/                # Kustomize base
    │   └── overlays/{dev,staging,prod}/
    └── feature-store/
```

### 2b. The `AppProject` — explicit boundaries

`AppProject` constrains what sources and destinations are
reachable and which namespaced/cluster-scoped resource kinds
are allowed. Without it you have effectively no isolation
between teams sharing an Argo CD instance.

```yaml
# projects/ml-platform.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ml-platform
  namespace: argocd
spec:
  description: ML platform workloads (serving, feature store, training)
  sourceRepos:
    - https://github.com/acme/gitops-ml-platform.git
  destinations:
    - namespace: ml-*
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
  roles:
    - name: ml-platform-admin
      policies:
        - p, proj:ml-platform:ml-platform-admin, applications, *, ml-platform/*, allow
      groups:
        - acme:ml-platform
```

Project schema and RBAC format: [AppProject spec](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#projects)
and [RBAC configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/).

### 2c. The root Application — app-of-apps bootstrap

The root Application points at `apps/<env>`, and every file in
that directory is itself an `Application`. Applying the root
by hand (once) is the only imperative step; from then on every
change is a Git commit.

```yaml
# bootstrap/root-app.yaml  (applied once with `kubectl apply -f`)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/acme/gitops-ml-platform.git
    targetRevision: HEAD
    path: apps/prod                # change to apps/dev on a dev cluster
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Pattern reference: [Cluster Bootstrapping — App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/).
The `finalizers` entry ensures Argo CD cleans up child
resources if the root Application is deleted
([App Deletion / Finalizer](https://argo-cd.readthedocs.io/en/stable/user-guide/app_deletion/)).

### 2d. A workload Application — Kustomize overlay per env

```yaml
# apps/prod/model-serving.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: model-serving-prod
  namespace: argocd
spec:
  project: ml-platform
  source:
    repoURL: https://github.com/acme/gitops-ml-platform.git
    targetRevision: HEAD
    path: manifests/model-serving/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: ml-serving
  syncPolicy:
    # NOTE: no `automated:` block — prod requires manual sync
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 3m
```

Dev's version of the same file includes an `automated: { prune:
true, selfHeal: true }` block ([Automated Sync Policy](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)).
This is the mechanical difference between "ship-fast" and
"human-must-approve" environments.

### 2e. ApplicationSet — fleet scaling without handwritten YAML

Once a second cluster or a third microservice appears, stop
copy-pasting Application YAML. Use an
[ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
with a generator.

```yaml
# bootstrap/model-serving-appset.yaml — one App per cluster
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: model-serving
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: prod
  template:
    metadata:
      name: 'model-serving-{{name}}'
    spec:
      project: ml-platform
      source:
        repoURL: https://github.com/acme/gitops-ml-platform.git
        targetRevision: HEAD
        path: manifests/model-serving/overlays/prod
      destination:
        server: '{{server}}'
        namespace: ml-serving
```

Each registered cluster labeled `env=prod` gets a matching
Application generated automatically.

### 2f. Sync-waves for ordered resources

When CRDs must exist before custom resources, or a secret must
exist before the Deployment that mounts it, annotate with
`argocd.argoproj.io/sync-wave`
([Sync Phases and Waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)).

```yaml
# manifests/model-serving/base/crd.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-10"
---
# manifests/model-serving/base/operator-deploy.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-5"
---
# manifests/model-serving/base/custom-resource.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
```

Lower wave numbers sync first. Resources within the same wave
sync in parallel.

### 2g. Environment promotion = a Git PR

The **only** way to move a change from staging to prod is a PR
against `manifests/.../overlays/prod/`. Concretely:

1. Developer opens a PR bumping the image tag in the `staging`
   overlay. CI, Argo CD in dev syncs, soak period passes.
2. A second PR copies the tag change into the `prod` overlay.
   Required reviewers approve.
3. Merge triggers Argo CD in prod to detect `OutOfSync`; a
   human presses sync (or an approved pipeline calls
   `argocd app sync`).

Promotion-by-PR is the rollback mechanism, too: `git revert`
on the overlay restores the prior image tag and Argo CD
reconciles.

### 2h. Make drift observable

`OutOfSync` is a UI state in Argo CD; for alerting, scrape the
Prometheus metrics the application controller exposes
([Metrics](https://argo-cd.readthedocs.io/en/stable/operator-manual/metrics/)).
The relevant series is `argocd_app_info` labeled by
`sync_status`; alert on duration.

```yaml
# PrometheusRule (abbreviated)
groups:
  - name: argocd
    rules:
      - alert: ArgoCDAppOutOfSyncTooLong
        expr: |
          max by (name, dest_namespace, sync_status) (
            argocd_app_info{sync_status="OutOfSync"}
          ) == 1
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.name }} has been OutOfSync for 30m"
```

Metric name and labels: [Argo CD metrics reference](https://argo-cd.readthedocs.io/en/stable/operator-manual/metrics/).

## 3. Validation steps

1. `kubectl apply -f bootstrap/root-app.yaml` creates the
   `root` Application; its children appear in `kubectl get
   applications -n argocd` within one reconcile cycle.
2. The `AppProject` enforces its allow-list: changing a child
   Application's `repoURL` to a non-allowed repo makes the
   Application go `Unknown`/`Degraded`.
3. Pushing a commit to `manifests/model-serving/overlays/dev/`
   is reflected in the dev cluster without a human running
   `argocd app sync` (auto-sync enabled).
4. Pushing the same change under `overlays/prod/` leaves prod
   `OutOfSync` until a human clicks sync (auto-sync disabled).
5. A manual `kubectl edit deploy` against a managed resource is
   reverted within one reconcile cycle in environments with
   `selfHeal: true`; in environments without, it surfaces as
   `OutOfSync` on the application dashboard.
6. Deleting a resource from Git causes it to be pruned on the
   next sync when `prune: true` is set, and surfaces as a sync
   warning when prune is off (default).
7. A sync-waved deployment applies CRDs before the custom
   resources that depend on them; reversing the annotations
   reproduces the ordering bug.
8. The Prometheus `ArgoCDAppOutOfSyncTooLong` alert fires in a
   test by deliberately diverging a test cluster from Git for
   longer than the `for:` window.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | A single root Application bootstraps the cluster (app-of-apps) | 10 |
| R2 | At least one `AppProject` restricts `sourceRepos` and `destinations` per team or workload class | 10 |
| R3 | Environment overlays separate dev / staging / prod; no environment-conditional logic inside base manifests | 10 |
| R4 | Dev environments enable `automated.prune` + `automated.selfHeal`; prod does not auto-sync | 15 |
| R5 | Promotions are PRs against higher-env overlays, never live `argocd app create` | 15 |
| R6 | Fleet growth uses `ApplicationSet` with a generator, not copy-pasted `Application` YAML | 10 |
| R7 | Resources with ordering constraints use `sync-wave` annotations | 5 |
| R8 | Argo CD Prometheus metrics are scraped and an `OutOfSync` duration alert exists | 15 |
| R9 | Secrets are not stored in plain YAML in the Git repo (sealed-secrets / external-secrets; see exercise-04) | 10 |

Pass threshold: **80/100**. R5, R4, and R9 are must-haves — a
GitOps setup that accepts prod changes outside Git, auto-syncs
to prod without approval, or commits raw secrets is not safe
regardless of total.

## 5. Common mistakes

1. **`kubectl apply` or Helm upgrades directly against a
   managed cluster.** Every out-of-band change produces an
   `OutOfSync` that drowns real signal. If something needs to
   change, it changes in Git.
2. **Auto-sync enabled on prod without a review gate.** A PR
   merged to `main` becomes a prod deploy within seconds. Keep
   prod manual (`syncPolicy.automated` omitted) or gate via
   `syncPolicy.syncOptions: Validate=true` plus a required PR
   review and a separate pipeline that calls `argocd app sync`.
3. **One giant `Application` for everything.** Loses the
   ability to pause, rollback, or inspect a single workload.
   One Application per deliverable per env.
4. **ApplicationSet with `list` generator for 50 services.**
   Defeats the point. Use `git` or `cluster` generators so new
   services are picked up by directory convention.
5. **Prune never enabled.** Deleting manifests from Git doesn't
   remove the resource from the cluster; drift accumulates.
   Decide per environment, but default `prune: true` in dev.
6. **No alerting on `OutOfSync` duration.** The dashboard turns
   yellow and nobody sees it. Prometheus alert, not a visual
   state.
7. **Secrets in Git overlays.** Even with private repos, a
   commit is forever. Use sealed-secrets or external-secrets
   (see exercise-04).

## 6. References

Primary (official project documentation):

- Argo CD — Overview — <https://argo-cd.readthedocs.io/en/stable/>
- Argo CD — Core Concepts — <https://argo-cd.readthedocs.io/en/stable/core_concepts/>
- Argo CD — Declarative Setup (Application, AppProject) —
  <https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/>
- Argo CD — Cluster Bootstrapping (App of Apps) —
  <https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/>
- Argo CD — ApplicationSet Controller —
  <https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/>
- Argo CD — Sync Phases and Waves —
  <https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/>
- Argo CD — Automated Sync Policy —
  <https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/>
- Argo CD — RBAC configuration —
  <https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/>
- Argo CD — Metrics —
  <https://argo-cd.readthedocs.io/en/stable/operator-manual/metrics/>
- Argo CD — App Deletion / Finalizers —
  <https://argo-cd.readthedocs.io/en/stable/user-guide/app_deletion/>

Security / governance context:

- NIST AI Risk Management Framework (MANAGE function —
  continuous monitoring and corrective action over time)
  — <https://www.nist.gov/itl/ai-risk-management-framework>

Local context:

- Module-level rationale: `modules/mod-208-iac-gitops/SOLUTION.md`
  (Decisions 1, 2, 5 — Argo CD over Flux, app-of-apps,
  pull-based deploys).
- Engineer-tier GitOps implementation reference:
  `engineer-solutions/mod-109` (per module README cross-reference).
