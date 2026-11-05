# SOLUTION — Lab 02: Zero-Trust Architecture Implementation

Reference solution for
[lab-02-zero-trust-architecture.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-209-security-compliance/exercises/lab-02-zero-trust-architecture.md).

The engineer-tier mechanics (mTLS, SPIFFE, workload identity)
are the same as `engineer-solutions/mod-109 ex-07/08`; the
senior responsibility here is a *written architecture* —
tenets, trust zones, enforcement points, and an adoption
sequence — that an ML platform team can actually land, not a
diagram labelled "zero trust" on top of an unchanged perimeter.

## 1. Solution overview

Zero Trust is a model, not a product. NIST SP 800-207 defines
it as "a collection of concepts and ideas designed to minimize
uncertainty in enforcing accurate, least-privilege per-request
access decisions in information systems and services in the
face of a network viewed as compromised"
([NIST SP 800-207 — Zero Trust Architecture](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-207.pdf)).
The architecture is anchored by seven tenets (SP 800-207 §2.1);
the ones that bind hardest in an ML platform are:

- Every resource access is authenticated and authorised **per
  request**, from a dynamic policy.
- Trust is a property of identity + context, not of network
  location.
- Communication is secured regardless of network.
- Access to resources is on a per-session basis.
- The enterprise collects as much state about the current
  state of assets, network infrastructure, and communications
  as possible to improve its security posture.

The reference architecture realises these in four planes:

1. **Identity plane** — every principal (human, workload,
   pipeline) has a cryptographically verifiable identity
   (SPIFFE / OIDC —
   [SPIFFE spec](https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE.md),
   [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)).
2. **Policy plane** — a Policy Decision Point (PDP) evaluates
   per-request access from signals (identity, device posture,
   workload context). SP 800-207 calls this the Policy
   Engine + Policy Administrator pair.
