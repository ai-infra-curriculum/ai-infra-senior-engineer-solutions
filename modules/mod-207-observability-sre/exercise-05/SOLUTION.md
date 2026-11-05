# SOLUTION — Lab 05: SRE On-Call Program

Reference solution for
[lab-05-sre-on-call-program.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-207-observability-sre/exercises/lab-05-sre-on-call-program.md).

This is a **design exercise**. The deliverable is not code; it is
an on-call program — the rotation shape, the paging policy, the
runbook contract, the postmortem discipline, and the humane
practices that let a team operate a service around the clock
without burning out. Per the AICG design-exercise contract, the
sections below cover a worked example, decision rationale, and
grading rubric.

## 1. Solution overview

A senior-tier on-call program has six load-bearing components,
derived from the Google SRE book
([Site Reliability Engineering — Chapter 11: Being On-Call](https://sre.google/sre-book/being-on-call/)):

1. **Rotation** — who is on call, for how long, and how they
   hand off.
2. **Paging policy** — what wakes a human; what goes to a
   ticket; what gets dropped.
3. **Runbook contract** — every pageable alert has a reachable
   runbook with a first command.
4. **Escalation** — when and how to pull in the next person;
   named secondary on-call and manager escalation.
5. **Incident response** — severity levels, incident-commander
   role, status-page cadence, communications.
6. **Postmortem culture** — blameless template, action-item
   tracking, reverification.

Senior-tier additions beyond "we have an on-call":

- Error-budget-driven decisions about when to pause feature
  work (module-level [Decision 6](../SOLUTION.md)).
- Compensation and humane constraints (not more than X pages
  per shift, etc.) treated as a program invariant, not a wish.
- Explicit links from the program to the SLO definitions
  ([exercise-02](../exercise-02/SOLUTION.md)) and the
  observability stack ([exercise-01](../exercise-01/SOLUTION.md)).

## 2. Worked answer or implementation — reference on-call program

### 2a. Rotation shape

For a team of **six to eight engineers** (the smallest size at
which weekly on-call is sustainable per the SRE book — see
[SRE book — Being On-Call: Balanced On-Call](https://sre.google/sre-book/being-on-call/)):

- **One-week rotation.** Monday 10:00 to the following Monday
  10:00 (local to the primary team).
- **Primary + secondary.** Secondary covers primary's gaps
  (meetings, sleep) and is the first escalation.
- **Follow-the-sun** if the team spans ≥ 8 h of timezone
  separation: primary rotates across regions every 12 h,
  secondary is the other region.
- **Compensation.** Per the SRE book, on-call time is treated
  as explicit work, not as volunteering. Reference policy:
  carried time off or on-call-pay stipend; the organization's
  exact form is a people-ops decision.

Hand-off is a **synchronous 15-minute call** between outgoing
and incoming primary at the start of the rotation, covering:

- Any open incidents.
- Any ongoing degradation / flaky alerts that aren't yet
  pages.
- Any planned work (releases, chaos experiments, migrations)
  during the upcoming rotation.

A hand-off document template lives in the service repo and is
updated during the call, so the next person inherits a written
state, not a verbal one.

### 2b. Paging policy — the three-tier filter

Not every alert pages. The reference three-tier policy (routed
by Alertmanager severity labels from
[exercise-01](../exercise-01/SOLUTION.md)):

| Tier | Severity label | Channel | Human SLA |
|---|---|---|---|
| Page | `severity=page` | PagerDuty / Opsgenie / phone | Acknowledge ≤ 5 min |
| Ticket | `severity=ticket` | Slack channel + Jira | Acknowledge ≤ 4 h in business hours |
| Info | `severity=info` | Dashboard / Grafana annotation only | No SLA |

A rule of thumb from the SRE book: **if it would be acceptable
to ignore the alert until morning, it is not a page**. Everything
else goes in `ticket`.

Pages must satisfy four properties (adapted from
[SRE book — Being On-Call: What to do when paged](https://sre.google/sre-book/being-on-call/)):

- **Actionable.** The on-caller can do something in the next 30
  minutes.
- **Investigable.** The alert has a runbook URL and a dashboard
  URL.
- **User-impacting.** The SLO is actually threatened
  (multi-window multi-burn-rate shape; see
  [exercise-02](../exercise-02/SOLUTION.md)).
- **Novel.** If the same alert has fired three times this
  rotation with the same fix, it is a bug in the alert or the
  code — fix one or the other.

The paging policy also sets humane caps:

- **No more than 2 pages per shift on average.** More than that
  is a program bug; the sprint after will prioritize alert
  tuning.
- **Pages between 00:00 and 06:00 local require a post-shift
  review.** A sleep-disrupting page that was not SLO-critical is
  an alert bug to fix.

### 2c. Runbook contract

Every pageable alert has a runbook URL in its annotation
(enforced by the CI gate from
[exercise-02](../exercise-02/SOLUTION.md)). A runbook has a
fixed shape:

```markdown
# Runbook: ApiAvailabilityFastBurn

## Alert
Fires when api availability error ratio > 1.44% over the last
hour, confirmed by 5-minute window.

## First command
    kubectl -n production top pods -l app=api
Then:
    kubectl -n production logs -l app=api --tail=200 | grep ' 5[0-9]{2} '

## Dashboards
- Service health: https://grafana.example.com/d/api-health
- SLO burn: https://grafana.example.com/d/api-slo

## Known causes (ordered by frequency)
1. Feature-store latency spike → check `feast_materialization_lag_seconds`.
2. Model-server pod OOM → check `kube_pod_container_status_restarts`.
3. Upstream DB failover → check `db_connection_errors_total`.

## Mitigations
1. Roll back the last deploy: `argocd app rollback api`.
2. Scale out: `kubectl scale deploy/api --replicas=+2`.
3. Fail over to challenger model: alias flip, see
   `engineer-solutions/mod-106/exercise-04`.

## Escalation
If none of the above resolve within 15 min:
1. Page secondary on-call.
2. Page the team's manager if secondary has not acknowledged
   within 10 min.
```

The **first command** is the single most important field. An
on-caller paged at 3am opens the runbook URL and runs one
command; the output narrows the branches. A runbook whose first
section is three paragraphs of context has failed the user.

### 2d. Escalation

Reference escalation policy (Alertmanager + pager provider):

1. Page primary. Acknowledge SLA 5 min.
2. If not acknowledged in 10 min, page secondary.
3. If secondary not acknowledged in 10 min, page manager.
4. Manager's responsibility is to find a human, not to debug.

Escalation is always **up** (to the manager) when a human is
needed, **sideways** (to a sibling team) when domain expertise
is needed, **never silent**. The on-caller may always escalate
immediately without first attempting to resolve; "I'm out of my
depth" is a valid call, not a failure.

### 2e. Incident response

Severity ladder (consistent with module-level
[Decision 3](../SOLUTION.md)):

| Severity | Definition | Response |
|---|---|---|
| SEV1 | Production is down for a significant share of users, or SLO is burning faster than the error budget allows; data loss or integrity risk | Incident commander, status page update ≤ 15 min, hourly updates, all-hands as needed |
| SEV2 | Significant degradation; a page-tier SLO alert is firing; recovery requires engineering intervention | Incident commander, status page update ≤ 30 min, updates every 2 h |
| SEV3 | Degraded but working; no active user complaint; can be resolved during business hours | Ticket; no incident commander required |

Roles ([SRE book — Managing Incidents](https://sre.google/sre-book/managing-incidents/)):

- **Incident commander (IC).** Coordinates. Does not fix.
  Assigns tasks.
- **Operations lead (OL).** Executes mitigations. Reports to
  IC.
- **Communications lead (CL).** Writes status-page updates;
  liaises with support and leadership.
- **Scribe.** Captures the timeline into the incident doc.

Small incidents (SEV2 with one engineer) collapse IC + OL into
one person; SEV1 always separates them.

### 2f. Status page

Status-page updates are **automated-first, manual-confirmed**
(per module-level mistake #4):

- A degraded SLO burn automatically posts a "we are
  investigating" entry.
- The CL confirms or revises the text within the SLA window
  for the severity.

The reference uses Statuspage.io or an OSS equivalent (Cachet,
Gatus); the strategy from
[exercise-04](../exercise-04/SOLUTION.md) names which one.

### 2g. Postmortem

Blameless template committed in the service repo
([SRE book — Postmortem Culture](https://sre.google/sre-book/postmortem-culture/)):

```markdown
# Postmortem: <incident short title>

## Summary
<2–3 sentences: what, when, impact.>

## Impact
- Duration: <start to resolution, UTC>.
- Users affected: <number / fraction, or qualitative>.
- SLO impact: <error budget consumed>.

## Timeline (UTC)
- HH:MM — <event>
- HH:MM — <event>

## Root cause
<The defect in the system that allowed the incident. Blameless:
describe the system, not the person.>

## What went well
<What did the team do that worked.>

## What went badly
<What the team wants to improve. Blameless.>

## Action items
| # | Action | Owner | Due | Tracked in |
|---|---|---|---|---|
| 1 | ... | ... | YYYY-MM-DD | <ticket link> |

## Reverification
How we'll confirm the action items actually prevent recurrence
(usually: a chaos experiment reproducing the trigger).
```

Action items go into the same tracker as feature work, are
visible on the service health dashboard, and are reviewed in
the team's weekly sprint planning until closed. The postmortem
is **shared broadly** — across teams for SEV1 and SEV2, within
the team for SEV3 — because the point is organizational
learning, not individual review.

### 2h. Error-budget policy — the governance tie-in

Per module-level [Decision 6](../SOLUTION.md), the on-call
program has an explicit error-budget policy:

- **Budget consumed ≥ 100% for the window** → feature freeze
  until reliability work restores ≥ 50% headroom.
- **Budget consumed ≥ 50%** → weekly SLO review in team
  standup; new feature work requires named justification.
- **Budget consumed < 50%** → normal operations.

The policy is a contract, not a negotiation (module-level
trade-off §"Error budget conversations are political"). The PM
agrees to the policy when the SLO is adopted; the on-call
program enforces it.

### 2i. One-paragraph pitch

> Six-to-eight-engineer team on a one-week primary + secondary
> rotation, with a three-tier paging policy (page / ticket /
> info) that caps pages at two per shift. Every pageable alert
> has a runbook whose first field is one command the on-caller
> can run. Severity levels tie directly to SLO burn; an
> incident commander coordinates everything at SEV1 or SEV2.
> Postmortems are blameless, their action items are tracked
> alongside feature work, and the service's error-budget state
> drives feature-freeze decisions. We land it in 60 days
> starting with the paging-policy audit.

## 3. Validation steps

This is a design review, not a test run. A submission is
validated by checking that the student deliverable:

1. Describes all six components in §1 (rotation, paging policy,
   runbook contract, escalation, incident response,
   postmortems).
2. States the rotation shape with team size and shift length,
   and acknowledges humane caps (pages-per-shift, night-page
   review).
3. Includes a three-tier paging policy where `page` severity is
   explicitly tied to SLO burn (not to arbitrary thresholds).
4. Provides a runbook template whose first field is a single
   command.
5. Defines a severity ladder tied to SLO impact and names the
   incident-response roles.
6. Includes a blameless postmortem template with action items
   tracked against a reverification criterion.
7. States an error-budget policy with explicit thresholds and
   consequences.
8. References the SLOs ([exercise-02](../exercise-02/SOLUTION.md))
   and stack ([exercise-01](../exercise-01/SOLUTION.md)) the
   program depends on, rather than redefining them.

A successful submission reads like a program a new on-caller
could be onboarded against in a day.

## 4. Rubric / review checklist

Weightings sum to 100. Below 70 → rewrite; 70–84 →
pass-with-comments; 85+ → ship.

| # | Criterion | Weight |
|---|---|---|
| R1 | All six components present and scoped (rotation / paging / runbook / escalation / incident response / postmortem) | 10 |
| R2 | Rotation shape names team size, shift length, hand-off mechanism, and compensation posture | 10 |
| R3 | Three-tier paging policy with page tier explicitly tied to SLO burn-rate alerts | 15 |
| R4 | Humane caps stated (pages-per-shift, night-page review); policy for when caps are breached | 5 |
| R5 | Runbook template includes a first-command field, dashboard URLs, known causes, mitigations, and escalation | 15 |
| R6 | Escalation policy names timeouts and the next hop; "escalate immediately" explicitly permitted | 5 |
| R7 | Severity ladder is defined in terms of SLO impact; incident-commander role is named and distinct from operations lead | 10 |
| R8 | Postmortem template is blameless, has action-item tracking and a reverification criterion | 10 |
| R9 | Error-budget policy stated with explicit thresholds and consequences | 10 |
| R10 | Links to [exercise-01](../exercise-01/SOLUTION.md), [exercise-02](../exercise-02/SOLUTION.md), and the module-level SOLUTION are present | 5 |
| R11 | No invented facts, metrics, or incidents — claims tied to official standards or clearly labeled local context | 5 |

R3, R5, and R9 are must-haves — a program without SLO-tied
paging, first-command runbooks, or an error-budget policy fails
the senior bar regardless of total.

## 5. Common mistakes

1. **Pages that aren't tied to SLOs.** Threshold alerts fire on
   infrastructure counters, on-caller debugs for an hour,
   discovers the SLO is fine. Pages are for SLO burn.
2. **Runbooks that read like essays.** The first field is a
   command, not a history lesson. Module-level mistakes 2 and 3
   already called this out; still common.
3. **No humane caps.** "We page as much as it takes" is a plan
   to lose engineers. The program must have caps and a policy
   for when they are breached.
4. **No incident commander.** One person debugs, communicates,
   and updates the status page at the same time. SEV1 requires
   IC separation.
5. **Postmortems with blame.** Module-level mistake. The
   template forbids personal blame; a reviewer who finds it
   must send the postmortem back.
6. **Action items with no reverification.** Fixed in the PR
   that resolves the incident; the team learns the fix works
   only by not having the incident again — which is not the
   same as knowing.
7. **Error-budget policy written down but never enforced.** PM
   pressure arrives and the policy evaporates. Senior-tier
   programs make the policy a contract agreed before the SLO
   was adopted.
8. **Status page updated manually only.** Module-level mistake
   #4 — automate the first update, confirm manually.
9. **On-call defined without compensation.** The program is
   unserious to its engineers if the organizational side is
   unresolved.

## 6. References

Primary (official and canonical sources):

- Google SRE book — Chapter 11, Being On-Call —
  <https://sre.google/sre-book/being-on-call/>
- Google SRE book — Chapter 14, Managing Incidents —
  <https://sre.google/sre-book/managing-incidents/>
- Google SRE book — Chapter 15, Postmortem Culture —
  <https://sre.google/sre-book/postmortem-culture/>
- Google SRE Workbook — On-Call —
  <https://sre.google/workbook/on-call/>
- Google SRE Workbook — Incident Response —
  <https://sre.google/workbook/incident-response/>
- Prometheus Alertmanager (severity-based routing) —
  <https://prometheus.io/docs/alerting/latest/alertmanager/>

Governance context:

- NIST AI Risk Management Framework — MANAGE 1.3 (response and
  recovery plans) and MANAGE 2.3 (incident communication) are
  the framework slots an SRE on-call program occupies for AI
  workloads — <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — detection of ML09
  Output Integrity failures runs through this on-call program
  via the SLIs from [exercise-02](../exercise-02/SOLUTION.md) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — detections produced by the observability stack
  feed this on-call program's paging policy —
  <https://atlas.mitre.org/>

Local context:

- Module rationale: `modules/mod-207-observability-sre/SOLUTION.md`
  (Decisions 3, 4, 6 — severity tied to data, blameless
  postmortems, per-team error budgets).
- Stack the program depends on: [exercise-01](../exercise-01/SOLUTION.md).
- SLOs the paging policy fires on: [exercise-02](../exercise-02/SOLUTION.md).
- Chaos cadence that exercises the on-call path:
  [exercise-03](../exercise-03/SOLUTION.md).
- Strategy that integrates on-call with the broader program:
  [exercise-04](../exercise-04/SOLUTION.md).
- Engineer-tier implementation reference: `engineer-solutions/mod-108`
  (per module README cross-reference).
