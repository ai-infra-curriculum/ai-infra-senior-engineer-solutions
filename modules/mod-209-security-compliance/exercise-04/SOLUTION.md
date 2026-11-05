# SOLUTION — Lab 04: Security Program Design at Senior Scale

Reference solution for
[lab-04-security-program-design.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-209-security-compliance/exercises/lab-04-security-program-design.md).

This is the capstone design exercise. The prior labs built
pieces — hardened clusters, zero-trust zones, an audit frame.
This one is the *program* that owns them: the operating
model, roles, threat model, roadmap, and executive narrative
that turn a collection of controls into an AI-security
practice the organisation can run.

## 1. Solution overview

A security program is the set of long-lived agreements that
let many teams build and operate AI systems safely without a
central security team becoming the bottleneck. NIST AI RMF
organises it by four functions
([NIST.AI.100-1](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf)):

- **GOVERN** — structures, policies, roles, culture. The
  program lives here.
- **MAP** — context, intended use, risk framing.
- **MEASURE** — instrumentation that shows controls work.
- **MANAGE** — responding to risks over time.

The reference program is six layers (document, not org
chart):

1. **Charter and mandate** — why the program exists, what
   authority it has, which risks it owns and which it
   doesn't.
2. **Threat model** — the OWASP ML Top 10 / MITRE ATLAS
   adversary landscape, scoped to the organisation's
   actual systems.
3. **Control catalogue** — the structural controls
   (admission, mesh, lineage, audit) with owners and
   enforcement surface.
4. **Operating model** — RACI, intake, exceptions,
   escalation.
5. **Roadmap** — 12-month rolling plan, sequenced against
   business risk.
6. **Metrics and executive narrative** — what gets
   reported upward, at what cadence, in what format.

Senior additions on top of "a security slide deck":

- **Program is a product.** It has a backlog, a roadmap, a
  release cadence, and SLOs. It is reviewed quarterly.
- **Exceptions are first-class.** Most failures in
  real-world programs come from exceptions becoming the
  norm; the program design includes how exceptions are
  granted, logged, and expired.
- **The exec narrative is a working artifact**, not a
  one-off. The same metrics that run the program drive the
  board slide — one source of truth.

## 2. Worked answer or implementation

### 2a. Charter (one page, signed)

The charter answers the questions an auditor and a new VP
ask on day one.

> **Mission.** Reduce the probability and impact of security
> incidents involving acme's AI/ML systems to a level the
> business has explicitly accepted, through structural
> controls operated by the owners of those systems.
>
> **Scope.** Model training pipelines, model registry,
> inference-serving clusters, feature stores, and the
> development paths that feed them. Out of scope: user
> account security (owned by IT Security), physical
> security, non-ML corporate systems.
>
> **Framework alignment.** NIST AI RMF 1.0; NIST AI 600-1
> for generative-AI components; NIST SP 800-207 for the
> technical architecture; OWASP ML Top 10 and MITRE ATLAS
> as threat references.
>
> **Authority.** The program owner approves exceptions to
> any structural control in scope; the CISO approves
> exceptions that would raise residual risk above the
> organisation's documented appetite.
>
> **Accountabilities.** Program owner: Head of ML
> Platform Security. Risk owner (accepts residual risk):
> VP Engineering. Executive sponsor: CTO.
>
> **Review cadence.** Quarterly program review; annual
> full charter review.

### 2b. Threat model (scoped, not generic)

The OWASP ML Top 10 catalogue is the taxonomy; the program's
threat model picks the subset that applies to *our* systems
and documents why.