3. **Enforcement plane** — Policy Enforcement Points (PEPs)
   at every ingress to a protected resource: service mesh
   sidecar (Istio
   — [Istio security](https://istio.io/latest/docs/concepts/security/);
   Linkerd
   — [Linkerd authentication policy](https://linkerd.io/2.15/features/server-policy/)),
   API gateway, model-server auth filter.
4. **Observability plane** — every decision is logged with
   the inputs it used, feeding both audit and the policy
   engine's own feedback loops.

Senior additions on top of "mTLS is on":

- **Trust zones named up front** — datasets, training
  compute, model registry, inference serving, user-facing
  APIs. Each has a documented data classification and set
  of allowed interactions.
- **Deny by default at *every* PEP**, not only at the
  cluster edge. A stolen service token in zone A does not
  grant implicit access to anything in zone B.
- **ML-specific inclusions.** Training data and model
  artifacts are *resources* in the Zero Trust sense: access
  is per-request and audited, not "the trainer has
  read-the-bucket IAM". This maps to NIST AI RMF MANAGE and
  MEASURE functions.

## 2. Worked answer or implementation

### 2a. Trust-zone map (decision artifact)

A table the student produces; the reference set below is a
representative mid-size ML platform.

| Zone | Contents | Classification | Who may call in | Who it may call |
|---|---|---|---|---|
| Z-DATA | Dataset object storage, feature store | PII possible | Z-TRAIN (read), Z-OPS (read-write via platform console) | KMS, audit sink |
| Z-TRAIN | Training clusters, notebooks | Code of any origin | Z-OPS (admin), humans via workload identity | Z-DATA (read), Z-REG (write-artifact), KMS |
| Z-REG | Model registry, artifact store | Signed-only | Z-TRAIN (write), Z-SERVE (read) | Z-SIGSTORE, audit sink |
| Z-SERVE | Model servers, routing | Signed-only | Z-USER (via gateway) | Z-REG (read), KMS |
| Z-USER | User-facing API gateway | Public | Internet | Z-SERVE |
| Z-OPS | Human admin paths, CI/CD | Privileged | Human IdP, CI OIDC | All (narrowly) |

The rule: *every* arrow between two zones is a PEP-enforced
policy. "In the same VPC" or "in the same cluster" is not a
grant.

### 2b. Identity: SPIFFE for workloads, OIDC for humans

Workload identity uses SPIFFE IDs issued via SPIRE
([SPIRE concepts](https://spiffe.io/docs/latest/spire-about/spire-concepts/)).
A SPIFFE ID encodes trust-domain + path:

```
spiffe://ai.acme.internal/ns/ml-training-team-a/sa/trainer
```

Workloads attest to SPIRE using Kubernetes attributes (node,
namespace, service account, container image SHA) — the pod
cannot present an identity that doesn't match what SPIRE sees
at attestation time
([SPIRE — Attestation](https://spiffe.io/docs/latest/spire-about/spire-concepts/#attestation)).

Human identity uses the enterprise IdP via OIDC, with MFA
enforced on the IdP side. kubectl, Argo CD, the model
registry UI, and the CI runner all accept the same IdP's
tokens; no service maintains its own user table.

### 2c. Network and transport: mesh-enforced mTLS

Istio or Linkerd is installed with strict mTLS between all
mesh-enabled namespaces
([Istio — Peer authentication: Strict mode](https://istio.io/latest/docs/reference/config/security/peer_authentication/)):

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

PeerAuthentication alone does **not** authorise — it
authenticates the SPIFFE identity of the caller. Authorisation
is a separate policy:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: serving-reads-registry
  namespace: model-registry
spec:
  selector: { matchLabels: { app: registry-api } }
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/model-serving/sa/model-server
      to:
        - operation:
            methods: ["GET"]
            paths:   ["/v1/models/*/versions/*/artifact"]
```

Reference semantics: the SPIFFE principal
`cluster.local/ns/model-serving/sa/model-server` can **GET**
artifact paths only; no other method, no other path. Any other
caller is denied even inside the mesh. This is the per-request,
per-session enforcement SP 800-207 §3.1 specifies.

### 2d. API-gateway edge policy (Z-USER → Z-SERVE)

The gateway terminates TLS, validates the end-user JWT (OIDC
from the IdP), and mints or passes an internal credential for
Z-SERVE. The policy includes the resource claim *and* a rate
budget (SP 800-207 §3.4.2 — dynamic risk inputs):

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: user-jwt
  namespace: gateway
spec:
  selector: { matchLabels: { app: edge-gateway } }
  jwtRules:
    - issuer: "https://idp.acme.com"
      jwksUri: "https://idp.acme.com/.well-known/jwks.json"
      audiences: ["ml-api"]
      forwardOriginalToken: true
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-authenticated-user
  namespace: gateway
spec:
  selector: { matchLabels: { app: edge-gateway } }
  action: ALLOW
  rules:
    - from:
        - source:
            requestPrincipals: ["https://idp.acme.com/*"]
      when:
        - key: request.auth.claims[scope]
          values: ["ml.inference.invoke"]
```

### 2e. Data-plane policies (Z-DATA)

Object storage is the highest-value asset. Three stacked
controls:

1. **Workload identity for the bucket** (IRSA or GCP Workload
   Identity) so no service account key exists anywhere.
2. **Bucket policy** denying anything except known SPIFFE-derived
   IAM roles.
3. **At-request-time audit** — every GET of a PII-classified
   object is logged with the caller's IAM role, the requesting
   SPIFFE ID (from an X-SPIFFE-ID header the mesh injects),
   and the dataset classification.

AWS example condition on the bucket policy:

```json
{
  "Effect": "Deny",
  "NotPrincipal": {
    "AWS": [
      "arn:aws:iam::111122223333:role/trainer-team-a-dataset-reader",
      "arn:aws:iam::111122223333:role/pipeline-dataset-copier"
    ]
  },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::acme-ml-datasets-pii/*"
}
```

### 2f. Admin paths (Z-OPS): short-lived, JIT, audited

No standing admin. Kubernetes cluster-admin and cloud-admin
roles are granted via a Just-in-Time flow (IdP-driven
privileged access, e.g. Teleport / Entra PIM / OpenBao-fronted
SSH certs — the exact tool is organisational) with:

- Max duration **4 hours**.
- Mandatory ticket number as justification.
- Session recording for interactive sessions.
- Dual-control (second approver) for changes touching Z-DATA
  or Z-REG.

The "least-privilege" tenet of SP 800-207 is not "small
Role" — it is also "short duration".

### 2g. Observability and the feedback loop

Every PEP emits structured audit:

- **Service mesh:** Envoy access logs with the client's
  SPIFFE ID (Istio
  — [Envoy access logs](https://istio.io/latest/docs/tasks/observability/logs/access-log/)).
- **API gateway:** JWT subject, scopes, resource, status.
- **Cloud data plane:** CloudTrail / Cloud Audit Logs for
  object access.
- **Kubernetes:** API audit log (exercise-01 §2g).

All sinks feed a tamper-evident SIEM. The feedback loop
(SP 800-207 §3.1, policy engine's "trust algorithm")
ingests signals:

- Impossible travel, unusual egress volume, calls to
  previously-unused resources → raises risk score →
  gateway downgrades scope or asks for step-up auth.
- Dataset access from a non-training namespace →
  immediate deny, alert to on-call.

### 2h. Adoption sequence (not a big-bang rewrite)

| Phase | Weeks | Deliverables | Exit criterion |
|---|---|---|---|
| 1. Identity | 1–3 | SPIRE installed, OIDC federation live, workload identity in training namespaces | Every workload has a SPIFFE ID; no static cloud keys in `Secret`s |
| 2. mTLS everywhere | 4–6 | Mesh installed, `PeerAuthentication: PERMISSIVE` → metrics → `STRICT` | ≥99% mesh traffic mTLS'd; dashboard green for 2 weeks |
| 3. Authz on the east-west edges | 7–9 | `AuthorizationPolicy` for Z-REG, Z-DATA; deny-by-default | Deliberate cross-zone call without policy fails |
| 4. North-south lockdown | 10–12 | API gateway with JWT + scopes; no bypass path | Every production request has an authenticated principal in logs |
| 5. Risk signals + JIT | 13–16 | SIEM feeding policy engine; JIT admin live | Standing admin reduced to zero |

Permissive → Strict with a telemetry gate is the one
sequencing choice that almost every production migration
converges to; flipping to `STRICT` before measuring
non-mTLS traffic produces incident, not security.

## 3. Validation steps

(For an architecture lab, "validation" is both document review
and a working smoke test.)

1. Trust-zone map is reviewed and every arrow between zones
   points at a named PEP (not "the VPC" or "the cluster").
2. From a workload in `model-serving`, calling the registry
   artifact endpoint succeeds with mTLS; the same workload
   calling a dataset bucket is denied by the bucket policy
   and the mesh AuthorizationPolicy.
3. From a workload with a forged SPIFFE SAN cert (self-signed,
   not SPIRE-issued), the mesh rejects the TLS handshake
   because the cert is not signed by the trust domain's root.
4. A user without the `ml.inference.invoke` scope hits the
   gateway and gets `403` before the request reaches the
   model server.
5. `kubectl get secret` from inside a training pod does not
   find any static cloud credentials; `aws sts
   get-caller-identity` succeeds via IRSA token exchange.
6. A deliberately-induced "impossible travel" on a human
   session downgrades the gateway's allowed scopes within the
   configured window (if the policy engine is wired in).
7. Breaking the mesh (uninstall sidecar on a serving pod) and
   calling the registry: traffic flows but
   `AuthorizationPolicy` denies at the registry side because
   the request is now anonymous.
8. The SIEM query `source.spiffe_id missing AND
   destination.zone IN (DATA, REG)` returns zero results.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Written set of zero-trust tenets explicitly cites SP 800-207 §2.1 and names which tenets drive which controls | 10 |
| R2 | Trust-zone map lists every zone, its classification, and every inter-zone edge with its PEP | 15 |
| R3 | Workloads have SPIFFE/SPIRE (or equivalent short-lived, attested) identities; **no long-lived service account keys** anywhere | 15 |
| R4 | Human identity via enterprise OIDC + MFA; no local user tables in any service | 10 |
| R5 | Strict mTLS enforced in the mesh with authentication **and** separate AuthorizationPolicy with explicit deny-by-default | 15 |
| R6 | North-south edge (API gateway) validates OIDC tokens with scope-based authorisation; no bypass path | 10 |
| R7 | Data plane (object storage / registry) enforces identity *and* resource policy; audit log captures caller SPIFFE ID | 10 |
| R8 | Admin access is JIT, time-bounded, dual-controlled for sensitive zones; no standing admin | 5 |
| R9 | A phased adoption plan exists; `STRICT` mTLS flip is gated on a telemetry threshold | 10 |

Pass threshold: **80/100**. R3, R5, and R7 are must-haves —
a Zero Trust design without attested workload identities,
without authorisation (only authentication), or without
data-plane enforcement is Zero Trust in name only.

## 5. Common mistakes

1. **"Zero trust = buy a service mesh."** The mesh is one
   PEP. Without named zones, written tenets, and policies on
   the data plane and the human plane, mTLS alone is
   encryption-in-transit, not zero trust.
2. **mTLS without authorisation.** `PeerAuthentication:
   STRICT` proves who called, not what they may do. Pair
   with `AuthorizationPolicy`; otherwise every mesh workload
   is an authenticated anonymous caller.
3. **Bearer tokens that outlive sessions.** Long-lived PATs
   for CI, long-lived kubeconfigs for admin. Replace with
   OIDC-federated short-lived tokens (SPIFFE JWT-SVID or
   cloud workload identity).
4. **Trusting the network for east-west.** "The call came
   from the VPC" is exactly the assumption SP 800-207 §2.1
   tells you not to make. Enforce at the destination,
   always.
5. **Policy engine without observability.** The dynamic risk
   score is only as good as its signals. If the SIEM isn't
   fed, the policy is a rulebook, not an engine.
6. **One trust domain across production and development.**
   A SPIRE trust domain collapses blast radius. Keep prod
   and non-prod in separate trust domains; federate only
   where required.
7. **JIT admin without session recording.** Short duration
   plus no record = no forensics after an incident. Both.

## 6. References

Primary (official standards and project documentation):

- NIST SP 800-207 — Zero Trust Architecture —
  <https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-207.pdf>
- SPIFFE spec (identity format, trust domains) —
  <https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE.md>
- SPIRE — Concepts and attestation —
  <https://spiffe.io/docs/latest/spire-about/spire-concepts/>
- Istio — Security concepts —
  <https://istio.io/latest/docs/concepts/security/>
- Istio — PeerAuthentication / AuthorizationPolicy —
  <https://istio.io/latest/docs/reference/config/security/>
- Linkerd — Server authorization policy —
  <https://linkerd.io/2.15/features/server-policy/>
- OpenID Connect Core 1.0 —
  <https://openid.net/specs/openid-connect-core-1_0.html>
- Envoy — Access logs —
  <https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/access_log>
- AWS — IAM Roles for Service Accounts (IRSA) —
  <https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html>
- GKE — Workload Identity —
  <https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity>

Security / governance context:

- NIST AI Risk Management Framework (MANAGE and MEASURE
  functions — monitoring, continuous risk assessment, and
  incident response map directly to the observability plane
  and policy-engine feedback loop) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST AI 600-1 (Generative AI Profile — section on
  Information Integrity and Information Security applies to
  per-request authorisation on model-serving endpoints) —
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
- OWASP Machine Learning Security Top 10 (ML05: Model Theft
  and ML06: AI Supply Chain Attacks constrain Z-REG and
  Z-SERVE policy) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS (adversary techniques map: initial access via
  stolen credentials, lateral movement inside mesh,
  exfiltration through legitimate paths — each is a Zero
  Trust PEP gap) — <https://atlas.mitre.org/>

Local context:

- Module-level rationale:
  `modules/mod-209-security-compliance/SOLUTION.md`
  (Decision 1 — no long-lived credentials; Decision 3 —
  default-deny networks; Decision 4 — secrets via a secret
  store).
- Prior lab: exercise-01 (K8s security primitives — PSA,
  RBAC, NetworkPolicy, admission) that this exercise
  composes into trust-zone enforcement.
- Engineer-tier zero-trust reference:
  `engineer-solutions/mod-109 ex-07/08` (per module README
  cross-reference).
