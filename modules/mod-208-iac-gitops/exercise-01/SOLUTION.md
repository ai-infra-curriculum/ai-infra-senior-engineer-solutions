# SOLUTION — Lab 01: Building Terraform Modules for ML Infrastructure

Reference solution for
[lab-01-terraform-modules.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-208-iac-gitops/exercises/lab-01-terraform-modules.md).

The engineer-tier Terraform mechanics are the same as
`engineer-solutions/mod-109`; the senior responsibility here is
designing the *module interface* that an ML platform team will
consume across many services, clusters, and accounts without
copy-pasting HCL.

## 1. Solution overview

A Terraform module is a reusable unit of configuration with a
declared interface (inputs), declared outputs, and a defined
blast radius ([Terraform docs — Modules](https://developer.hashicorp.com/terraform/language/modules)).
For an ML platform, three module tiers are enough to cover
most needs:

- **Resource modules** — thin wrappers around a single provider
  resource (e.g. "an S3 bucket with our tagging + encryption
  defaults").
- **Service modules** — a coherent capability composed of
  several resource modules (e.g. "a GPU training cluster" =
  node group + IAM + autoscaler + logs).
- **Environment modules / root configs** — pin providers,
  backends, and compose service modules into a deployable
  environment.

Standard module structure follows HashiCorp's convention
([Standard Module Structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure)):
`main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`,
`README.md`, `examples/`, optional `modules/` for nested
sub-modules.

Senior additions on top of "a module that runs":

- **Versioning + published registry source** — modules are
  consumed by `source = "git::…?ref=v1.2.0"` or a private
  module registry, never by local relative paths across
  repositories.
- **Validation of inputs** at `terraform plan` time
  ([Custom Conditions](https://developer.hashicorp.com/terraform/language/expressions/custom-conditions))
  so misconfiguration fails before apply.
- **Opinionated defaults with escape hatches** — the module
  encodes platform policy (encryption on, tags required) and
  exposes only the knobs that are actually variable.

## 2. Worked answer or implementation

### 2a. Repository layout

A single-purpose `gpu-training-cluster` service module for
AWS EKS. The structure below is the layout HashiCorp
recommends in the Standard Module Structure doc:

```
terraform-aws-gpu-training-cluster/
├── README.md
├── versions.tf
├── providers.tf          # intentionally empty (see §2d)
├── variables.tf
├── outputs.tf
├── main.tf
├── iam.tf
├── nodegroup.tf
├── logs.tf
├── examples/
│   └── minimal/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars.example
└── tests/
    └── gpu_training_cluster.tftest.hcl
```

### 2b. Pin provider + Terraform versions

`versions.tf` — any module that will be consumed across many
root configs must pin a `required_version` and provider version
constraints ([Provider Requirements](https://developer.hashicorp.com/terraform/language/providers/requirements)).

```hcl
terraform {
  required_version = ">= 1.6.0, < 2.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.50"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.30"
    }
  }
}
```

### 2c. Typed variables with validation

Every input is typed and, where it carries a platform invariant,
validated. `validation` blocks run at plan time
([Custom Conditions — Input Validation](https://developer.hashicorp.com/terraform/language/expressions/custom-conditions#input-variable-validation)).

```hcl
variable "cluster_name" {
  type        = string
  description = "EKS cluster name. Used as a prefix for all owned resources."
  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{2,39}$", var.cluster_name))
    error_message = "cluster_name must be lowercase alphanumeric with hyphens, 3-40 chars."
  }
}

variable "gpu_instance_types" {
  type        = list(string)
  description = "GPU instance types for the training node group."
  default     = ["g5.2xlarge"]
  validation {
    condition     = length(var.gpu_instance_types) > 0
    error_message = "At least one GPU instance type is required."
  }
}

variable "min_nodes" {
  type    = number
  default = 0
}

variable "max_nodes" {
  type    = number
  default = 8
  validation {
    condition     = var.max_nodes >= var.min_nodes
    error_message = "max_nodes must be >= min_nodes."
  }
}

variable "tags" {
  type        = map(string)
  description = "Mandatory platform tags. Must include owner and cost_center."
  validation {
    condition = alltrue([
      contains(keys(var.tags), "owner"),
      contains(keys(var.tags), "cost_center"),
    ])
    error_message = "tags must include 'owner' and 'cost_center' (platform policy)."
  }
}
```

### 2d. Providers belong to the caller, not the module

`providers.tf` is intentionally empty. Modules should not
configure their own providers; the root configuration owns
provider blocks and passes them in. HashiCorp's guidance:
[Providers within Modules](https://developer.hashicorp.com/terraform/language/modules/develop/providers)
— "A module intended to be called by one or more other modules
must not contain any `provider` blocks."

The reason: if a module hardcodes `provider "aws" { region =
"us-east-1" }`, no caller can deploy it in a different region
or against a different account without forking the module.

### 2e. Resource composition

`nodegroup.tf` — minimal shape; full EKS-cluster plumbing
elided but follows the same pattern:

```hcl
resource "aws_eks_node_group" "gpu" {
  cluster_name    = var.cluster_name
  node_group_name = "${var.cluster_name}-gpu"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.subnet_ids
  instance_types  = var.gpu_instance_types
  capacity_type   = "ON_DEMAND"

  scaling_config {
    min_size     = var.min_nodes
    max_size     = var.max_nodes
    desired_size = var.min_nodes
  }

  labels = {
    "workload"             = "ml-training"
    "nvidia.com/gpu.present" = "true"
  }

  tags = merge(var.tags, {
    "k8s.io/cluster-autoscaler/enabled" = "true"
    "k8s.io/cluster-autoscaler/${var.cluster_name}" = "owned"
  })

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [scaling_config[0].desired_size]
  }
}
```

The `ignore_changes` on `desired_size` is the pattern for
letting cluster-autoscaler own that field at runtime without
Terraform fighting it back
([Resource Lifecycle](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle)).

### 2f. Narrow, typed outputs

`outputs.tf` — outputs are the *consumer* interface. Expose
what the caller needs, not everything the module created
([Terraform Output Values](https://developer.hashicorp.com/terraform/language/values/outputs)).

```hcl
output "node_group_arn" {
  description = "ARN of the GPU node group."
  value       = aws_eks_node_group.gpu.arn
}

output "node_role_arn" {
  description = "IAM role used by GPU nodes (for IRSA attachment, etc.)."
  value       = aws_iam_role.node.arn
}

output "node_security_group_id" {
  description = "Security group ID for GPU nodes (for peering)."
  value       = aws_eks_node_group.gpu.resources[0].remote_access_security_group_id
}
```

### 2g. An `examples/` directory — doubles as docs + tests

Terraform's module convention is that `examples/` contains
runnable root configurations ([Standard Module Structure — Examples](https://developer.hashicorp.com/terraform/language/modules/develop/structure#examples)).
They are the first thing a consumer reads and the integration
test target.

```hcl
# examples/minimal/main.tf
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.50" }
  }
}

provider "aws" {
  region = "us-west-2"
}

module "training" {
  source = "../../"

  cluster_name       = "ml-training-dev"
  subnet_ids         = ["subnet-aaa", "subnet-bbb"]
  gpu_instance_types = ["g5.2xlarge"]
  min_nodes          = 0
  max_nodes          = 4
  tags = {
    owner       = "ml-platform"
    cost_center = "ai-research"
    env         = "dev"
  }
}
```

### 2h. Native test harness

Terraform 1.6+ ships a native test framework
([Terraform Tests](https://developer.hashicorp.com/terraform/language/tests)).
A `.tftest.hcl` file runs plan/apply against a module and
asserts on outputs — this is what belongs in CI alongside
`terraform validate`.

```hcl
# tests/gpu_training_cluster.tftest.hcl
variables {
  cluster_name       = "ml-ci-01"
  subnet_ids         = ["subnet-aaa", "subnet-bbb"]
  gpu_instance_types = ["g5.2xlarge"]
  min_nodes          = 0
  max_nodes          = 2
  tags = {
    owner       = "ml-platform"
    cost_center = "ai-research"
  }
}

run "validate_plan" {
  command = plan

  assert {
    condition     = output.node_group_arn != ""
    error_message = "node_group_arn must be produced."
  }
}

run "reject_missing_owner_tag" {
  command = plan
  variables {
    tags = { cost_center = "ai-research" }   # missing owner
  }
  expect_failures = [var.tags]
}
```

### 2i. Versioning + consumption

The module is published as a tag in its own Git repository and
consumed by version pin, not `main`
([Module Sources — Selecting a Revision](https://developer.hashicorp.com/terraform/language/modules/sources#selecting-a-revision)).

```hcl
# root config
module "training" {
  source = "git::ssh://git@github.com/acme/terraform-aws-gpu-training-cluster.git?ref=v1.3.0"
  # ...inputs...
}
```

The `terraform-<PROVIDER>-<NAME>` repo name follows the Terraform
Registry naming convention ([Publishing Modules](https://developer.hashicorp.com/terraform/registry/modules/publish#requirements)),
which lets the same repo be registered in a private registry later
without renaming.

## 3. Validation steps

1. `terraform init` in `examples/minimal/` succeeds — provider and
   source resolution works.
2. `terraform validate` passes in both the module root and the
   example.
3. `terraform fmt -check -recursive` is clean.
4. `terraform plan -var-file=terraform.tfvars.example` in the
   example prints the expected set of resources.
5. Tampering with a tag (remove `owner` from the example's `tags`)
   fails at plan time with the validation message, not at apply.
6. `terraform test` from the module root passes `validate_plan`
   and fails the `reject_missing_owner_tag` case as expected.
7. Consuming the module from a *different* root configuration via
   the `git::…?ref=` source works without importing providers
   from inside the module.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Module follows Standard Module Structure (main / variables / outputs / versions / README / examples) | 10 |
| R2 | `required_version` and `required_providers` are pinned with ranges, not floating | 10 |
| R3 | Every variable has a `type` and a `description` | 10 |
| R4 | Platform-policy invariants (tagging, encryption, naming) are enforced by `validation` blocks | 15 |
| R5 | Module contains **no** `provider` blocks — providers come from the caller | 15 |
| R6 | Outputs are limited to what downstream modules actually need; no leaking of internal state | 10 |
| R7 | `examples/` contains at least one runnable, self-contained example | 10 |
| R8 | At least one `.tftest.hcl` run asserts on outputs and at least one runs a negative case | 10 |
| R9 | Module is consumed via a versioned Git/registry source, not relative path | 10 |

Pass threshold: **80/100**. R4, R5, and R8 are must-haves — a
module that encodes no policy, owns its own providers, or has no
tests is not a platform module regardless of total.

## 5. Common mistakes

1. **`provider` block inside the module.** Breaks any caller
   that needs a different region, account, or assume-role.
   Providers always live with the root config.
2. **Floating version constraints.** `version = ">= 5.0"` on the
   AWS provider means an upstream minor release can break your
   consumers overnight. Pin with `~>`.
3. **Overbroad outputs.** Exposing every attribute of every
   resource turns the module into an opaque bag and locks you
   out of refactoring internals. Expose the contract only.
4. **No input validation, policy enforced "by code review".**
   The whole point of a platform module is to make the policy
   structural: `validation { }` or `precondition { }` blocks,
   not a wiki page.
5. **Consuming via local path across repos.** Breaks
   reproducibility; whoever runs `terraform plan` gets whatever
   their checkout happens to be. Use a versioned source.
6. **Module does both the resource and the cluster.** One
   module trying to own the EKS cluster, the node groups, the
   VPC, IAM roles, and Helm releases becomes a ball that
   nobody can refactor. Split by blast radius.
7. **`terraform apply` directly to production from a laptop.**
   No plan review, no state locking enforcement, no audit
   trail. Apply from CI against a shared remote backend only.

## 6. References

Primary (official project documentation):

- Terraform — Modules overview — <https://developer.hashicorp.com/terraform/language/modules>
- Terraform — Standard Module Structure —
  <https://developer.hashicorp.com/terraform/language/modules/develop/structure>
- Terraform — Providers within Modules —
  <https://developer.hashicorp.com/terraform/language/modules/develop/providers>
- Terraform — Custom Conditions (variable validation,
  preconditions, postconditions) —
  <https://developer.hashicorp.com/terraform/language/expressions/custom-conditions>
- Terraform — Provider Requirements —
  <https://developer.hashicorp.com/terraform/language/providers/requirements>
- Terraform — Output Values —
  <https://developer.hashicorp.com/terraform/language/values/outputs>
- Terraform — Tests (`.tftest.hcl`) —
  <https://developer.hashicorp.com/terraform/language/tests>
- Terraform — Module Sources —
  <https://developer.hashicorp.com/terraform/language/modules/sources>
- Terraform — Publishing Modules (naming conventions) —
  <https://developer.hashicorp.com/terraform/registry/modules/publish>

Security / governance context:

- NIST AI Risk Management Framework (infrastructure is a GOVERN-
  and MAP-function surface — tagging, ownership, and change
  control map directly to RMF subcategories) —
  <https://www.nist.gov/itl/ai-risk-management-framework>

Local context:

- Module-level rationale: `modules/mod-208-iac-gitops/SOLUTION.md`
  (Decision 3 — tooling choices; Decision 5 — pull-based
  deployment model that these modules feed).
- Engineer-tier IaC implementation reference:
  `engineer-solutions/mod-109` (per module README cross-reference).
