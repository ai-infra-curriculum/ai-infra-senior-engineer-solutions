# SOLUTION — Lab 03: Technical Presentation Practice

Reference solution for
[lab-03-presentation-practice.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-210-technical-leadership/exercises/lab-03-presentation-practice.md).

The deliverable for this lab is a short internal
technical talk — 15 to 20 minutes, with slides and speaker
notes — that a senior engineer would give to an audience of
peers plus a sprinkling of cross-functional stakeholders.
The talk teaches a decision or a system change; it does not
sell, and it does not read the architecture doc aloud.

This reference walks through the structure, provides a
worked outline with speaker notes, and gives a rubric for
grading a submitted talk (recorded or delivered live).

## 1. Solution overview

Good senior-engineer technical talks share a three-arc
shape that comes out of the technical-communication
literature and the internal conference-talk guides at major
engineering orgs (Monica Lent — "How to give a good
technical talk"
— <https://monicalent.com/blog/2019/08/27/how-to-give-a-good-technical-talk/>;
GitLab engineering handbook — "Presenting at conferences"
— <https://handbook.gitlab.com/handbook/engineering/development/principles/#writing-things-down>):

1. **Hook** (1-2 min) — a concrete question, incident, or
   number that makes the audience want the rest. Not an
   agenda slide.
2. **Body** (10-14 min) — the content, in three to five
   points, each of which earns its own ~2-3 minutes of
   oxygen.
3. **Takeaway** (2-3 min) — the one thing you want the
   audience to remember, and the next action they can
   take.

For AI/ML platform talks specifically, two additional
constraints apply:

- **Ground claims.** If the talk cites a number ("30%
  faster", "halved GPU hours"), a backing measurement must
  be nameable in a speaker note. Unbacked claims erode the
  presenter's credibility with the technical audience and
  get picked apart in Q&A.
- **Threat / governance lens.** When the talk touches
  model serving, data handling, or deployment surface, it
  should name the relevant standard (NIST AI RMF,
  OWASP ML Top 10, or MITRE ATLAS) rather than
  reinventing the vocabulary.

The worked outline below applies this shape to a realistic
15-minute internal talk: *"How we cut model-serving tail
latency by redesigning the registry fetch path."*

## 2. Worked answer or implementation

### 2a. Target audience and talk goal

> **Audience.** ML platform engineers (primary), ML
> engineers from product teams (secondary), SRE and
> security (one or two), engineering manager (one).
>
> **Slot.** 15 minutes talk + 5 minutes Q&A at the weekly
> platform brown-bag.
>
> **Goal (single sentence).** "After this talk, platform
> engineers will know that the model-registry fetch path
> was the dominant source of P99 serving latency, what the
> fix was, and when to apply the same diagnostic shape to
> their own services."

Writing the goal as a sentence is itself the discipline.
If you can't write it, you don't have a talk yet.

### 2b. Slide-by-slide outline with speaker notes

Target: ~10 slides for a 15-minute slot (≈90 seconds per
slide average; some slides get 30 seconds, some 3 minutes).
Speaker notes are the sentences the presenter *will say*,
not bullet points to be read aloud from the slide.

---

**Slide 1 — Hook (0:00-0:45).**
*On-screen: a single latency chart with a red line and a
green line. No title. No agenda.*

Speaker notes:
> "This is model-A's P99 serving latency over 30 days. The
> red line is where we were at the start of the quarter.
> The green line is where we landed last week. That gap is
> what I'm going to walk you through today — what it was,
> how we found it, and why it probably exists in two more
> of our services."

The hook is a chart with a stake attached. No agenda slide.

---

**Slide 2 — Why this matters to you (0:45-1:30).**
*On-screen: three bullet points phrased as audience
outcomes.*

Speaker notes:
> "Three reasons this is worth 15 minutes of your time.
> First, if you own a KServe-fronted model, this is
> probably happening to you and you don't know. Second,
> the fix is roughly a day of work, not a quarter.
> Third, the diagnostic approach — which is most of the
> talk — generalises beyond serving."

The audience should know, by minute 2, whether they should
keep paying attention.

---

**Slide 3 — The system as it was (1:30-3:30).**
*On-screen: architecture diagram, 5 boxes, 4 arrows. Boxes
labelled, arrows unlabelled.*

Speaker notes:
> "Here is the serving path the way we built it in January.
> KServe inference service pulls from S3 on cold start.
> The S3 bucket is behind the model-registry, which we
> front with MLflow. Metadata lookup, then artefact pull.
> Normally sub-second. Sometimes 8 seconds. That variance
> is what we're going to chase."

Two minutes on the picture. Audience must leave this slide
able to redraw the diagram on a napkin.

---

**Slide 4 — The symptom and the trap (3:30-5:30).**
*On-screen: a histogram of fetch latency with a long tail,
and the pod's startup-probe timeout line overlaid.*

Speaker notes:
> "Here's the trap. The mean fetch was 400ms. Everybody
> looked at the mean, nobody looked at the P99. The P99
> was 6 seconds. Our startup probe timeout was 10. We
> were one bad week away from a flapping serving tier,
> and we didn't see it until a cold-start storm after a
> cluster upgrade actually tripped it."

Call out the diagnostic error explicitly. This is the
teaching moment, not the number.

---

**Slide 5 — How we localised it (5:30-7:30).**
*On-screen: a trace flamegraph with the metadata-lookup
span highlighted.*

Speaker notes:
> "We added OpenTelemetry spans around the three phases of
> the fetch — metadata lookup, auth, artefact download.
> This is one representative trace from the P99 bucket.
> 5.4 seconds of the 6 were in metadata lookup. The
> download was fine. The surprise wasn't *that* metadata
> was slow — it was that metadata was the whole bill."

Name the tool (OpenTelemetry) and show the evidence.
OpenTelemetry — <https://opentelemetry.io/>.

---

**Slide 6 — Root cause, in one sentence (7:30-9:00).**
*On-screen: the sentence, large font, nothing else.*

> "MLflow's REST client does a sync DNS + TLS handshake
> per fetch, and our cold-start storm hit a DNS cache miss
> in the pod's resolver."

Speaker notes:
> "That's it. Nothing exotic. The variance was dominated
> by DNS misses on first contact. Once we knew it was DNS,
> everything else fell into place. The lesson here —
> which is bigger than this talk — is that 'the registry
> is slow' was a convenient story that cost us two weeks
> before we instrumented the inside of the fetch."

One-sentence slides are permitted and powerful when the
sentence is the point.

---

**Slide 7 — The fix (9:00-11:00).**
*On-screen: a before/after code snippet (10 lines each
side) and a bullet list of three changes.*

Speaker notes:
> "Three changes. One: swapped the per-call MLflow REST
> client for a long-lived `requests.Session` with HTTP
> keep-alive, so DNS resolution and TLS negotiation happen
> once per pod lifetime instead of once per fetch. Two:
> warmed the resolver cache in the serving pod's init
> container with a single pre-flight call to the registry,
> so even the first real fetch skips the cold DNS miss.
> Three: added a readiness-probe-based cold-start health
> check so any pod that exceeds 2 seconds on first fetch
> fails fast instead of serving timeouts. The connection
> reuse did most of the work, but the warm-up is what
> tightened the tail."

Three changes maximum per slide. If the real fix had seven,
split across two slides and name the subset.

---

**Slide 8 — The numbers (11:00-12:30).**
*On-screen: the same latency chart from slide 1 with
annotation of which change landed when.*

Speaker notes:
> "P50 went from 400ms to 310ms. P99 went from 6s to
> 800ms. Cold-start storm last Tuesday — the one that
> would have knocked us over in January — the fleet
> stayed under 1-second P99 across the storm. The cost of
> the change is one extra second on pod startup, which
> for a model-serving pod with a 30-second load phase is
> noise."

Backing measurement, not vibes. Every number on this
slide has a dashboard behind it.

---

**Slide 9 — When to apply the same shape (12:30-13:30).**
*On-screen: a checklist of three triggers.*

Speaker notes:
> "If you own a service that (one) fetches from a remote
> registry on cold start, (two) has P99 more than 5x its
> P50, and (three) has a startup probe within 2x of its
> worst observed cold start — you have this bug. The
> diagnostic playbook — OpenTelemetry spans around the
> fetch, warm the resolver, measure the tail not the mean
> — is what to reach for before you blame the registry."

This is the generalisation slide — the reason the audience
is sitting there instead of reading the postmortem.

---

**Slide 10 — Takeaway and ask (13:30-15:00).**
*On-screen: one takeaway sentence and one call-to-action
link.*

> "Mean hides tail. Instrument before you blame.
> Playbook at `platform/runbooks/cold-start.md`."

Speaker notes:
> "If you take one thing: mean hides tail, especially for
> anything with a cache. The runbook link has the exact
> OpenTelemetry snippet and the resolver warmup container.
> I'm around for Q&A, and SRE's on-call is in
> #platform-help if you want to walk through your own
> service."

The talk's goal from §2a — "when to apply the same
diagnostic shape to their own services" — lands on this
slide. The call to action is a link, not a ticket.

---

### 2c. Q&A preparation (the invisible fourth arc)

The Q&A is part of the talk. Prepare three questions you
expect and the one-sentence answer to each. For this talk
those are:

- "Why not just raise the probe timeout?" —
  *"That hides the regression. The symptom disappears; the
  tail stays long. A future upgrade with a worse cold
  start tips us over again."*
- "Does this generalise to feature-store fetches?" —
  *"Probably. The diagnostic — span the fetch, measure
  the tail — is the same. The fix may differ."*
- "What did we not fix?" —
  *"The registry API itself is still a per-call REST
  surface. For now the pooled session absorbs the overhead.
  If registry latency doubles, we'll have to redesign the
  registry, not the client."*

The "what did we not fix" answer is the senior-engineer
tell. Admitting scope honestly builds credibility faster
than victory-lap claims.

### 2d. Delivery checklist

The night before the talk, run through these:

1. **Read the opening sentence out loud, from memory.** If
   you cannot say the hook without the slide, rewrite it
   shorter.
2. **Time the talk once.** Aim for 15 min on a 15-min slot
   with a 2-minute buffer for slower delivery when
   nervous.
3. **Check the chart screenshots render.** On the actual
   projector / Zoom at the actual resolution. Half of
   talks ship because the bar chart is unreadable from the
   back of the room.
4. **Put a backup URL for the key dashboard in the
   speaker notes.** If someone in Q&A wants to see the
   real data live, you want the link one tab away.
5. **Rehearse the ask slide out loud.** The last 30
   seconds are what people remember.

## 3. Validation steps

1. Count the slides. For a 15-minute talk, 8-12 slides is
   the sweet spot; above 20 you are projecting, not
   presenting.
2. Read slide 1 as if you were the audience. Is there a
   question or stake? If it is an agenda slide, rewrite.
3. For each body slide, can you state the single sentence
   it is trying to land? If not, cut the slide or split
   it.
4. For each quantitative claim on a slide, can you name
   the dashboard or query it came from in the speaker
   notes? If not, remove the number or back it.
5. Read the last slide. Is there one takeaway and one
   action? If the last slide is a Thanks + Questions
   bullet list, the talk did not land a message.
6. Do a timed dry run. If the talk runs more than 10%
   over slot, cut from the body, not from the hook or
   takeaway.
7. Have one peer read the slides without you. Can they
   explain the point back? If not, the slides are
   presenter-only; rewrite the ones that failed the test.

## 4. Rubric / review checklist

Use this rubric to grade a recorded or live submission:

| # | Check | Weight |
|---|---|---|
| R1 | Talk has a single, writable goal sentence ("After this talk, the audience will ...") that the presenter can state | 10 |
| R2 | Opens with a hook — a chart, a question, an incident — not with an agenda slide or self-introduction | 10 |
| R3 | Body is structured as 3-5 points, each with its own slide and ~2-3 minutes of airtime; no slide tries to carry more than one idea | 15 |
| R4 | Every quantitative claim ("30% faster", "P99 cut in half") has a backing measurement nameable by the presenter in speaker notes or Q&A | 15 |
| R5 | Technical content is accurate to the underlying system; slides refer to tools and standards by their correct names (OpenTelemetry, NIST AI RMF, OWASP ML Top 10, etc. as applicable) | 10 |
| R6 | Talk closes with one memorable takeaway and one concrete call-to-action (link, playbook, next step) | 10 |
| R7 | Speaker notes contain sentences the presenter will *say*, not bullet points read from the slide | 10 |
| R8 | Delivery is within 10% of the time slot and the presenter can state three anticipated Q&A questions with one-sentence answers | 10 |
| R9 | Visuals are readable at projector / Zoom resolution; no illegible screenshots or 8-point font tables | 5 |
| R10 | The talk teaches a generalisable lesson, not only "we did this cool thing"; the audience leaves with something they can apply | 5 |

Pass threshold: **75/100**. R1 and R3 are must-haves — a
talk without a goal sentence or without a structured body
is a monologue, not a teaching artefact.

## 5. Common mistakes

1. **Agenda slide as the opening.** The audience already
   has the title from the invite. Open with the hook.
2. **Reading slides aloud.** Slides are visual aids; the
   presenter is the content. If every slide's bullets
   match the speaker notes word-for-word, remove either
   the slide or the notes.
3. **No single takeaway.** The audience leaves with a
   collage of impressions, not a sentence. Most talks that
   under-perform fail this specifically.
4. **Numbers without backing.** "We improved throughput
   by 40%" with no graph and no benchmark named. In a
   senior-engineer audience this is read as "I don't
   know but the number sounds good."
5. **Architecture slide as the whole middle.** Twelve
   minutes describing the system means two minutes
   describing *what you did and what you learned*. Flip
   the ratio.
6. **Too many slides.** 40 slides in 15 minutes = 22
   seconds per slide. You are a human flipbook. 8-12
   slides, each earning its time.
7. **No Q&A prep.** The audience's hardest question is
   the one that reveals what the talk skipped. Anticipate
   and answer it on-slide, or flag it explicitly as
   out-of-scope with a reason.
8. **"Thanks — Questions?" as the last slide.** Waste of
   the most-remembered real estate in the talk. Put the
   takeaway and the link there; take questions from the
   takeaway.
9. **Jargon without grounding.** Dropping "MLOps" or
   "inference mesh" without defining the specific
   component you mean. Use the canonical standard names
   (KServe, MLflow Model Registry, NIST AI RMF
   subcategory numbers) where you can.
10. **Victory-lap tone.** No mention of what didn't work,
    what remains open, or what the next decision is.
    Technical audiences discount advocacy; they trust
    honesty about scope.

## 6. References

Primary (technical-communication references and standards
the content leans on):

- Monica Lent — "How to give a good technical talk" —
  <https://monicalent.com/blog/2019/08/27/how-to-give-a-good-technical-talk/>
- GitLab Engineering Handbook — "Writing things down" and
  presentation guidance —
  <https://handbook.gitlab.com/handbook/engineering/development/principles/>
- OpenTelemetry (the instrumentation the worked talk
  depends on for its diagnostic) —
  <https://opentelemetry.io/>
- NIST AI Risk Management Framework 1.0 (reference when a
  talk touches governance, data handling, or model risk) —
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 (reference when
  the talk touches serving-surface risks) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS (reference when the talk discusses ML
  adversary techniques) —
  <https://atlas.mitre.org/>

Local context:

- Module-level framing on how senior-engineer
  communication differs from management communication:
  `modules/mod-210-technical-leadership/SOLUTION.md`
  §"What this module is really teaching" — the talk
  described here is the "tech-lead-of-project" artefact
  of the same voice.
- Observability / tracing context for the worked talk's
  diagnostic approach:
  `modules/mod-207-observability-sre/SOLUTION.md`.
- Model-registry context referenced in the worked talk:
  `modules/mod-210-technical-leadership/exercise-02/SOLUTION.md`
  §2a (the ADR that put MLflow Model Registry in the
  path).