| OWASP ML | Risk | Applies to us? | Rationale | Primary control(s) |
|---|---|---|---|---|
| ML01 Input Manipulation | Adversarial evasion | Yes — public-facing inference | Any public endpoint is probed | Rate limiting + adversarial eval on release (MEASURE) |
| ML02 Data Poisoning | Corrupted training data | Yes | We ingest third-party data | Dataset signing + provenance (AC-08 from ex-03) |
| ML03 Model Inversion | Reconstructing training data | Partial | Only for high-privacy models | DP during training on PII sets |
| ML04 Membership Inference | Membership detection | Partial | Same scope as ML03 | DP + output minimisation |
| ML05 Model Theft | Weight exfiltration | Yes | Models are IP | Registry authz + egress policy (ex-02) |
| ML06 AI Supply Chain | Compromised components | Yes | Multi-vendor stack | Signature verification + SBOM (ex-01, ex-03) |
| ML07 Transfer Learning | Poisoned base model | Yes | We use foundation models | Pin + verify + eval base models |
| ML08 Model Skewing | Feedback-loop corruption | Yes — recommender | Online learning on user signal | Guardrails on label sources |
| ML09 Output Integrity | Compromised output path | Yes | We ship model outputs to prod APIs | mTLS + authz on serving (ex-02) |
| ML10 Model Poisoning | Weights tampering | Yes | Shared registry | Signed artifacts + cosign verify (ex-01) |

