# SOLUTION — Lab 05: Influence Without Authority

Reference solution for
[lab-05-influence-without-authority.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-210-technical-leadership/exercises/lab-05-influence-without-authority.md).

Senior engineers consistently deliver work that spans teams
they do not manage. The leadership currency is *influence*:
getting other teams to adopt a change, prioritise a
dependency, or stop doing something, without a reporting
line to lean on. The deliverable for this lab is a written
**influence plan** for a real scenario, followed by three
worked scenarios that show the plan applied.

The senior-engineer altitude matters here. Staff/principal
leadership operates at the org level; this lab exercises
the next-door-team / adjacent-stakeholder altitude —
smaller blast radius, concrete artefacts, verifiable
outcomes. The lens is the one the module's own SOLUTION.md
sets out in §"Small-blast-radius decisions": decisions that
affect 5-10 engineers and one or two adjacent teams.

## 1. Solution overview

The reference approach pulls from two literatures:

- **Technical-leadership writing** on cross-team work
  (Will Larson — *Staff Engineer* and
  [StaffEng.com](https://staffeng.com/),
  Tanya Reilly — *The Staff Engineer's Path*,
  [LeadDev](https://leaddev.com/) library), where the
  core move is *shared artefacts over persuasion*.
- **Negotiation and interests-based conflict** (Fisher
  & Ury — *Getting to Yes*,
  [Harvard Program on Negotiation](https://www.pon.harvard.edu/)),
  where the core move is *interests behind positions*.

The influence plan is five steps:

1. **State the outcome.** One sentence. The success
   condition, not the activity.
2. **Map stakeholders.** For each person or team whose
   action you need: what are their *interests* (not
   positions), what do they *know*, what do they *trust*,
   what's their *cost* to say yes?
3. **Build the shared artefact.** A doc, an ADR, a
   benchmark, a prototype — the thing the conversation
   anchors on. Without it, influence collapses into
   politics.
4. **Sequence the conversations.** Not everyone meets at
   the same time. Who's first, who's last, and what does
   each conversation add to the artefact?
5. **Name the fallback.** If the influence plan fails,
   what happens — escalation, scope cut, or carrying the
   cost yourself?

Applied to AI/ML platform work there's a sixth
discipline: **ground the ask in standards**. "We should
sign model artefacts" is a position. "NIST AI RMF
MANAGE-4.1 requires reproducibility and OWASP ML06
addresses supply-chain tampering; our model-artefact path
closes both; here's a 2-hour prototype" is an argument the
next team can engage with. The frameworks (NIST AI RMF,
OWASP ML Top 10, MITRE ATLAS) are not only governance
anchors — they are the shared vocabulary that lets
influence travel across teams.

## 2. Worked answer or implementation

### 2a. The influence-plan template (copy-paste)

```markdown
# Influence Plan — <short title>

## Outcome
One sentence. "By <date>, team X has <done Y>."

## Why now
What forces (incident, standard, upcoming launch) make
this the right time to act?

## Stakeholders
- <team / person> — Interest: <what they care about>.
  Current position: <what they say today>.
  Trust bridge: <shared history, shared person, shared
  pain>. Cost of yes: <engineering days, risk, scope cut>.
- ...

## Shared artefact
The doc / ADR / benchmark / prototype that the
conversation will anchor on. Status: <drafting / shared /
reviewed>.

## Conversation sequence
1. <person / meeting> — purpose: <align on problem,
   collect requirements, pre-wire decision>.
2. ...

## Fallback
If this does not land by <date>: <escalate to X / cut
scope to Y / carry the cost in platform>.
```

The plan is a **working doc you keep updating**. Senior
engineers who are good at this consistently keep the plan
visible and short — one page — and iterate it after each
conversation.

### 2b. Worked scenario 1 — adoption ask across teams

> **Context.** The platform has rolled out signed model
> artefacts (control C02 in
> `modules/mod-210-technical-leadership/exercise-04/SOLUTION.md`).
> Admission policy enforces the signature in Production
> but is permissive in Staging for migration. Four
> product teams need to flip their CI to sign. One team
> has resisted for two months.

**Influence plan.**

> **Outcome.** By 2026-03-15, team-ranker has adopted the
> signing step in CI and removed its staging exemption.
>
> **Why now.** Staging-wide exemption expiry is
> 2026-03-20; after that, admission will reject their
> next promotion.
>
> **Stakeholders.**
>
> - *Team-ranker lead (M. Chen).* Interest: shipping the
>   personalisation rerank by Q2. Current position: "the
>   signing step adds 4 minutes to CI and we're already
>   over budget on CI time." Trust bridge: we both came
>   up through the same mentor on the data team; we have
>   shipped a cross-team project before. Cost of yes:
>   ~1 engineer-day to wire cosign-sign + sidecar cache
>   for the keys.
> - *Team-ranker TL (D. Singh).* Interest: not being the
>   last team holding the migration open. Current
>   position: deferring to lead. Cost of yes: zero
>   direct, political cost if the lead pushes back.
> - *Security lead (R. Patel).* Interest: closing the
>   signing-gap metric before the next audit. Already
>   an ally; unchanged by this plan.
>
> **Shared artefact.** A one-page document comparing
> (a) their current CI timings, (b) the 20-second
> addition the signing step actually adds with the
> sidecar cache (not 4 minutes), (c) the admission
> rejection they would otherwise hit on 2026-03-20.
> Drafted and shared before the first conversation.
> Grounds the ask in OWASP ML06 and NIST AI RMF
> MANAGE-4.1 (two-sentence appendix, not a lecture).
>
> **Conversation sequence.**
>
> 1. *Pre-wire with D. Singh (TL).* Purpose: confirm the
>    TL will not block if the lead signs off; surface
>    any concern the lead hasn't voiced. Quick 1:1.
> 2. *Working session with M. Chen (lead).* Purpose:
>    walk through the one-pager. Offer to pair on the
>    CI change for an hour. Do not ask for a decision
>    in the meeting; ask what would need to be true for
>    yes.
> 3. *Platform implementation PR.* Offer a draft PR
>    against team-ranker's CI config implementing the
>    sidecar cache change. The ask drops from "change
>    your CI" to "review my PR".
> 4. *Security update at next cross-team sync.* Report
>    coverage going from 3/4 to 4/4 as a joint result,
>    not a platform-team result.
>
> **Fallback.** If team-ranker does not adopt by
> 2026-03-10, platform tech-lead escalates to the
> engineering director with the audit-gap framing (not
> the "team didn't cooperate" framing). Escalation
> material is the same one-page doc with a timeline
> appendix. If escalation produces no action by
> 2026-03-18, platform cuts scope: team-ranker's
> promotions from Staging to Prod freeze; models
> already in Prod continue; the risk register picks up
> the gap under control C02.

**What the plan models.**

- *Interest, not position.* The position is "4 minutes
  is too much". The interest is "we're on a deadline and
  I don't have cycles for platform migration." The
  pre-wire and the PR offer address the interest, not
  the position.
- *Shared artefact.* The one-pager with real numbers
  moves the conversation from "we think" to "we
  measured." The appendix cites NIST and OWASP without
  belabouring them.
- *Sequenced conversations.* Pre-wire first; working
  session second; artefact (the PR) third. No team-wide
  Slack call-outs; those are escalation, not influence.
- *Fallback is named.* Escalation path and scope-cut are
  both in the plan, so the influence attempt is not an
  all-or-nothing bet. The fallback is the thing that
  lets you be patient *with the conversation*.

### 2c. Worked scenario 2 — pushing back on a bad plan

> **Context.** A principal engineer in a sister org has
> proposed moving the inference tier to a new serving
> stack ("because KServe is too complicated"). The
> senior engineer here owns the current serving tier and
> believes the proposal is under-cooked. Direct
> contradiction in the review thread has already
> hardened positions.

**Influence plan.**

> **Outcome.** By end of the next architecture review,
> the proposal is either (a) withdrawn pending data, or
> (b) refined with the data that would make it a sound
> decision — but in either case not adopted on vibes.
>
> **Why now.** The review meeting is in ten days; once
> the proposal clears review, re-opening the decision is
> much more expensive.
>
> **Stakeholders.**
>
> - *Principal engineer (proposer).* Interest: solve a
>   real reliability pain they experienced last quarter
>   (two incidents in KServe). Position: "swap stacks".
>   Trust bridge: senior-eng has delivered against
>   principal's prior asks; relationship is intact.
>   Cost of yes to *refining* the proposal: ~1 week.
> - *Serving team manager.* Interest: operational
>   stability; is nervous about a stack change without
>   data. Already aligned with the senior engineer;
>   not a conversation needed, but useful as a backchannel
>   sanity check.
> - *VP of platform.* Interest: not approving stack
>   changes that get rolled back in six months. Values
>   written artefacts; low tolerance for "we just think
>   X."
>
> **Shared artefact.** A document that *does not*
> argue against the proposal. Instead, it enumerates
> the specific data the decision requires — incident
> taxonomy for the KServe pain points, benchmark
> matrix for the two stacks, migration cost estimate —
> and offers to collect it jointly. Frames both options
> as still on the table. Grounds in NIST AI RMF
> GOVERN-1.1 ("decisions have accountable processes").
>
> **Conversation sequence.**
>
> 1. *1:1 with the principal engineer, in private.*
>    Purpose: acknowledge the pain, admit it is real;
>    present the data document, not as a counter but as
>    "what we both need before deciding"; ask them to
>    co-own it. Do **not** litigate the proposal in the
>    review thread; it hardens.
> 2. *Review-thread reply (after the 1:1).* Post the
>    joint data plan with the principal engineer as
>    co-author. The review audience sees collaboration,
>    not opposition.
> 3. *Architecture review meeting.* Present the data
>    plan. Ask the committee to defer the decision one
>    cycle pending the data. The principal engineer has
>    already agreed to this ask, so the meeting is a
>    formality.
> 4. *Follow-up.* Whichever direction the data points,
>    senior-eng co-owns the resulting ADR with the
>    principal.
>
> **Fallback.** If the principal engineer declines the
> joint data-gathering, the senior engineer brings a
> written dissent to the review (not a Slack argument),
> grounded in the data gaps the proposal does not
> address, and names the specific risks under
> NIST AI RMF MANAGE-1.1 (risk prioritisation) that go
> unaddressed. The written dissent is the fallback; the
> *intent* is a joint plan.

**What the plan models.**

- *Disagree in private, align in public.* The 1:1 is
  the mechanism. The review thread is the publication.
- *Argue about data, not plans.* Moving the conversation
  from "your plan is bad" to "we both need data X" is
  the single most reliable influence pattern in
  technical disagreements.
- *Fallback is written.* A written dissent is a
  high-cost, high-signal fallback. It preserves the
  relationship better than a verbal fight because it
  can be engaged with, revised, or withdrawn.

### 2d. Worked scenario 3 — asking an adjacent team to stop

> **Context.** The data-science org has been running
> ad-hoc inference jobs on the production training
> cluster (GPUs) when the training cluster has
> headroom. The practice saves data-science real money,
> but it has caused two Sev2 incidents where a long
> inference job starved a queued training run. The
> senior engineer on the training platform needs the
> practice to stop.

**Influence plan.**

> **Outcome.** By 2026-04-01, ad-hoc inference jobs on
> the training cluster are blocked at admission, and
> data-science has a documented alternative path.
>
> **Why now.** Second Sev2 was last week; the postmortem
> action item is this change; the training platform's
> SLO is at risk if it recurs.
>
> **Stakeholders.**
>
> - *Data-science staff engineer.* Interest: cheap
>   inference; afraid "stop" means paying for a new
>   cluster. Position: "it's been working fine until
>   last week."
> - *Data-science manager.* Interest: not being on the
>   hook for the next incident. Already quietly
>   agreeing with the training platform; useful ally.
> - *Finance partner.* Interest: GPU-hour accountability.
>   Has previously shipped the training-cluster
>   chargeback model; an ally on anything that improves
>   cost attribution.
>
> **Shared artefact.** A short *transition plan*, not
> a *stop order*:
>
> - the admission rule that will block inference-shaped
>   jobs (effective 2026-04-01);
> - the alternative path — a lower-tier inference pool
>   with spot GPUs, lower cost than training-cluster
>   opportunistic use, documented in a one-page
>   runbook;
> - the two Sev2 incidents cited directly, with
>   postmortem links.
>
> Ground the ask in NIST AI RMF MANAGE-4.3 (response
> to incidents) and the platform's control C07.
>
> **Conversation sequence.**
>
> 1. *Pre-wire with data-science manager.* Confirm
>    they support the direction; collect objections
>    their staff engineer is likely to raise.
> 2. *Working session with the staff engineer.* Lead
>    with the alternative path, not the block. Ask
>    what would need to be true for the alternative to
>    meet their needs; iterate.
> 3. *Joint Slack post.* Co-authored by data-science and
>    training platform, announcing both the block and
>    the alternative. Co-authorship is the signal that
>    this is a change, not a conflict.
> 4. *Admission rule ships behind a feature flag*
>    with a one-week dry-run; data-science leads
>    verify the alternative in dry-run before enforce.
>
> **Fallback.** If data-science cannot migrate by the
> deadline, the training platform ships the admission
> rule in warn-only mode and extends by one month,
> exactly once, logged in the exceptions register per
> §2d of `exercise-04/SOLUTION.md`. The exception
> expires regardless of migration state; after that,
> enforce.

**What the plan models.**

- *Give them a way to say yes.* "Stop doing X" without
  an alternative is a position, not a plan. The
  alternative inference pool is the thing that makes
  the yes cheap.
- *Co-authorship as signal.* The joint post, not a
  unilateral platform-team post, is how the change
  reads as an improvement rather than an edict.
- *Exceptions register is the fallback, not the
  default.* Using the governance exception process as
  a *bounded* fallback is a strong senior move — it
  concedes gracefully without abandoning the control.

### 2e. The four influence moves these scenarios share

Across the three scenarios, the same four moves appear:

1. **Interests not positions.** Named for each
   stakeholder; addressed concretely in the plan.
2. **Shared artefact.** One or two pages, grounded in
   data (and in NIST / OWASP where applicable),
   co-authored where possible.
3. **Sequenced conversations.** 1:1 before group,
   alignment before announcement, data before vote.
4. **Named fallback.** Escalation, dissent, or
   scope-cut written down in advance, so the influence
   attempt can afford patience.

Senior engineers who are strong at influence are rarely
better speakers or more charismatic than their peers.
They are more consistent about these four moves.

## 3. Validation steps

1. **Outcome test.** Can the plan's outcome be
   verified by an observer? "Team-ranker has adopted
   signing in CI" is verifiable; "better relationship
   with team-ranker" is not. Rewrite anything that
   fails the observer test.
2. **Stakeholder-interest test.** For each named
   stakeholder, the plan names an *interest* (what they
   care about) and not only a *position* (what they
   say). Positions without interests are bumper stickers.
3. **Artefact test.** The plan names a specific
   artefact (doc, ADR, prototype, PR) and its status.
   "We'll align in a meeting" is not an artefact.
4. **Sequence test.** The plan names at least two
   conversations in order, and at least one is a 1:1
   or small-group pre-wire before any public forum.
5. **Fallback test.** The plan names what happens if
   the ask fails, who authorises the fallback, and when
   the fallback triggers. A plan without a fallback is
   a hope.
6. **Standards-anchor test (ML-platform scenarios).**
   The shared artefact grounds the ask in at least one
   relevant NIST AI RMF subcategory, OWASP ML risk, or
   MITRE ATLAS technique, where applicable. Avoid
   dropping standard names where they don't apply.
7. **Read-back test.** Give the plan (not the
   conversation) to a peer. Can they predict how the
   ask will land? If not, the plan is probably
   missing a stakeholder or an interest.

## 4. Rubric / review checklist

| # | Check | Weight |
|---|---|---|
| R1 | Outcome is a single, observer-verifiable sentence with a date | 10 |
| R2 | For each stakeholder, the plan names an *interest* distinct from *position*; at least one stakeholder-of-concern per plan | 15 |
| R3 | Plan names a concrete shared artefact (doc, ADR, prototype, PR) with status, not "we'll align in a meeting" | 15 |
| R4 | Conversation sequence has at least two steps and at least one pre-wire / 1:1 ahead of any public forum | 15 |
| R5 | Fallback is written down: what triggers it, who authorises it, what it does | 15 |
| R6 | ML-platform plans anchor the shared artefact in at least one applicable standard (NIST AI RMF / OWASP ML / ATLAS), without cargo-culting | 10 |
| R7 | Plan avoids personalising disagreement — no stakeholder entry is framed as "person X is being unreasonable" | 10 |
| R8 | Plan is one to two pages and is a working doc (versioned, iterated after conversations), not a one-off deliverable | 5 |
| R9 | Plan does not rely on authority the senior engineer does not have; escalations are named explicitly as fallbacks, not primary moves | 5 |

Pass threshold: **75/100**. R2, R4, and R5 are must-haves
— without interests, without sequencing, or without a
fallback, the plan is wishful thinking.

## 5. Common mistakes

1. **Position stated as interest.** "Team X is against
   us" is a position. Interests sit underneath: deadline,
   cost, reputation, prior bad experience. If the plan
   doesn't name the interest, the influence will miss.
2. **No shared artefact.** The plan is a schedule of
   meetings. Meetings without an artefact are where
   decisions quietly fail to happen.
3. **One big group meeting as the influence move.**
   Public forums are where decisions are *ratified*, not
   where they are *made*. Use them to confirm, not to
   convince.
4. **Fallback is "escalate".** Escalation is a tool, not
   a plan. If "escalate" is the only fallback, the senior
   engineer is outsourcing the decision.
5. **No standards anchor on ML-platform work.** "We
   should sign model artefacts" with no NIST / OWASP
   reference is harder to carry across teams. Grounded
   asks travel further.
6. **Personalising disagreement.** Writing "person X is
   obstructive" into a plan. Even in private, it
   narrows your thinking to the person, not the
   interest. Rewrite in terms of the interest.
7. **Too polished an artefact.** A 30-page design doc
   signals "I've decided; please agree." A one-page
   data memo with real numbers signals "let's decide
   together." One-pagers win.
8. **Influence used instead of authority you do have.**
   If you *own* a decision (platform tech-lead on the
   control catalogue, for instance), decide it. Running
   an influence campaign over something you can decide
   signals indecisiveness and burns capital.
9. **Patience without a fallback.** Patience is a
   superpower in influence work, but only if a fallback
   is already written. Patience without a fallback is
   just avoidance.
10. **Victory-lap write-ups.** Narrating influence wins
    in public post-hoc ("I got team X to adopt Y")
    makes the next influence attempt harder. Share
    credit; publish the artefact, not the manoeuvre.

## 6. References

Primary (technical-leadership literature and standards):

- Will Larson — *Staff Engineer: Leadership Beyond the
  Management Track* / StaffEng project (cross-team
  influence patterns, "shared artefact" framing) —
  <https://staffeng.com/>
- Tanya Reilly — *The Staff Engineer's Path* (chapters on
  cross-team work are the closest literature match for
  this lab's altitude) —
  <https://www.oreilly.com/library/view/the-staff-engineers/9781098118723/>
- Fisher & Ury — *Getting to Yes* / Harvard Program on
  Negotiation (interests-not-positions, principled
  negotiation) —
  <https://www.pon.harvard.edu/>
- LeadDev — library of senior-engineering talks and
  write-ups on cross-team work —
  <https://leaddev.com/>
- NIST AI Risk Management Framework 1.0 (GOVERN-1.1,
  MANAGE-1.1, MANAGE-4.3 cited in scenarios) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 (ML06 cited in
  scenario 1) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS (for ML-adversary grounding when the
  influence ask touches threat surface) —
  <https://atlas.mitre.org/>

Local context:

- Module-level framing on senior-engineer altitude for
  cross-team work:
  `modules/mod-210-technical-leadership/SOLUTION.md`
  §"Small-blast-radius decisions" and §"Cross-team
  coordination".
- Control catalogue referenced in scenarios 1 and 3
  (signed artefacts, exceptions register):
  `modules/mod-210-technical-leadership/exercise-04/SOLUTION.md`
  §2c and §2d.
- ADR shape used as the "shared artefact" in scenario 2:
  `modules/mod-210-technical-leadership/exercise-02/SOLUTION.md`.
- Engineer-tier implementation references:
  `engineer-solutions/mod-106 ex-14` +
  `mlops-learning/projects/project-4-governance`
  (per module README cross-reference).
