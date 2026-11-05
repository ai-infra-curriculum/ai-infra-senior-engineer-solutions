# SOLUTION — Lab 02: Architecture Decision Record (ADR) Writing

Reference solution for
[lab-02-adr-writing.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-210-technical-leadership/exercises/lab-02-adr-writing.md).

An ADR is the written output of a decision. It is a
document a successor engineer can read six months later and
understand *why* a system is shaped the way it is, *what was
considered and rejected*, and *under what conditions the
decision should be revisited*. ADRs are the senior-engineer
equivalent of a laboratory notebook — the thinking is the
artefact, not just the result.

This lab's deliverable is a real ADR for a real-looking
decision, graded against a rubric that rewards the pattern
the module teaches: alternatives considered, trade-offs
named, reversibility stated.

## 1. Solution overview

The ADR format used in this reference is Michael Nygard's
("Lightweight Architecture Decision Records") because it is
the one the ADR community converged on and the one
`adr-tools` and most templates implement
([ADR GitHub organisation](https://adr.github.io/),
 [Michael Nygard — Documenting architecture decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)).

The minimal section set is:

1. **Title and metadata** — short, assertive, numbered;
   author; date; status (proposed / accepted / deprecated /
   superseded-by).
2. **Context** — the forces at play. Technical, business,
   organisational. What problem is the decision solving,
   and what constraints were real at the time of writing?
3. **Decision** — the choice, stated as a sentence a
   reader can disagree with. "We will do X" — not "we are
   considering X."
4. **Alternatives considered** — the viable options that
   were examined and rejected. This section is the
   senior-engineer differentiator; it is the one that is
   missing from most junior ADRs and the one that lets a
   successor revisit the decision honestly.
5. **Consequences** — positive, negative, and neutral
   effects. Second-order effects ("this means we now need
   X") belong here.
6. **Reversibility and revisit conditions** — Jeff Bezos's
   "type 1 vs. type 2 decisions" lens
   ([Jeff Bezos 2015 shareholder letter](https://www.aboutamazon.com/news/company-news/2016-letter-to-shareholders)).
   Which is this, and what would make us re-open it?

For AI/ML platform decisions specifically, the ADR should
ground itself in the applicable standards up front:

- **NIST AI RMF 1.0** ([NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework))
  — GOVERN / MAP / MEASURE / MANAGE framing when the
  decision carries risk or governance weight.
- **OWASP ML Top 10** ([OWASP ML Top 10](https://owasp.org/www-project-machine-learning-security-top-10/))
  and **MITRE ATLAS** ([MITRE ATLAS](https://atlas.mitre.org/))
  — for ML-security-shaped decisions (supply chain,
  evasion, inference abuse).

The worked ADR below applies this template to a realistic
ML-platform decision.

## 2. Worked answer or implementation

### 2a. The worked ADR

Filename: `docs/adr/0007-model-registry-choice.md`.

---

```markdown
# ADR-0007: Adopt MLflow Model Registry as the system of
# record for production model artefacts

- Status: Accepted
- Date: 2026-02-14
- Author: J. Rivera (platform)
- Reviewers: ML platform tech-leads; security; MLOps lead
- Supersedes: ADR-0003 ("Ad-hoc S3 prefix as model store")

## Context

We serve 40+ models across 12 product teams. Today, each
team publishes trained artefacts to an S3 prefix of its own
choosing, hands the path to the serving team, and informally
tracks versions in a shared spreadsheet. This causes three
recurring incidents per quarter (wrong-version deploys, lost
lineage, and one near-miss with a model that had never been
evaluated at all).

Forces:

- **Governance.** NIST AI RMF MAP-4.1 and MANAGE-4.1 require
  that model artefacts are traceable to their training run,
  dataset version, and evaluation results. Our current
  state cannot demonstrate this to an auditor.
- **Security.** OWASP ML Top 10 ML06 (AI Supply Chain
  Attacks) requires that the artefact a serving pod loads
  is the one that was approved. S3 prefixes with team-level
  write access do not satisfy this — any team member can
  overwrite "prod" without a record.
- **Operations.** The serving platform (KServe) expects a
  single, stable reference per model; constructing that
  reference from team-chosen S3 prefixes requires per-team
  adapters.
- **Cost.** We have ~2 FTE-weeks per quarter to spend on
  this; a full build-vs-buy with internal ML-registry work
  is not feasible in that window.

## Decision

We will adopt **MLflow Model Registry** (the open-source
project, self-hosted on our existing Postgres + S3 stack)
as the system of record for production model artefacts.
Every model served from a production serving cluster must
be published to and promoted via the registry; the KServe
inference service template will pull artefacts only from
the registry's `models:/<name>/<stage>` URI.

## Alternatives considered

1. **Status quo (S3 prefixes + spreadsheet).** Rejected.
   Fails NIST AI RMF MAP-4.1 (lineage) and OWASP ML06
   (supply-chain integrity). Incident rate is the evidence.
2. **Build an internal registry.** Rejected. Our platform
   team is three engineers; a registry is at least two
   quarters of work if we count metadata APIs, UI, and
   migration. The decision window is weeks, not months.
3. **Weights & Biases Models (SaaS).** Rejected for *this*
   decision, held open for revisit. Compliance has not yet
   finished the data-residency review; two of our models
   are trained on data we cannot send to a US-based SaaS
   today. Revisit when the compliance review closes (see
   §Revisit).
4. **Vertex AI Model Registry / SageMaker Model Registry
   (cloud-native).** Rejected. We are multi-cloud by
   `modules/mod-205-multi-cloud/SOLUTION.md`'s decision;
   adopting a per-cloud registry creates the per-team
   adapter problem one level up. MLflow runs in both.

## Consequences

Positive:

- Single, auditable promotion path: `Staging -> Production`
  becomes a logged event with the acting user, timestamp,
  and the model-version SHA.
- KServe templates simplify to one URI shape.
- Lineage via MLflow's `run_id` -> dataset-fingerprint chain
  closes the NIST RMF MANAGE-4.1 reproducibility gap.
- Cosign-signed model artefacts (the serving admission
  policy already enforces this for images; we extend it to
  model blobs in a follow-up ADR) will be verifiable before
  load.

Negative:

- MLflow's server is now a production dependency with its
  own on-call rotation. We will need to run it HA and back
  up its Postgres.
- The migration from S3-prefix references is manual for the
  40 existing models. Estimated 1 FTE-week.
- MLflow's RBAC is coarse. Fine-grained "Team A can only
  promote Team A's models" requires a thin authz proxy or
  the `mlflow-auth` plugin. See follow-up ADR-0008.

Neutral:

- Model artefacts still live in S3; we are not changing the
  storage layer, only the metadata and promotion layer.

## Revisit conditions

This is a **type-2 decision** (reversible with migration
cost measured in weeks, not years). Revisit if:

- **Compliance opens SaaS registries.** Weights & Biases
  Models or a newer entrant may offer features (automated
  eval gating, model-card integration) worth the switch.
- **MLflow registry development stalls** (no release in 12
  months, or a security CVE unpatched for > 90 days).
- **We acquire a team already standardised on a different
  registry.** One-way migrations are expensive; a merger
  is one of the few events that justifies re-opening.

## References

- MLflow Model Registry docs —
  <https://mlflow.org/docs/latest/model-registry.html>
- NIST AI RMF 1.0 (MAP-4.1, MANAGE-4.1) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP ML Top 10 (ML06) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- ADR format — Michael Nygard —
  <https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions>
- Supersedes: ADR-0003 (ad-hoc S3 prefix model store).
```

---

### 2b. Why the ADR looks the way it does

Each section is doing a specific job:

- **Context** names the standards (NIST, OWASP) and the
  operational reality (incident rate, team size, decision
  window). A reader six months from now does not have to
  reconstruct what was true.
- **Decision** is a single sentence, present tense, stated
  as a commitment. A successor can point to the sentence
  and say "that is what was decided."
- **Alternatives considered** names four options and
  explains the rejection of each. Rejections cite constraints
  ("compliance not finished"), not preferences. Option (3)
  is explicitly held open with a revisit trigger — an honest
  "not now" rather than a disguised "no".
- **Consequences** has three columns of effect, including
  second-order ones ("MLflow is now a production dependency
  with on-call"). The ADR does not hide the costs.
- **Revisit conditions** names the decision type
  (reversible) and the triggers that would re-open it. Most
  ADRs omit this; it is the senior-engineer addition that
  lets the organisation revisit decisions on evidence, not
  on who-shouts-loudest.

### 2c. A one-page ADR template (copy-paste)

```markdown
# ADR-NNNN: <short, assertive title>

- Status: Proposed | Accepted | Deprecated | Superseded by ADR-XXXX
- Date: YYYY-MM-DD
- Author: <name, role>
- Reviewers: <names / teams>
- Supersedes / Related: ADR-XXXX (if any)

## Context

What is the problem? What forces (technical, business,
organisational, regulatory) apply? What is true *now* that
matters?

## Decision

We will <do X>. One sentence a reader can agree or disagree
with.

## Alternatives considered

1. <Option> — rejected because <specific reason, with
   constraint named>.
2. <Option> — rejected because ...
3. <Option> — held open; revisit when <trigger>.

## Consequences

Positive:
- ...

Negative:
- ...

Neutral:
- ...

## Revisit conditions

This is a type-1 (hard-to-reverse) / type-2 (reversible)
decision. Revisit if:

- ...

## References

- <official standards / docs>
- <related ADRs>
```

## 3. Validation steps

1. Can a stranger read the Context section alone and
   explain the problem back? If not, the context is thin.
2. Is the Decision one sentence? If it spans a paragraph,
   either multiple decisions are hiding, or the decision
   isn't crisp enough.
3. Are there at least two Alternatives and at least one
   rejection reason that cites a constraint rather than a
   preference? If not, the ADR looks pre-decided.
4. For each Consequence listed as positive, is there at
   least one listed as negative? ADRs with only
   upsides are advertising, not decisions.
5. Does Revisit conditions name a *trigger* (an observable
   event) and not just a timeframe? "Revisit in a year" is
   weak; "revisit when the compliance review closes" is
   strong.
6. If the decision touches AI/ML risk, does the ADR cite
   at least one of NIST AI RMF, OWASP ML Top 10, or MITRE
   ATLAS? These are the standards the module expects
   grounding in.
7. Number and filename match the ADR directory convention
   (zero-padded, kebab-case title), and the Status line is
   current.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Title is short, assertive, and present-tense; metadata (status, date, author) is complete | 5 |
| R2 | Context names forces (technical, business, regulatory) and cites at least the applicable NIST / OWASP / ATLAS standards when relevant | 15 |
| R3 | Decision is a single commitment sentence; there is no hedging ("we may", "we are considering") in the Decision section | 10 |
| R4 | At least 2 viable alternatives are listed and rejected with constraint-based reasons, not preference | 20 |
| R5 | Consequences are listed with positive, negative, and (where applicable) neutral effects — including second-order consequences | 15 |
| R6 | Revisit conditions name observable triggers (not only timeframes) and state decision type (reversible / not) | 15 |
| R7 | Supersedes / related ADRs are linked; the ADR number is unique in the repo and the filename follows the convention | 5 |
| R8 | References section lists the exact URLs of the standards and tools cited; no uncited claims | 10 |
| R9 | ADR fits on one to two pages when rendered; density is prose + lists, not slide-deck bullets | 5 |

Pass threshold: **75/100**. R4 and R5 are must-haves — an
ADR without rejected alternatives and without named
negative consequences is a proposal, not a decision record.

## 5. Common mistakes

1. **No alternatives considered.** The author decided
   first and documented second. The successor cannot
   revisit the decision honestly because they do not know
   what was ruled out.
2. **Alternatives exist but are strawmen.** Three options
   listed, two of which are obviously bad. The real
   competing option is missing. A good test: show the ADR
   to someone who advocates a different approach and ask
   if their option is fairly represented.
3. **Decision stated in future / hedged tense.** "We are
   evaluating X" or "we may adopt X" is a proposal, not an
   ADR. Change the Status back to Proposed and name the
   open questions.
4. **No negative consequences.** Every real decision has a
   cost. If the ADR lists only benefits, the author did not
   consider the costs or is advocating rather than
   recording.
5. **Revisit conditions reduced to "revisit annually".**
   Calendar triggers are weak. Observable triggers
   (incident, dependency change, compliance ruling, team
   acquisition) are what re-opens decisions in practice.
6. **ADR as design doc.** 15 pages of architecture diagrams
   and API shapes. An ADR records *the decision*; the
   design belongs in a separate design doc the ADR links
   to.
7. **Status drift.** The ADR says Proposed; the decision
   has been live in production for a year. Status is
   authoritative only if it is maintained.
8. **No numbering / no index.** Forty ADRs with the same
   filename pattern and no index make them unfindable.
   Number, index, and link from the architecture README.
9. **ADR for a decision that is not architectural.**
   Choosing between tabs and spaces does not need an ADR.
   Reserve ADRs for decisions that shape the system or
   constrain future work.
10. **Citing standards without reading them.** Dropping
    "NIST AI RMF" into an ADR without naming the specific
    subcategory the decision addresses is cargo-culting.
    Name the subcategory and quote the relevant line.

## 6. References

Primary (ADR community references and official standards):

- Michael Nygard — "Documenting architecture decisions"
  (the canonical ADR format this lab uses) —
  <https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions>
- ADR GitHub organisation — templates, tools, community
  variants —
  <https://adr.github.io/>
- NIST AI Risk Management Framework 1.0 (used in the
  worked ADR's Context and in R2 of the rubric) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 (ML06 cited in
  the worked ADR's Context) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS (cited when ADRs address ML-adversary
  concerns) —
  <https://atlas.mitre.org/>

Local context:

- Module-level framing: `modules/mod-210-technical-leadership/SOLUTION.md`
  §"The tech spec (exercise 01)" — adjacent format whose
  Alternatives-and-Risks sections are the same shape as
  this ADR's Alternatives and Consequences.
- Multi-cloud constraint referenced in the worked ADR:
  `modules/mod-205-multi-cloud/SOLUTION.md`.
- Supply-chain / admission policy context referenced in
  the worked ADR's Consequences (cosign-signed artefacts):
  `modules/mod-209-security-compliance/SOLUTION.md` §2f.
