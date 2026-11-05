# SOLUTION — Lab 03: Infrastructure Testing

Reference solution for
[lab-03-infrastructure-testing.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-208-iac-gitops/exercises/lab-03-infrastructure-testing.md).

Infrastructure testing is not one activity; it is a pyramid of
cheap-static, medium-dynamic, and expensive-integration checks
that match the shape of software testing. The senior
responsibility is getting the layers right so the slow tests
aren't guarding problems the fast tests could have caught.

## 1. Solution overview

Four complementary layers, cheapest first
([Terraform — Testing concepts](https://developer.hashicorp.com/terraform/language/tests)):

| Layer | What it catches | Tool(s) |
|---|---|---|
| L1 Static | Syntax, formatting, dead code, obvious misuses | `terraform validate`, `terraform fmt`, `tflint` |
| L2 Policy | Forbidden resource shapes (public bucket, missing tag) | `OPA / Conftest`, `Checkov` |
| L3 Unit / plan tests | Module behaves as declared, given inputs | `terraform test` (`.tftest.hcl`) in plan mode |
| L4 Integration | Real resources come up, API responds, teardown clean | `terraform test` apply mode, Terratest, kind + Argo CD |

The pyramid applies the same discipline as software testing: run
the cheap, deterministic layers on every commit; run the
expensive ones on PR merge or nightly.

Argo CD manifests also need testing: `kubectl apply --dry-run`,
Kustomize/Helm render diff, and a kind-cluster smoke test
confirming the manifests sync to a real (ephemeral) cluster.

## 2. Worked answer or implementation

### 2a. L1 — Static checks

In CI, first and always:

```bash
terraform fmt -check -recursive
terraform validate                # per init'd config
tflint --recursive
```

`terraform fmt` enforces canonical style ([fmt](https://developer.hashicorp.com/terraform/cli/commands/fmt));
`terraform validate` checks internal consistency without a
backend ([validate](https://developer.hashicorp.com/terraform/cli/commands/validate));
`tflint` flags provider-specific anti-patterns (invalid instance
types, deprecated args) ([tflint docs](https://github.com/terraform-linters/tflint)).

All three are seconds-fast and run on every push.

### 2b. L2 — Policy as code

Policies are organizational rules that `validate` cannot express:
"no S3 bucket may be `public_access_block` disabled", "every
resource must have an `owner` tag", "no EC2 may have a public
IP in the `prod` workspace". Encode them as code; evaluate
against the `terraform plan` JSON.

Using OPA + Conftest ([OPA Terraform tutorial](https://www.openpolicyagent.org/docs/latest/terraform/),
[Conftest](https://www.conftest.dev/)):

```bash
terraform plan -out=plan.bin
terraform show -json plan.bin > plan.json
conftest test plan.json --policy policies/
```

```rego
# policies/tags.rego
package main

deny[msg] {
    resource := input.resource_changes[_]
    not resource.change.after.tags.owner
    msg := sprintf("%s missing required 'owner' tag", [resource.address])
}

deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket_public_access_block"
    resource.change.after.block_public_acls == false
    msg := sprintf("%s disables block_public_acls", [resource.address])
}
```

Checkov is a batteries-included alternative with a built-in
policy library ([Checkov docs](https://www.checkov.io/)). Run
one or the other, not both.

### 2c. L3 — Native `terraform test` in plan mode

`terraform test` (Terraform 1.6+) runs `.tftest.hcl` files
against the module ([Terraform Tests](https://developer.hashicorp.com/terraform/language/tests)).
Plan-mode tests are fast because no real resources are
provisioned.

```hcl
# tests/outputs.tftest.hcl
variables {
  cluster_name       = "ml-ci-01"
  subnet_ids         = ["subnet-aaa", "subnet-bbb"]
  gpu_instance_types = ["g5.2xlarge"]
  tags = {
    owner       = "ml-platform"
    cost_center = "ai-research"
  }
}

run "produces_node_group_arn" {
  command = plan

  assert {
    condition     = output.node_group_arn != null
    error_message = "node_group_arn must exist in plan."
  }
}

run "rejects_untagged" {
  command = plan
  variables {
    tags = { cost_center = "ai-research" }   # missing 'owner'
  }
  expect_failures = [var.tags]
}
```

`expect_failures` is the native way to test that a validation
block rejects the right inputs. These tests run in CI alongside
the static checks.

### 2d. L4 — Integration tests against real cloud

Integration tests spin up actual resources in a scratch
account, assert on behavior, and tear down. Two valid stacks:

**Native `terraform test` in `apply` mode**
([Writing tests — apply](https://developer.hashicorp.com/terraform/language/tests#test-modes)):

```hcl
# tests/apply.tftest.hcl
run "deploy_minimal" {
  command = apply

  assert {
    condition     = length(output.node_group_arn) > 0
    error_message = "apply must produce a node group."
  }
}
```

**Terratest** (Go) for cases where you need to assert on the
real cloud API, not just Terraform outputs ([Terratest docs](https://terratest.gruntwork.io/docs/)):

```go
// tests/integration_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestGPUTrainingCluster(t *testing.T) {
    opts := &terraform.Options{
        TerraformDir: "../examples/minimal",
        Vars: map[string]interface{}{
            "cluster_name": "ml-ci-" + t.Name(),
        },
    }
    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    arn := terraform.Output(t, opts, "node_group_arn")
    assert.NotEmpty(t, arn)
}
```

The `defer terraform.Destroy` is non-optional: an integration
test that doesn't guarantee teardown leaks cloud spend.

### 2e. Kubernetes manifest testing — kind + Argo CD smoke

For the GitOps manifests from exercise-02, test that they
actually sync. A small shell script around `kind` and Argo CD
is enough; this runs in CI on manifest changes
([kind — Quick start](https://kind.sigs.k8s.io/docs/user/quick-start/)).

```bash
#!/usr/bin/env bash
set -euo pipefail

kind create cluster --name gitops-smoke
kubectl create namespace argocd
kubectl apply -n argocd \
    -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server --timeout=5m

# Point root Application at the PR's manifests
kubectl apply -f bootstrap/root-app.yaml
# Wait for sync
for i in $(seq 1 30); do
  status=$(kubectl get app root -n argocd -o jsonpath='{.status.sync.status}')
  [[ "$status" == "Synced" ]] && exit 0
  sleep 10
done
echo "root Application did not reach Synced in 5m"
exit 1
```

Also run a static render diff before the cluster step — fastest
feedback on a Helm/Kustomize mistake:

```bash
kustomize build manifests/model-serving/overlays/prod > /tmp/rendered.yaml
kubectl apply --dry-run=client -f /tmp/rendered.yaml
```

### 2f. Wiring it into CI

A reasonable GitHub Actions matrix (schematic; same shape works
on any runner):

```yaml
name: iac-ci
on: [pull_request]

jobs:
  static:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform fmt -check -recursive
      - run: terraform init -backend=false
      - run: terraform validate
      - uses: terraform-linters/setup-tflint@v4
      - run: tflint --recursive

  policy:
    needs: static
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init -backend=false
      - run: terraform plan -out=plan.bin
      - run: terraform show -json plan.bin > plan.json
      - uses: open-policy-agent/setup-opa@v2
      - run: conftest test plan.json --policy policies/

  unit:
    needs: static
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init -backend=false
      - run: terraform test

  integration:
    # only on main / nightly; expensive
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: [policy, unit]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      # Assume-role into a scratch account:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::...:role/ci-integration
          aws-region: us-west-2
      - run: terraform test -filter=tests/apply.tftest.hcl
```

## 3. Validation steps

1. `terraform fmt -check -recursive` is clean.
2. `terraform validate` returns success in every root module.
3. `tflint --recursive` returns no findings on `warning` or
   above.
4. `conftest test plan.json` passes for compliant plans and
   fails with a clear message when a policy is deliberately
   violated (drop the `owner` tag, re-plan, re-test).
5. `terraform test` passes `run "produces_node_group_arn"` and
   produces the expected failure in `run "rejects_untagged"`.
6. `terraform test` in apply mode (or Terratest) provisions and
   destroys real resources end-to-end; final `terraform state
   list` is empty post-destroy.
7. The kind smoke script reaches `Synced` within its timeout on
   a clean PR; deliberately breaking a manifest reproduces the
   failure.
8. CI matrix enforces the ordering — static → policy → unit →
   integration — and fast layers block before slow layers run.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Static checks (`fmt`, `validate`, `tflint`) run on every push | 10 |
| R2 | At least one policy is enforced via OPA/Conftest or Checkov against `terraform show -json` | 15 |
| R3 | Policy includes both a positive assertion (tag present) and a negative one (public access forbidden) | 10 |
| R4 | `terraform test` includes at least one plan-mode assertion and one `expect_failures` negative case | 15 |
| R5 | Integration layer exists, scoped to a scratch account/project, with guaranteed teardown | 15 |
| R6 | GitOps manifests validated with `kubectl --dry-run` + a Kustomize/Helm render diff | 10 |
| R7 | A kind-based smoke test confirms Argo CD can sync the manifests from the PR branch | 10 |
| R8 | CI pipeline runs layers in order of cost; integration layer is gated to merge/nightly | 10 |
| R9 | Test failures produce an actionable message (resource address + rule name), not a stack trace | 5 |

Pass threshold: **80/100**. R2, R4, and R5 are must-haves — a
pipeline without policy enforcement, without unit assertions, or
without real-cloud integration is skipping load-bearing layers.

## 5. Common mistakes

1. **Only integration tests.** Tests take 30 minutes, costs
   accrue, the feedback loop dies. Shift left: a `fmt` error
   should fail in ten seconds, not after a Terraform apply.
2. **Policy enforced "in code review", not in CI.** Humans miss
   things. Policy as code is the structural guarantee.
3. **Tests that hit real cloud on every push.** Rate limits,
   cost, flakiness. Keep real-cloud tests on merge or nightly;
   use plan-mode tests for every push.
4. **Teardown that depends on the test passing.** A failed
   assertion before `terraform destroy` leaks the whole apply.
   Use `defer` / `trap` / Terratest's cleanup pattern.
5. **Testing only the happy path.** Validation blocks exist to
   reject bad inputs — prove it with `expect_failures`.
6. **Smoke test that uses the prod cluster.** Use `kind` (or an
   ephemeral EKS). A smoke test is not production diagnostics.
7. **No render diff on PR.** Reviewers see the Helm/Kustomize
   change but not the resulting YAML. A rendered diff posted to
   the PR is the cheapest way to catch "oops, that templates to
   nothing".

## 6. References

Primary (official project documentation):

- Terraform — Tests (`.tftest.hcl`) —
  <https://developer.hashicorp.com/terraform/language/tests>
- Terraform — `fmt` —
  <https://developer.hashicorp.com/terraform/cli/commands/fmt>
- Terraform — `validate` —
  <https://developer.hashicorp.com/terraform/cli/commands/validate>
- Terraform — `show -json` (plan representation) —
  <https://developer.hashicorp.com/terraform/cli/commands/show>
- Open Policy Agent — Terraform tutorial —
  <https://www.openpolicyagent.org/docs/latest/terraform/>
- Conftest — <https://www.conftest.dev/>
- Checkov — <https://www.checkov.io/>
- tflint — <https://github.com/terraform-linters/tflint>
- Terratest — <https://terratest.gruntwork.io/docs/>
- kind (Kubernetes IN Docker) — <https://kind.sigs.k8s.io/docs/user/quick-start/>
- Argo CD install manifests —
  <https://argo-cd.readthedocs.io/en/stable/getting_started/>

Security / governance context:

- NIST AI Risk Management Framework (MEASURE function —
  verification and validation of system behavior against
  requirements) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 (ML06: AI Supply Chain
  Attacks motivates testing *who* produced the IaC manifests
  and *what* they produce) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>

Local context:

- Module-level rationale: `modules/mod-208-iac-gitops/SOLUTION.md`
  (Decisions 3, 6 — tooling split, sync-waves — both become
  testable surfaces).
- Engineer-tier IaC implementation reference:
  `engineer-solutions/mod-109` (per module README cross-reference).
