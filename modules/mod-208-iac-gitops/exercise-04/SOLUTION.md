# SOLUTION — Lab 04: Secrets Management

Reference solution for
[lab-04-secrets-management.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-208-iac-gitops/exercises/lab-04-secrets-management.md).

A GitOps repo must contain the full desired state of the
cluster, including which secrets exist — but not their
contents. Two established patterns resolve this:
encrypted-in-Git (Sealed Secrets) and pointer-in-Git
(External Secrets Operator pulling from a secret store).

## 1. Solution overview

Kubernetes `Secret` objects are base64 — not encryption. A
committed `Secret` manifest is a committed credential
([Kubernetes — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)).
A durable solution picks one of two shapes:

- **Sealed Secrets** (Bitnami) — commit an encrypted
  `SealedSecret` CRD; an in-cluster controller decrypts into a
  normal `Secret` using its private key
  ([bitnami-labs/sealed-secrets](https://github.com/bitnami-labs/sealed-secrets)).
  The encrypted payload is scoped by namespace + name, which
  prevents reuse from being silently moved.
- **External Secrets Operator** — commit an `ExternalSecret`
  manifest that references an external store (AWS Secrets
  Manager, GCP Secret Manager, HashiCorp Vault, Azure Key
  Vault); the operator pulls values and projects them into a
  `Secret`
  ([external-secrets.io — Introduction](https://external-secrets.io/latest/introduction/overview/)).

When to pick which:

| | Sealed Secrets | External Secrets Operator |
|---|---|---|
| Source of truth | Git | External secret store |
| Rotation | New Git commit per rotation | Rotate in the store; cluster re-pulls |
| Multi-cluster same secret | Re-seal per cluster key | One source, many consumers |
| Audit log | Git history | Secret-store audit log |
| Works offline / no secret store | Yes | No |

The senior move is almost always ESO for anything shared across
services or clusters (central rotation, central audit) and
Sealed Secrets for self-contained deploys where introducing a
secret store would be overkill. Both can coexist.

A third, lower-trust pattern — **SOPS-encrypted files** —
([getsops/sops](https://github.com/getsops/sops)) — is useful
for Terraform or Helm values files that aren't Kubernetes
resources, but doesn't replace Sealed Secrets or ESO inside the
cluster.

## 2. Worked answer or implementation

### 2a. Install the controllers via GitOps

Both controllers are themselves Argo CD Applications in the
bootstrap repo. The install manifests are the project
maintainers'.

```yaml
# apps/platform/sealed-secrets.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://bitnami-labs.github.io/sealed-secrets
    chart: sealed-secrets
    targetRevision: 2.16.0        # pin
    helm:
      releaseName: sealed-secrets
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

```yaml
# apps/platform/external-secrets.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://charts.external-secrets.io
    chart: external-secrets
    targetRevision: 0.10.5        # pin
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

Chart install instructions: [Sealed Secrets install](https://github.com/bitnami-labs/sealed-secrets#installation)
and [ESO install](https://external-secrets.io/latest/introduction/getting-started/).

### 2b. Sealed Secrets — the commit flow

1. Author a plain `Secret` locally (never commit this file):

   ```yaml
   # /tmp/registry-creds.yaml  (gitignored path)
   apiVersion: v1
   kind: Secret
   metadata:
     name: registry-creds
     namespace: ml-serving
   type: kubernetes.io/dockerconfigjson
   data:
     .dockerconfigjson: "<base64>"
   ```

2. Seal it with the cluster's public key fetched from the
   controller ([`kubeseal` usage](https://github.com/bitnami-labs/sealed-secrets#usage)):

   ```bash
   kubeseal \
       --controller-name sealed-secrets \
       --controller-namespace kube-system \
       --format yaml \
       < /tmp/registry-creds.yaml \
       > manifests/ml-serving/sealed-registry-creds.yaml
   ```

3. Commit the resulting `SealedSecret` to Git. The sealed
   payload is bound to namespace + name by default: moving it
   to a different namespace or renaming it makes decryption
   fail
   ([Scope](https://github.com/bitnami-labs/sealed-secrets#scopes)).

4. Argo CD syncs the `SealedSecret`. The controller decrypts
   into a `Secret` of the same name/namespace. The decrypted
   `Secret` is never written to Git.

Resulting committed shape (abbreviated):

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: registry-creds
  namespace: ml-serving
spec:
  encryptedData:
    .dockerconfigjson: AgA3f7...
  template:
    metadata:
      name: registry-creds
      namespace: ml-serving
    type: kubernetes.io/dockerconfigjson
```

**Key rotation.** The controller rotates its private key on a
30-day default schedule and retains prior keys for backward
decrypt ([Secret rotation](https://github.com/bitnami-labs/sealed-secrets#secret-rotation)).
To rotate a sealed *value*, re-seal and commit; Argo CD picks
up the change.

### 2c. External Secrets — the pointer flow

A `SecretStore` (namespaced) or `ClusterSecretStore` declares
an auth path into the backing store ([SecretStore](https://external-secrets.io/latest/api/secretstore/)).
An `ExternalSecret` names which keys to pull and which
`Secret` to project them into ([ExternalSecret](https://external-secrets.io/latest/api/externalsecret/)).

```yaml
# manifests/ml-serving/secret-store.yaml  (committed)
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-sm
  namespace: ml-serving
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa    # IRSA-backed; never a static key
```

```yaml
# manifests/ml-serving/external-secret-model-key.yaml  (committed)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: model-serving-api-key
  namespace: ml-serving
spec:
  refreshInterval: 1h           # re-pull from store
  secretStoreRef:
    name: aws-sm
    kind: SecretStore
  target:
    name: model-serving-api-key
    creationPolicy: Owner       # ESO owns the resulting Secret
  data:
    - secretKey: api_key
      remoteRef:
        key: /ml-serving/prod/api-key
        version: AWSCURRENT
```

Notes:

- The *values* never appear in Git. The manifests describe
  *where* and *what to pull*.
- `refreshInterval` is the lag between store rotation and
  cluster-side availability
  ([refreshInterval semantics](https://external-secrets.io/latest/api/externalsecret/#externalsecretspec)).
- AWS auth uses IRSA via a ServiceAccount annotation
  ([AWS IRSA with ESO](https://external-secrets.io/latest/provider/aws-secrets-manager/));
  the ServiceAccount is itself a committed manifest.

### 2d. SOPS for Terraform / Helm values

For secrets that are inputs to IaC (Terraform `tfvars`, Helm
`values.yaml`), SOPS encrypts per-field with a KMS key
([getsops/sops — README](https://github.com/getsops/sops#readme)).

```bash
sops --encrypt --kms arn:aws:kms:us-west-2:...:key/UUID \
     terraform.tfvars.yaml > terraform.tfvars.enc.yaml
```

CI assumes an IAM role with `kms:Decrypt` on the specific key
and decrypts into memory at apply time; the plaintext file
never touches disk.

### 2e. Policy — prevent plain `Secret` manifests from merging

The weakest-link control: refuse any plain `Secret` on a PR
unless it is explicitly exempt. Enforce it in CI with Conftest
(policy-as-code — see exercise-03) and in the cluster with
Kyverno or OPA Gatekeeper.

```rego
# policies/no-plain-secrets.rego
package main

deny[msg] {
    input.kind == "Secret"
    not input.metadata.annotations["policy.acme/allow-plain-secret"]
    msg := sprintf("plain Secret %s/%s is forbidden; use SealedSecret or ExternalSecret",
                   [input.metadata.namespace, input.metadata.name])
}
```

Run `conftest test manifests/ --policy policies/` on every PR.

### 2f. Rotation runbook — who owns what

| Secret class | Owner | Rotation path |
|---|---|---|
| Static API keys in AWS Secrets Manager | Platform | Rotate value in SM; ESO re-pulls within `refreshInterval` |
| Database creds | Owner service | Rotate via AWS SM scheduled rotation; ESO re-pulls |
| Image-pull creds (`dockerconfigjson`) | Platform | New `SealedSecret` commit |
| TLS cert / key | cert-manager | Issuer re-issues; controller rolls the `Secret` |
| CI / Terraform assume-role credentials | Platform | GitHub OIDC; no long-lived key in Git |

## 3. Validation steps

1. `kubectl get sealedsecret -A` lists the committed Sealed
   Secrets; `kubectl get secret -A` shows the decrypted
   counterparts, same names, same namespaces.
2. Re-apply a `SealedSecret` under a different `namespace:` —
   the controller refuses to decrypt it (scope enforcement).
3. Updating a value in AWS Secrets Manager causes the
   corresponding `Secret` to update within `refreshInterval`
   (watch with `kubectl get secret model-serving-api-key -o
   jsonpath='{.data.api_key}' | base64 -d` before/after).
4. `grep -R 'kind: Secret' manifests/` returns nothing not
   covered by the `policy.acme/allow-plain-secret` exemption.
5. The Conftest policy fails a deliberately-added plain
   `Secret` in a test PR.
6. Deleting an `ExternalSecret` deletes the projected `Secret`
   (because `creationPolicy: Owner`); re-applying restores it.
7. Rotating the Sealed Secrets controller key and re-issuing
   `kubeseal` still decrypts prior `SealedSecret` resources
   (retained prior keys).

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | No plaintext `Secret` manifests committed to Git | 15 |
| R2 | Sealed Secrets and/or External Secrets Operator installed via GitOps (not `helm install` from a laptop) | 10 |
| R3 | At least one `SealedSecret` demonstrates namespace + name scoping | 10 |
| R4 | External `SecretStore` uses workload identity (IRSA / GCP WI / Vault K8s auth), not a static access key | 15 |
| R5 | `ExternalSecret.refreshInterval` is set explicitly, not left default | 5 |
| R6 | Policy rule (OPA/Kyverno) blocks plain `Secret` manifests on PRs | 15 |
| R7 | A rotation runbook exists: who rotates, how, and how the cluster picks up the change | 10 |
| R8 | Terraform secrets (if any) are SOPS-encrypted or sourced from a secret store, not plaintext `tfvars` in Git | 10 |
| R9 | Sealed Secrets controller keys are backed up (or rotation + retention policy is explicit) | 10 |

Pass threshold: **80/100**. R1, R4, and R6 are must-haves — a
repo containing plain secrets, static keys, or no enforcing
policy is not a secret-management solution regardless of total.

## 5. Common mistakes

1. **Plain `Secret` "just for now".** Git history keeps the
   value forever. Treat `git push` of a plain secret as an
   incident: rotate immediately.
2. **Sealed Secret sealed with the wrong cluster's public
   key.** The decryption silently fails; the workload gets no
   secret; the operator sees a confusing error. Always fetch
   the public key from the target cluster's controller.
3. **Reusing a sealed value across namespaces.** The default
   `strict` scope prevents it. Switching to `cluster-wide` to
   "make it work" removes a real guard; usually the right fix
   is to re-seal per namespace.
4. **ExternalSecret with a long `refreshInterval`.** Rotations
   don't propagate until hours later. Match to the workload's
   tolerance for stale credentials.
5. **Static IAM keys for the ESO SecretStore.** Defeats the
   point — you've just moved the problem. Use IRSA / workload
   identity / Vault K8s auth.
6. **No policy preventing plain `Secret` reintroduction.**
   One rushed PR regresses the whole program. Policy as code,
   in CI and admission.
7. **Controller private key with no backup.** Lose the key,
   lose the ability to decrypt every historical SealedSecret.
   Back up to a secret store; document the restore path.
8. **Terraform `tfvars` with plaintext secrets.** Equally bad
   as a plain `Secret`. SOPS or secret-store lookup, not a
   file committed "in a private repo".

## 6. References

Primary (official project documentation):

- Kubernetes — Secrets (base64 is encoding, not encryption) —
  <https://kubernetes.io/docs/concepts/configuration/secret/>
- bitnami-labs / sealed-secrets —
  <https://github.com/bitnami-labs/sealed-secrets>
- External Secrets Operator — Overview —
  <https://external-secrets.io/latest/introduction/overview/>
- ExternalSecret resource —
  <https://external-secrets.io/latest/api/externalsecret/>
- SecretStore / ClusterSecretStore resource —
  <https://external-secrets.io/latest/api/secretstore/>
- ESO AWS Secrets Manager provider —
  <https://external-secrets.io/latest/provider/aws-secrets-manager/>
- getsops / sops — <https://github.com/getsops/sops>
- HashiCorp Vault — Kubernetes auth method —
  <https://developer.hashicorp.com/vault/docs/auth/kubernetes>

Security / governance context:

- NIST AI Risk Management Framework (GOVERN function —
  policies, roles, and responsibilities for AI supply-chain
  components including credentials) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 (ML06: AI Supply Chain
  Attacks — compromised credentials are a primary vector) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS (adversary techniques against credential stores
  in AI systems) — <https://atlas.mitre.org/>

Local context:

- Module-level rationale: `modules/mod-208-iac-gitops/SOLUTION.md`
  (Decision 4 — Sealed Secrets / External Secrets as the
  cluster-side secret surface).
- Engineer-tier IaC implementation reference:
  `engineer-solutions/mod-109` (per module README cross-reference).