The matching MITRE ATLAS mapping is kept as a sibling table
([ATLAS Matrices](https://atlas.mitre.org/matrices/ATLAS)) —
each tactic/technique maps to one or more controls in the
catalogue so attackers' moves have named defenders.

### 2c. Control catalogue (owners and enforcement surface)

The structural controls in the program, each tagged with a
single enforcement surface (where it runs) and a single
owner. A control is only real if it has both.

| ID | Control | Framework | Enforcement surface | Owner |
|---|---|---|---|---|
| C-AUTH-1 | Workload identity, no long-lived creds | AI RMF GV-1.6; SP 800-207 §2.1 | Cloud IAM + SPIRE | Platform SRE |
| C-NET-1 | Default-deny network; explicit east-west authz | SP 800-207 §3.1; OWASP ML05 | Service mesh | Platform SRE |
| C-IMG-1 | Image signature + provenance verification | OWASP ML06; AI RMF MS-2.7 | Kyverno admission | ML Platform |
| C-DATA-1 | Dataset classification + lineage per training run | AI RMF MP-4.1; OWASP ML02 | Pipeline metadata store | ML Platform |
| C-MODEL-1 | Model card per released model | AI RMF MP-5.1 | Registry | Model owners |
| C-EVAL-1 | Pre-release eval on safety + robustness benchmarks | AI RMF MS-2.5; AI 600-1 MS-2.3-003 | CI gate | ML Platform |
| C-MON-1 | Drift/degradation alerts tied to control IDs | AI RMF MS-2.6 | Prometheus / SIEM | SRE |
| C-SEC-1 | Audit log retention ≥1 year, immutable store | AI RMF MG-4.3 | SIEM / object-lock bucket | Security |
| C-IR-1 | AI incident runbook + quarterly tabletop | AI RMF MG-2.4 | Documented + rehearsed | Security |
| C-EXC-1 | Exception process with expiry and residual-risk record | AI RMF GV-1.7 | Exception register | Program owner |

### 2d. Operating model (RACI)

| Activity | Program owner | Platform SRE | ML Platform | Security | Model owner | Exec sponsor |
|---|---|---|---|---|---|---|
| Approve new control into catalogue | A | C | C | R | I | I |
| Enforce an existing control (operational) | I | R (infra) | R (pipeline) | A | — | — |
| Grant an exception (short) | A, R | C | C | C | Requests | I |
| Grant an exception (raises residual risk) | R | C | C | A | Requests | Approves |
| Investigate control failure | A | R | R | C | — | I |
| Author / revise a model card | I | — | C | I | R, A | — |
| Declare an AI incident | A | C | C | R | I | I |
| Quarterly program review | R | C | C | C | — | A |

Legend: R = Responsible, A = Accountable, C = Consulted,
I = Informed. One `A` per row, by design.

### 2e. Intake and exception workflow

A new system or a change to an existing one enters the
program via a lightweight intake form answered by the model
owner. Three questions determine the review depth:

1. Does the model make a decision about a person?
2. Does the training data include PII, protected-class
   attributes, or regulated content?
3. Is the model generative, and if so, does it produce
   outputs shown to users or ingested by other systems?

Yes to any → full review (threat model, model card, eval
plan). No to all → attestation-only; owner confirms the
default controls apply.

Exceptions are the lever that keeps the program moving
without accreting risk:

```
Exception: E-2026-11-04
  Control(s): C-IMG-1 (image signature verification)
  Scope: ghcr.io/acme/experimental/* in mlp-exp-use1 only
  Reason: PoC for new vendor; vendor signing in Q1 2027
  Compensating controls:
    - Experimental cluster is isolated from Z-DATA by
      NetworkPolicy (confirmed with policy report).
    - Images are scanned on-push.
  Granted by: Program owner (date, signature)
  Expiry: 2027-01-31
  Residual risk: Low-Medium
  Review: 2027-01-15 (two weeks before expiry)
```

Every open exception is in a register that the program
review reads top to bottom.

### 2f. Rolling 12-month roadmap

A reference shape — concrete weeks are the student's. The
sequencing rationale matters more than the dates.

| Quarter | Theme | Deliverables | Risk reduced |
|---|---|---|---|
| Q1 | Foundations | Charter signed; control catalogue published; intake process live; SIEM retention ≥1y | Program coherence |
| Q2 | Supply chain | C-IMG-1 Enforce everywhere; SBOM per release; base-model verification process (ML06, ML07) | Supply-chain attack path |
| Q3 | Data and lineage | C-DATA-1 every training run; dataset classification 100% of in-scope stores (ML02) | Data poisoning + privacy |
| Q4 | Measurement and response | C-EVAL-1 blocking release; adversarial eval for public endpoints; AI incident tabletop; residual-risk report to board | Evasion + incident readiness |

Why this order? Supply chain (Q2) before data (Q3): a signed
pipeline writing poisoned data is still poisoned, but an
*unsigned* pipeline with clean data is already compromised.
Supply chain is the perimeter of lineage.

### 2g. Metrics and executive narrative

Four leading indicators (predict risk) and four lagging
indicators (measure incidents). All are produced by the
same pipelines that enforce the controls — no separate
reporting sheet.

Leading:

- % of in-scope workloads with workload-identity-only
  (target: 100%).
- % of production models with a current model card
  (target: 100%).
- Open control exceptions by age (median, p95).
- Time from threat intel publication → control catalogue
  update, per OWASP ML / ATLAS update.

Lagging:

- AI incidents per quarter by severity.
- Audit findings by severity, open and closed.
- Quarterly tabletop outcome (red/amber/green).
- Residual risk trend over time.

Executive narrative is one page per quarter: top three
risks, what moved last quarter, what will move next.

### 2h. Seams with the rest of the organisation

The program does not operate alone. Explicit seams prevent
drift:

- **Legal / Privacy** — review of model cards and data
  classifications; owns PII/GDPR assessments.
- **IT Security / SOC** — operates SIEM and 24/7 on-call;
  owns detection for the alerts C-MON-1 raises.
- **Enterprise Risk** — residual-risk acceptance over the
  appetite threshold; owns the risk register.
- **Vendor Management** — approves third-party AI vendors
  and model providers (AI 600-1 **GV-6** subcategories).

## 3. Validation steps

(For a program-design exercise, validation is review of the
artifact set against a rubric; §3 names concrete tests.)

1. Charter is signed by the program owner, the risk
   owner, and the executive sponsor; stored in a durable
   location (not an email thread).
2. Threat model enumerates OWASP ML Top 10, picks in-scope
   items, and justifies out-of-scope items.
3. Every control in the catalogue has an ID, an
   enforcement surface, and a single owner.
4. RACI has exactly one `A` per activity; no activity is
   `A` by more than one role.
5. The intake form rejects a submission that's missing any
   of the three screening questions.
6. The exception register has no exception whose expiry
   has passed without a review entry (an expired exception
   that has silently continued is a program failure).
7. The 12-month roadmap sequences supply-chain controls
   before data-provenance controls, with a written reason.
8. The executive quarterly is one page, references the
   leading/lagging indicator set, and does not depend on
   hand-assembled data.
9. The seam list names a human-readable owner on the
   other side of every seam.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Written charter with scope, authority, accountabilities, and review cadence | 10 |
| R2 | Threat model keyed to OWASP ML Top 10 and MITRE ATLAS; each item has a scope decision + rationale + primary control | 15 |
| R3 | Control catalogue — each row has ID, framework reference, enforcement surface, single owner | 15 |
| R4 | RACI covers at least: new control, exception, incident, model card, quarterly review; single `A` per row | 10 |
| R5 | Intake process explicitly filters generative, PII, and automated-decision cases into full review | 10 |
| R6 | Exception process has expiry, compensating-controls field, residual-risk rating, and a register | 10 |
| R7 | Rolling 12-month roadmap with written rationale for sequencing (not arbitrary quarters) | 10 |
| R8 | Metrics split leading vs lagging; metrics derive from the same pipelines that run the controls | 10 |
| R9 | Executive narrative artifact (one page / quarter) with top-three-risks framing | 5 |
| R10 | Seams to Legal, IT Security, Enterprise Risk, and Vendor Management are named with owners on both sides | 5 |

Pass threshold: **80/100**. R2, R3, and R6 are must-haves —
a program without a scoped threat model, without named
control owners, or without exception discipline is a
document, not a program.

## 5. Common mistakes

1. **Charter without authority.** "The program recommends…"
   with no approval authority produces advice nobody
   actions. The charter must name what the program can
   decide unilaterally and what it must escalate.
2. **Importing the full OWASP ML Top 10 as "in scope".**
   The list is a catalogue; the program's threat model is a
   subset with rationale. Claiming all of it is the same as
   claiming none of it operationally.
3. **Controls owned by "the platform team".** A team is not
   an accountable entity — a role within it is. Platform
   SRE Lead owns C-NET-1, not "SRE".
4. **Exceptions without expiry.** Exceptions become
   permanent by default; expiry + mandatory review is the
   only mechanism that keeps the catalogue honest.
5. **Model cards treated as marketing.** Cards are the MAP
   artifact; shipping a card that omits foreseeable misuse
   or evaluation slices is non-compliance, not style.
6. **Metrics the program measures itself with that aren't
   the same as the ones enforcement produces.** Separate
   reporting = drift between what the program *claims*
   and what the pipeline *does*.
7. **No seam to Legal / Privacy.** Data classifications,
   consent basis, PII disposition — none of these is a
   security decision alone. The seam is required.
8. **Roadmap of 20 projects.** A 12-month roadmap with 20
   deliverables is a wishlist. Pick 4–6 themes,
   justify the sequencing, and ship them.
9. **"We'll do a tabletop once things settle."** Things
   never settle. The first tabletop is always the one
   where the runbook has gaps; that is the point.

## 6. References

Primary (official standards and project documentation):

- NIST — AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- NIST AI 100-1 — AI RMF 1.0 —
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf>
- NIST AI 600-1 — Generative AI Profile —
  <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>
- NIST SP 800-207 — Zero Trust Architecture —
  <https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-207.pdf>
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — Matrices —
  <https://atlas.mitre.org/matrices/ATLAS>
- Mitchell et al. — Model Cards for Model Reporting —
  <https://arxiv.org/abs/1810.03993>

Practitioner reference (programs that publish their
structure can be read as examples, not as authority):

- VeriSwarm Trust Center — practitioner-grade AI-security
  program artifacts (control summaries, model-card
  conventions) — <https://veriswarm.ai/trust>

Local context:

- Module-level rationale:
  `modules/mod-209-security-compliance/SOLUTION.md`
  (all six decisions appear as controls in §2c; this
  exercise is the programmatic frame around them).
- Prior labs supplying the pieces:
  - exercise-01 (K8s security): C-IMG-1 and C-NET-1
    enforcement surface.
  - exercise-02 (Zero-trust): C-AUTH-1 and C-NET-1
    architecture.
  - exercise-03 (Audit): control-to-evidence mapping and
    model-card discipline.
- Engineer-tier implementation references:
  `engineer-solutions/mod-109 ex-07/08`,
  `security-solutions/projects` (per module README
  cross-reference).
