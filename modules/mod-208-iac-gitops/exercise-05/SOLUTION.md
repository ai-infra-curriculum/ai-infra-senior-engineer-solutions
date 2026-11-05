# SOLUTION — Lab 05: Platform IaC + GitOps Program

Reference solution for
[lab-05-platform-iac-program.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-208-iac-gitops/exercises/lab-05-platform-iac-program.md).

This is the capstone design exercise. The other four labs built
pieces — Terraform modules, Argo CD, testing, secrets. This one
is about owning them as a *program*: the policies, roles,
rollout plan, and operating model that makes the pieces a
platform rather than a collection.

## 1. Solution overview

A platform IaC + GitOps program is the set of agreements and
controls that let many teams ship infrastructure safely through
a shared pipeline. It has four layers:

1. **Source-of-truth model** — what lives in which repo; who
   owns each; how PRs flow from service teams to the platform.
2. **Policy as code** — the controls that run in CI and
   admission (Conftest / OPA Gatekeeper / Kyverno
   — [Kyverno overview](https://kyverno.io/docs/introduction/)).
3. **Promotion and rollback** — how a change moves `dev` →
   `staging` → `prod`, how it reverses, and who signs off.
4. **Operations** — observability of the pipeline itself,
   drift detection, on-call ownership.

The deliverables for this exercise are therefore documents and
decisions, not code. Code examples below are small artifacts
that make each decision concrete (a `CODEOWNERS` entry, a
single Kyverno policy, one PromQL alert).

## 2. Worked answer or implementation

### 2a. Repository topology (decision + rationale)

Two canonical shapes; pick one and commit.

| | Mono-repo | Multi-repo |
|---|---|---|
| Change discoverability | One place to grep | Must know which repo |
| Blast radius of a bad merge | Larger | Contained |
| CODEOWNERS granularity | File-level (works but noisy) | Repo-level (clean) |
| Argo CD sources | One `repoURL` | Many |
| Access control | Mono-repo must stay unified | Per-repo RBAC |

**Reference decision:** multi-repo, with three tiers —
`platform-modules/` (reusable Terraform modules),
`platform-infra/` (environment root configs that compose
modules), `gitops/<team>/` (per-team manifest repos pulled by
Argo CD). The **why**: the three tiers have different change
velocity, different review audiences, and different access
patterns. A mono-repo collapses them into one review queue and
one access boundary.

`CODEOWNERS` encodes the review requirement
([GitHub — About code owners](https://docs.github.com/en/repositories/managing-your-repositories-settings-and-security/customizing-your-repository/about-code-owners)):

```
# platform-modules/CODEOWNERS
*                                 @acme/platform-infra
/modules/gpu-training-cluster/    @acme/platform-infra @acme/ml-platform
```

Branch protection ([protected branches docs](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches))
requires:

- At least 1 code-owner approval.
- Signed commits ([commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification)).
- Linear history (no merge commits in the main branch).
- Required status checks: static → policy → unit (from
  exercise-03).

### 2b. RACI — who does what

This is the senior deliverable: an explicit RACI so the
program has owners, not volunteers.

| Activity | Platform infra | Service team | Security | Argo CD on-call |
|---|---|---|---|---|
| Add new Terraform module | R, A | C | C | I |
| Change module API (breaking) | R, A | I | I | I |
| Add a workload Application | C | R, A | I | I |
| Promote to prod (GitOps PR) | I | R, A | I | I |
| Define a new Kyverno policy | C | I | R, A | I |
| Rotate Sealed Secrets key | R, A | I | C | C |
| Investigate `OutOfSync` alert | C | R, A | I | — |
| Argo CD upgrade | R, A | I | I | C |
| Access-grant to a new `AppProject` | R, A | I | C | I |

Legend: R = Responsible, A = Accountable, C = Consulted,
I = Informed.

### 2c. Policy catalog (minimum viable)

The platform runs the following policies in CI *and* at
admission. Enforcing only in CI lets anyone with `kubectl`
bypass them; enforcing only in admission produces failures too
late in the loop. Run both ([Kyverno — Policies and Rules](https://kyverno.io/docs/kyverno-policies/)).

| ID | Rule | Rationale |
|---|---|---|
| POL-001 | All resources carry `owner`, `cost_center`, `env` labels | Billing / on-call / blast-radius routing |
| POL-002 | No plain `Secret` manifests (see exercise-04) | Prevent credential commits |
| POL-003 | No `hostNetwork: true` without allow-list exemption | Network blast-radius |
| POL-004 | Container images come from the approved registry | Supply chain (ML06 of OWASP ML Top 10) |
| POL-005 | PVCs declare `storageClassName` explicitly | Avoid default-class drift between clusters |
| POL-006 | No `latest` image tag on Deployments | Reproducibility |
| POL-007 | Terraform plan must not destroy stateful resources without `-target` + explicit approval | Data-loss guard |

Example Kyverno rule for POL-006:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-image-tag
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Using 'latest' image tag is forbidden."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

Reference policy format: [Kyverno — Validate rules](https://kyverno.io/docs/writing-policies/validate/).

### 2d. Promotion model

```
dev overlay      staging overlay     prod overlay
   │                   │                  │
   │  auto-sync        │  auto-sync       │  manual sync
   │  self-heal        │  no selfHeal     │  2 required approvers
   ▼                   ▼                  ▼
dev cluster        staging cluster    prod cluster
```

Concretely:

- `dev` and `staging` overlays have Argo CD `automated.prune +
  selfHeal` (exercise-02 §2d).
- `prod` overlay omits `automated.*`. Sync is triggered by a
  release pipeline that calls `argocd app sync` after PR merge
  and approval.
- Promotion is always a PR copying the *tag* (or
  Kustomize-rendered field) from the lower-env overlay to the
  higher-env overlay. Image rebuilds don't happen during
  promotion.
- Rollback is `git revert` on the overlay. No special tooling.

**Soak time.** The pipeline enforces a minimum soak between
envs: 30 min in `staging` after `dev`, 60 min in `prod` after
`staging`, both overridable with a documented break-glass
label on the PR. Soak is measured against the staging
cluster's error-budget burn rate
([Google SRE book — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)).

### 2e. Operate the pipeline itself

The pipeline is a production service. It has SLOs. A
non-exhaustive set:

| SLI | SLO | Owner |
|---|---|---|
| Argo CD application controller availability | 99.9% monthly | Argo CD on-call |
| Time from merge to `Synced` in dev | p95 < 2 min | Platform infra |
| `OutOfSync` duration above 30 min | 0 per week | Argo CD on-call |
| Policy check wall-clock (CI) | p95 < 2 min | Platform infra |

Alerts (exercise-02 §2h plus):

```yaml
- alert: ArgoCDAppControllerReconcileSlow
  expr: |
    histogram_quantile(0.95,
      sum by (le) (rate(argocd_app_reconcile_bucket[5m]))
    ) > 300
  for: 15m
  labels: { severity: warning }
  annotations:
    summary: "Argo CD reconcile p95 > 5m — pipeline slow"
```

Metric names: [Argo CD metrics reference](https://argo-cd.readthedocs.io/en/stable/operator-manual/metrics/).

### 2f. Rollout plan

A 90-day adoption plan for a mid-size org, used as a template
in the solution — concrete dates are the student's.

| Phase | Weeks | Deliverables |
|---|---|---|
| 1. Foundations | 1–3 | Repo topology + CODEOWNERS; `terraform-aws-*` module scaffolds; Argo CD installed via GitOps; Sealed Secrets or ESO installed; first dev cluster bootstrap |
| 2. Policy + CI | 4–6 | Policy catalog agreed; Conftest + Kyverno enforcing POL-001/002/006 non-blocking; dashboards for violations |
| 3. First workload | 7–9 | One service team onboarded end-to-end, dev → staging. Policy violations now blocking in that service's repo |
| 4. Prod | 10–12 | Manual-sync prod environment live; two services in prod; policies blocking everywhere |
| 5. Expand | 13+ | Service team self-service via `AppProject` template; platform-infra reduced to module authorship + reviews |

The rollout order is intentional: policy *before* prod. A
policy added after prod workloads exist is a migration project.

### 2g. Break-glass procedure

Documented, limited, auditable.

1. **Trigger.** On-call declares a SEV1 and a reason that
   cannot wait for a PR (e.g. revoking a leaked token in prod
   now).
2. **Access.** A time-limited (4h) role grant via the identity
   provider, logged to SIEM.
3. **Action.** `kubectl apply` or `argocd app sync --revision`
   is allowed. Every command is logged.
4. **Reconcile within 24h.** The out-of-band change is
   reflected in Git as a follow-up PR. Argo CD must be `Synced`
   again within 24 hours of the incident resolution.
5. **Post-mortem.** The break-glass invocation is reviewed; if
   the same root cause recurs, a policy or runbook change is
   required.

## 3. Validation steps

(For a design exercise, "validation" is reviewing the
artifacts against the requirements.)

1. Repo topology and ownership model are documented; a new
   engineer can find who approves a PR to any path by looking
   at `CODEOWNERS`.
2. RACI covers every activity listed in §2b; no activity is
   `A` in more than one column.
3. At least one Kyverno (or OPA Gatekeeper) policy from the
   catalog is enforced in a test cluster, and a deliberately
   violating manifest is rejected at admission.
4. The same policy is also enforced in CI (via Conftest or the
   Kyverno CLI) and rejects the same violation on PR.
5. A PR copying an image tag from `staging` to `prod` overlay
   requires two reviewers and leaves prod `OutOfSync` until
   the release pipeline runs sync.
6. `git revert` of that PR causes Argo CD to roll back on the
   next sync.
7. Prometheus alerts for `OutOfSync > 30m` and controller
   reconcile p95 > 5m are deployed and tested by inducing
   failure.
8. Break-glass procedure is documented with concrete commands
   and a log-retention statement.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Repo topology decision (mono vs multi) is made and justified in writing | 10 |
| R2 | `CODEOWNERS` and branch protection are configured; signed commits required on protected branches | 10 |
| R3 | RACI matrix covers at least: new module, module breaking change, prod promotion, secret rotation, policy addition, Argo CD upgrade | 10 |
| R4 | A policy catalog of at least 5 rules is documented, each with ID, rule, rationale | 10 |
| R5 | At least one policy is enforced *both* in CI and at admission | 15 |
| R6 | Promotion model specifies per-env sync policy (auto vs manual), minimum soak, and rollback mechanism | 15 |
| R7 | At least three pipeline SLOs are defined with owner + alert | 10 |
| R8 | A time-sequenced rollout plan exists and sequences policy before prod | 10 |
| R9 | Break-glass procedure specifies trigger, access path, logging, reconcile deadline, post-mortem requirement | 10 |

Pass threshold: **80/100**. R5, R6, and R9 are must-haves — a
program without admission-side enforcement, a documented
promotion path, or a break-glass procedure is not a
production-ready program regardless of total.

## 5. Common mistakes

1. **Copying a successful team's structure without the
   policies.** The structure (two layers of repos, four
   policies, an on-call) is load-bearing. Importing only the
   repo layout is cargo-culting.
2. **Policy added after prod.** Every existing workload is now
   non-compliant. Start with `Audit` mode, get the violations
   to zero, then flip to `Enforce`.
3. **CODEOWNERS that defaults everything to one team.** The
   team becomes the bottleneck. Scope ownership by directory
   so service teams own their own manifests.
4. **Promotion by re-running CI in a higher env.** Produces a
   different image per environment; defeats
   image-is-immutable-artifact. Promote the *same* image tag,
   not a rebuild.
5. **Break-glass that nobody rehearses.** The first time it's
   used is during an incident, and the documentation is
   wrong. Dry-run quarterly.
6. **SLOs on availability but not pipeline latency.** A
   pipeline that's up but takes an hour to deploy is
   unusable; still green on the dashboard. Measure both.
7. **No admission-side enforcement — "CI is enough".** Anyone
   with cluster access can bypass CI. Policy belongs on both
   sides of the pipe.
8. **Platform team owns everything in RACI.** Nothing scales.
   Push `A` for workload-level activities onto the owning
   service team; platform is `A` for the platform.

## 6. References

Primary (official project documentation):

- Argo CD — Operator manual (bootstrapping, RBAC, metrics) —
  <https://argo-cd.readthedocs.io/en/stable/operator-manual/>
- Kyverno — Introduction —
  <https://kyverno.io/docs/introduction/>
- Kyverno — Writing policies (validate) —
  <https://kyverno.io/docs/writing-policies/validate/>
- Open Policy Agent — Gatekeeper —
  <https://open-policy-agent.github.io/gatekeeper/website/>
- Conftest — <https://www.conftest.dev/>
- GitHub — About code owners —
  <https://docs.github.com/en/repositories/managing-your-repositories-settings-and-security/customizing-your-repository/about-code-owners>
- GitHub — About protected branches —
  <https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches>
- GitHub — Commit signature verification —
  <https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification>
- Google SRE workbook — Alerting on SLOs —
  <https://sre.google/workbook/alerting-on-slos/>
- Argo CD — Metrics reference —
  <https://argo-cd.readthedocs.io/en/stable/operator-manual/metrics/>

Security / governance context:

- NIST AI Risk Management Framework (GOVERN function —
  structures, roles, and accountability; MANAGE function —
  response to identified risks over time) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 (ML06: AI Supply
  Chain Attacks — approved-registry and signed-commit
  controls address this directly) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS (adversary techniques map to GitOps controls:
  development-environment access, trusted-image abuse,
  pipeline poisoning) — <https://atlas.mitre.org/>

Local context:

- Module-level rationale: `modules/mod-208-iac-gitops/SOLUTION.md`
  (all six decisions; this exercise is the programmatic frame
  around them).
- Prior labs supplying the pieces: exercise-01 (modules),
  exercise-02 (Argo CD topology), exercise-03 (test pyramid),
  exercise-04 (secrets).
- Engineer-tier IaC implementation reference:
  `engineer-solutions/mod-109` (per module README cross-reference).
