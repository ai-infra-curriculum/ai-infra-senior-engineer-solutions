# SOLUTION — Lab 01: Kubernetes Security Configuration

Reference solution for
[lab-01-k8s-security.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-209-security-compliance/exercises/lab-01-k8s-security.md).

Engineer-tier Kubernetes primitives are the same as
`engineer-solutions/mod-109 ex-07/08`; the senior
responsibility here is picking the *minimum* set of controls
that is defensible, enforceable by the API server (not by code
review), and coherent with the rest of the module's
architecture (short-lived credentials, signed artifacts,
default-deny networks — see `modules/mod-209-security-compliance/SOLUTION.md`).

## 1. Solution overview

A Kubernetes cluster hosting ML workloads is a multi-tenant
environment: the training cluster runs code written by many
teams against datasets of varying sensitivity. The control
surface the API server offers is well documented
([Kubernetes — Security Overview](https://kubernetes.io/docs/concepts/security/overview/))
and groups into four layers:

1. **Authentication and authorisation** — who the API server
   believes a caller is, and what that principal is allowed to
   do (RBAC
   — [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)).
2. **Workload isolation** — what a pod is allowed to run and
   touch at runtime (Pod Security Standards
   — [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/),
   PSA
   — [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)).
3. **Network posture** — which pod may talk to which
   (NetworkPolicy
   — [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)).
4. **Supply chain** — which images the cluster will admit and
   how their origin is verified (admission + image signatures
   — [Kubernetes — Admission Controllers Reference](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/),
   [sigstore cosign](https://docs.sigstore.dev/cosign/overview/)).

Senior additions on top of a "controls that pass CIS":

- **Default-deny is the ground state** everywhere it's
  available (RBAC, NetworkPolicy, PSA).
- **Workload identity, never static kubeconfigs** for CI/CD
  and cross-service auth
  ([IAM Roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html);
  [GKE Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity)).
- **Enforcement at admission, not just lint** — Kyverno or
  OPA Gatekeeper
  ([Kyverno overview](https://kyverno.io/docs/introduction/),
  [Gatekeeper docs](https://open-policy-agent.github.io/gatekeeper/website/docs/)) —
  so `kubectl apply` from a developer laptop cannot bypass
  what CI enforced.
- **Threat model is explicit.** The controls below map to
  OWASP Machine Learning Security Top 10 and MITRE ATLAS
  tactics; §6 names which.

## 2. Worked answer or implementation

### 2a. Namespace model and PSA baseline

Each tenant workload lives in its own namespace, labelled for
Pod Security Admission
([PSA — Namespace labels](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)).
Training tenants run `restricted`; privileged system
components live in a separate namespace with explicit
exemptions.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ml-training-team-a
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
    owner: team-a
    data-classification: pii-allowed
```

`restricted` blocks privileged containers, host namespaces,
host paths, and requires `runAsNonRoot`, `seccompProfile:
RuntimeDefault`, dropped capabilities, and read-only root
filesystems — the exact field set is in the
[Pod Security Standards reference](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted).

### 2b. A compliant training pod

The minimum pod shape that satisfies `restricted` + the
module's architectural defaults:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: trainer
  namespace: ml-training-team-a
spec:
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: trainer
      image: ghcr.io/acme/trainer@sha256:abc...   # digest, not tag
      imagePullPolicy: IfNotPresent
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      resources:
        requests: { cpu: "2", memory: "8Gi", "nvidia.com/gpu": "1" }
        limits:   { cpu: "4", memory: "16Gi", "nvidia.com/gpu": "1" }
      volumeMounts:
        - { name: scratch, mountPath: /scratch }
        - { name: cache,   mountPath: /tmp }
  volumes:
    - name: scratch
      emptyDir: { sizeLimit: 50Gi }
    - name: cache
      emptyDir: { sizeLimit: 2Gi }
```

Points the rubric cares about:

- `automountServiceAccountToken: false` — the pod has no
  need to talk to the API server, so it gets no token. When
  the pod *does* need to call the API (e.g. a sidecar),
  bind a purpose-specific ServiceAccount with a narrow Role,
  not `default`.
- **Image pinned by digest.** Tags are mutable; digests are
  not ([Kubernetes — Image names](https://kubernetes.io/docs/concepts/containers/images/#image-names)).
  Prevents a repushed `:v1` from silently changing what runs.
- **GPU scheduled by `nvidia.com/gpu`** resource request, the
  device-plugin contract
  ([Kubernetes — Scheduling GPUs](https://kubernetes.io/docs/tasks/manage-gpu/scheduling-gpus/)).

### 2c. RBAC: least-privilege ServiceAccounts

A training workload that needs to write metrics to Prometheus
PushGateway via the API server (contrived example) gets its
own SA bound to a Role that permits exactly that verb on
exactly that resource.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: trainer
  namespace: ml-training-team-a
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: trainer-metrics-writer
  namespace: observability
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["training-metrics-cursor"]
    verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: trainer-metrics-writer
  namespace: observability
subjects:
  - kind: ServiceAccount
    name: trainer
    namespace: ml-training-team-a
roleNamespace: observability    # cross-ns: Role in observability, Subject in training
```

Hard rules:

- **No `ClusterRoleBinding` to `cluster-admin` for a
  workload.** Ever. That binding is for break-glass human
  operators, not services
  ([RBAC — Default roles and role bindings](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings)).
- **No wildcard verbs.** `verbs: ["*"]` is a ClusterRole
  smell; name the verbs.
- **Bind to Roles (namespace-scoped) first.** Only reach for
  ClusterRole when the API resource is itself cluster-scoped
  (Node, PersistentVolume, CustomResourceDefinition).

### 2d. Default-deny NetworkPolicy, then explicit allow

`NetworkPolicy` is additive: no policies = allow all. The
baseline per tenant namespace is a deny-all plus one or more
narrow allows
([Network Policies — Default policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/#default-policies)).

```yaml
# default deny all ingress + egress in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: ml-training-team-a
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# allow the trainer to reach the model registry over https
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: trainer-to-registry
  namespace: ml-training-team-a
spec:
  podSelector:
    matchLabels: { app: trainer }
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector: { matchLabels: { name: platform-registry } }
          podSelector:       { matchLabels: { app: registry } }
      ports:
        - { protocol: TCP, port: 443 }
    - to:
        - ipBlock: { cidr: 169.254.169.254/32 }   # cloud metadata, if required
      ports:
        - { protocol: TCP, port: 80 }
```

Egress policy is the one that bites. Without it, a
compromised training pod can exfiltrate the dataset to
anywhere on the internet. The rule above requires a reason —
each destination is an intentional allow.

### 2e. Workload identity instead of static cloud creds

Service accounts federate to cloud IAM via OIDC. The pod's
projected token is exchanged for short-lived cloud
credentials
([ServiceAccount token volume projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection);
 [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html);
 [GKE Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity)).

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: trainer
  namespace: ml-training-team-a
  annotations:
    # AWS: SA assumes this role via OIDC
    eks.amazonaws.com/role-arn: "arn:aws:iam::111122223333:role/trainer-dataset-reader"
```

The IAM role's trust policy restricts the OIDC subject to
this exact (namespace, SA) pair. A pod in a different
namespace cannot assume the role even with a stolen token,
because the token's `sub` claim won't match.

### 2f. Admission policy — enforce what is already written

Kyverno (or Gatekeeper) runs at the admission boundary. The
module's architectural decisions become `ClusterPolicy`
objects so `kubectl apply` by any principal is also
filtered.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-digests
spec:
  validationFailureAction: Enforce
  rules:
    - name: images-must-use-digests
      match:
        any: [ { resources: { kinds: [Pod] } } ]
      validate:
        message: "Images must be pinned by digest (image@sha256:...)."
        pattern:
          spec:
            containers:
              - image: "*@sha256:*"
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  rules:
    - name: signed-by-platform-key
      match:
        any: [ { resources: { kinds: [Pod] } } ]
      verifyImages:
        - imageReferences: ["ghcr.io/acme/*"]
          attestors:
            - entries:
                - keys:
                    publicKeys: |
                      -----BEGIN PUBLIC KEY-----
                      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
                      -----END PUBLIC KEY-----
```

Signature verification (sigstore/cosign
— [cosign verify](https://docs.sigstore.dev/cosign/verifying/verify/))
closes the supply-chain path — a typosquatted image that
fetches but isn't signed by the platform key is rejected by
the admission controller. This directly addresses
OWASP ML Top 10 **ML06: AI Supply Chain Attacks**
([OWASP ML Top 10](https://owasp.org/www-project-machine-learning-security-top-10/)).

### 2g. etcd encryption at rest and audit logging

`EncryptionConfiguration` on the API server encrypts Secrets
before they land in etcd
([Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)).
On a managed cluster this is a provider flag; on a
self-hosted cluster it is a kube-apiserver `--encryption-provider-config`
argument.

Audit logging
([Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/))
is turned on at the `Metadata` level minimum, shipped to a
tamper-evident SIEM. Two policy entries that catch the
majority of attacker footprints:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    resources:
      - group: ""            # core
        resources: ["secrets", "configmaps", "serviceaccounts"]
  - level: Metadata
    verbs: ["create", "update", "patch", "delete"]
    omitStages: ["RequestReceived"]
```

### 2h. A reusable checklist tenants can self-serve

A Markdown runbook (lives next to the manifests) that a
service team walks through before onboarding. It is the
procedural half of the admission policies — the structural
half is Kyverno. See §4 for the matching review rubric.

## 3. Validation steps

1. `kubectl auth can-i --as=system:serviceaccount:ml-training-team-a:default '*' '*'` →
   `no`. The namespace default SA has no permissions.
2. `kubectl label namespace ml-training-team-a pod-security.kubernetes.io/enforce=restricted --overwrite` —
   deploying a pod with `privileged: true` is rejected by
   PSA.
3. `kubectl exec` into a running trainer and
   `curl https://example.com` from inside the pod — fails
   because the default-deny egress NetworkPolicy drops the
   packet. The allow-listed registry call still works.
4. `kubectl apply` a pod with an `image: nginx:latest` (no
   digest) — rejected by the `require-image-digests`
   Kyverno policy at admission.
5. `kubectl apply` a pod whose image is signed by a
   different cosign key — rejected by
   `verify-image-signatures`.
6. `cosign verify --key cosign.pub ghcr.io/acme/trainer@sha256:abc...` —
   succeeds for the admitted image.
7. `aws sts get-caller-identity` from inside the trainer pod
   returns the IRSA role ARN, not an IAM user. Removing the
   SA annotation and redeploying makes the same call fail.
8. `etcdctl get /registry/secrets/.../... --print-value-only`
   on the backing etcd shows ciphertext, not plaintext.
9. The audit log stream in SIEM shows the creation of the
   Secret above with a non-reversible hash of the object
   name, no body.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Every tenant namespace is PSA-labelled `enforce=restricted`; privileged exemptions are explicit and named | 10 |
| R2 | No workload is bound to `cluster-admin` or uses wildcard RBAC verbs; `automountServiceAccountToken` is `false` unless a Role is bound | 10 |
| R3 | Namespace ships a `default-deny` NetworkPolicy for ingress *and* egress; every allow names a destination selector or an `ipBlock` with a comment | 15 |
| R4 | Images are pinned by digest; `latest` is rejected at admission | 10 |
| R5 | Images from the platform registry are signature-verified at admission (cosign / Sigstore keys or Rekor identity) | 15 |
| R6 | Workloads use workload identity (IRSA / GKE WI / AKS Workload Identity); no long-lived cloud access keys mounted into pods | 10 |
| R7 | etcd is encrypted at rest with a provider-managed KMS key (or equivalent self-hosted KMS provider) | 10 |
| R8 | Kubernetes audit policy at least `Metadata` on all write verbs, `RequestResponse` on Secrets/SAs/CMs, shipped off-cluster | 10 |
| R9 | The controls above are **enforced by admission** (Kyverno / OPA Gatekeeper), not only by CI lint | 10 |

Pass threshold: **80/100**. R3, R5, and R9 are must-haves —
a cluster without default-deny egress, signature verification,
or admission-side enforcement is an open blast radius regardless
of total.

## 5. Common mistakes

1. **Pod Security Admission labels on the wrong namespace.**
   Labelling `kube-system` `restricted` breaks CoreDNS and
   the CNI. Label tenant namespaces, exempt system namespaces
   explicitly, and version-pin the enforce level.
2. **`ClusterRole` reused as `Role`.** Binding a ClusterRole
   with a RoleBinding scopes it correctly, but defining
   permissions at cluster scope "just in case" and binding
   narrowly later is how wildcard verbs leak in. Start at
   Role.
3. **NetworkPolicy for ingress but not egress.** Ingress
   lockdown stops neighbours; egress lockdown stops
   exfiltration. ML data egress is the one that bites after
   a credential compromise.
4. **Mounting cloud metadata (`169.254.169.254`) for every
   pod.** On AWS/GCP/Azure, the IMDS is a credential source;
   leaving it reachable from workload pods defeats workload
   identity. Block it by NetworkPolicy and require IMDSv2
   with hop-limit 1.
5. **Signature verification in CI only.** A developer with
   kubectl permissions can push an unsigned image path
   directly. The verification must run at admission.
6. **Sealed Secrets or ExternalSecrets but no etcd
   encryption.** Secret projection still writes plaintext to
   etcd. Both are required.
7. **Audit logging at `Metadata` for everything, including
   Secret CRUD.** You lose the before/after state when a
   Secret is rotated maliciously. Use `RequestResponse` for
   the sensitive resources.
8. **PSP YAMLs still in the repo.** PodSecurityPolicy was
   removed in v1.25
   ([PSP Removal](https://kubernetes.io/docs/concepts/security/pod-security-policy/));
   migrate to PSA.

## 6. References

Primary (official project documentation):

- Kubernetes — Security Overview —
  <https://kubernetes.io/docs/concepts/security/overview/>
- Kubernetes — RBAC Authorization —
  <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
- Kubernetes — Pod Security Standards —
  <https://kubernetes.io/docs/concepts/security/pod-security-standards/>
- Kubernetes — Pod Security Admission —
  <https://kubernetes.io/docs/concepts/security/pod-security-admission/>
- Kubernetes — Network Policies —
  <https://kubernetes.io/docs/concepts/services-networking/network-policies/>
- Kubernetes — Admission Controllers —
  <https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/>
- Kubernetes — Encrypting Confidential Data at Rest —
  <https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/>
- Kubernetes — Auditing —
  <https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/>
- Kubernetes — Image names and digests —
  <https://kubernetes.io/docs/concepts/containers/images/>
- Kubernetes — Scheduling GPUs —
  <https://kubernetes.io/docs/tasks/manage-gpu/scheduling-gpus/>
- Kyverno — Introduction / Policies —
  <https://kyverno.io/docs/introduction/>
- Open Policy Agent — Gatekeeper —
  <https://open-policy-agent.github.io/gatekeeper/website/docs/>
- sigstore cosign — Overview and verification —
  <https://docs.sigstore.dev/cosign/overview/>
- AWS — IAM Roles for Service Accounts (IRSA) —
  <https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html>
- GKE — Workload Identity —
  <https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity>

Security / governance context:

- OWASP Machine Learning Security Top 10 (ML06: AI Supply
  Chain Attacks is addressed by §2f image signing; ML02/ML09
  pressure comes in via tenant isolation and RBAC) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS (adversary techniques in cluster context:
  initial access via public-facing application, credential
  theft from mounted SA tokens, resource hijacking of GPUs) —
  <https://atlas.mitre.org/>
- NIST AI Risk Management Framework (GOVERN subcategories on
  roles, access, and change control map directly to RBAC +
  admission policy here) —
  <https://www.nist.gov/itl/ai-risk-management-framework>

Local context:

- Module-level rationale:
  `modules/mod-209-security-compliance/SOLUTION.md`
  (Decision 1 — no long-lived credentials; Decision 2 —
  SLSA + cosign supply chain; Decision 3 — default-deny
  networks).
- Engineer-tier Kubernetes-security implementation reference:
  `engineer-solutions/mod-109 ex-07/08` (per module README
  cross-reference).
