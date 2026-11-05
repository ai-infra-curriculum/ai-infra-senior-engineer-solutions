# SOLUTION — Lab 01: Code Review Exercise

Reference solution for
[lab-01-code-review-exercise.md](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-learning/blob/main/lessons/mod-210-technical-leadership/exercises/lab-01-code-review-exercise.md).

A senior-engineer code review is *not* a style lint. It is one
of the primary mechanisms through which the quality bar, the
architectural intent, and the security posture of a system are
transmitted across a team. The deliverable for this lab is
therefore a review, not an approval — a concrete set of
comments on a pull request that demonstrates the senior-level
lens described in `modules/mod-210-technical-leadership/SOLUTION.md`
("Tech specs that don't include alternatives", "PR reviews
that nit-pick formatting and miss architecture" — the two
failure modes this lab exists to prevent).

## 1. Solution overview

A good senior review has three layered concerns, in this
order:

1. **Is it correct?** Does the code do what the PR
   description says? Does it have the tests that would
   demonstrate it does? Would it have surfaced the bug it
   claims to fix?
2. **Does it belong here?** Does the change fit the system's
   architecture, or is it an orthogonal addition that will
   need to be reworked when the next change lands? Are the
   abstractions at the right altitude?
3. **Is the next reader going to be fine?** Naming,
   docstrings, failure messages, operational concerns (what
   does an on-caller see at 3am?).

For ML-infrastructure PRs there are three additional
concerns that *do not* show up in generic review guides:

- **Data contract and reproducibility.** A change to a
  preprocessing step, a feature definition, or a tokenizer
  silently changes the model's inputs. Reviews catch the
  missing version bump, dataset fingerprint, or feature-store
  schema update.
- **Model supply-chain integrity.** Image tags, model
  weights, and dataset references must be content-addressed
  (digest/hash), not mutable tags. This is
  OWASP ML Top 10 **ML06 — AI Supply Chain Attacks**
  ([OWASP ML Top 10](https://owasp.org/www-project-machine-learning-security-top-10/)).
- **Threat-model fit.** New endpoints, new input paths, new
  model-serving code: do they widen the attack surface for
  prompt injection, membership inference, or model extraction
  (MITRE ATLAS tactics —
  [MITRE ATLAS](https://atlas.mitre.org/))?

The reference review below works through a realistic
training-pipeline PR and shows the exact comment set a senior
engineer would leave.

## 2. Worked answer or implementation

### 2a. The PR under review

> **Title.** `training: add new PII-scrub step before
> tokenizer`
>
> **Description.** "Adds a regex-based PII scrubber to the
> data-loading path for the LLM fine-tune. Fixes the data
> team's request from last week's incident."
>
> **Diff size.** 320 lines added, 8 deleted, across 4 files.

Diff of the main file (annotated below):

```python
# training/preprocess.py
import re
+import logging
+
+SSN   = re.compile(r"\d{3}-\d{2}-\d{4}")
+EMAIL = re.compile(r"[\w\.-]+@[\w\.-]+")
+PHONE = re.compile(r"\d{3}-\d{3}-\d{4}")
+
+def scrub(text: str) -> str:
+    text = SSN.sub("<SSN>", text)
+    text = EMAIL.sub("<EMAIL>", text)
+    text = PHONE.sub("<PHONE>", text)
+    return text

 def load_examples(path: str) -> list[dict]:
     rows = read_jsonl(path)
-    return [tokenize(r["text"]) for r in rows]
+    return [tokenize(scrub(r["text"])) for r in rows]
```

```dockerfile
# Dockerfile
-FROM python:3.11
+FROM python:3.11.7
 COPY . /app
-RUN pip install -r requirements.txt
+RUN pip install -r requirements.txt && \
+    pip install pii-scrubber==0.3.*
```

```yaml
# training/job.yaml  (unchanged — see comment below)
spec:
  containers:
    - image: ghcr.io/acme/trainer:latest
```

### 2b. The review

The review is grouped by severity. In a real review tool
these are left as inline comments on the relevant lines.

**Blocking — must be resolved before merge:**

1. **`training/preprocess.py:4-7` — the regexes are
   under-specified and will produce silent data corruption.**
   `[\w\.-]+@[\w\.-]+` matches far more than email: the
   NumPy matrix-multiplication operator in code samples
   (`weights@x`, `logits@W.T`), SSH clone URLs
   (`git@github.com:owner/repo.git`), and Docker image
   digests (`trainer@sha256:abc123...`) all satisfy the
   pattern and will be replaced with `<EMAIL>`, silently
   corrupting any code or ops snippet in the corpus.
   Please either (a) use a vetted library (`pii-scrubber`
   is already being added to requirements, but it is not
   used — see §2c below) or (b) anchor the patterns
   (require a TLD-shaped suffix, require whitespace or
   string boundaries on each side) and add unit tests that
   prove the false-positive rate on a held-out corpus
   sample is below whatever threshold the data team
   accepts. There is no "regex PII" baseline that is
   correct without this measurement.
2. **`training/preprocess.py:11` — no dataset fingerprint is
   emitted, so this change is invisible to lineage.** The
   model trained after this merge will consume a *different*
   corpus than the model before it, and we currently have no
   way to tell them apart in the registry. Please log the
   SHA-256 of the scrubbed corpus (or a sampled fingerprint
   for large corpora) and include it in the training-job
   metadata so it lands in MLflow / the model card. This is
   the reproducibility concern in NIST AI RMF
   MANAGE-4.1 — the ability to re-derive results
   ([NIST AI RMF 1.0](https://www.nist.gov/itl/ai-risk-management-framework)).
3. **`Dockerfile:2-4` — `pii-scrubber==0.3.*` is a floating
   minor-version pin and will make this build non-reproducible.**
   Use `==0.3.1` (or whatever the exact version you tested is)
   and rely on Dependabot / Renovate to propose upgrades. The
   image itself should also be pinned by digest, not tag,
   downstream — see the next comment.
4. **`training/job.yaml:3` — `image: ghcr.io/acme/trainer:latest`
   is still a mutable tag.** The
   module's admission policy
   (see `modules/mod-209-security-compliance/SOLUTION.md`
   §2a — "Images pinned by digest") rejects this at admission
   in staging. This PR will work locally and fail in the
   first environment that enforces the policy. Replace with
   `image: ghcr.io/acme/trainer@sha256:...` and let the CI
   job stamp the digest after build.

**Non-blocking but important — should be resolved either in
this PR or a follow-up tracked in the PR description:**

5. **`training/preprocess.py:4` — `pii-scrubber` is
   installed but not imported.** Either remove it from
   `requirements.txt` or wire it in and remove the regexes.
   Shipping a dependency you don't use is a supply-chain
   surface for no benefit.
6. **The PR lacks a test for the scrubber.** A 20-line unit
   test with ~10 representative inputs (true positives,
   true negatives, edge cases like code blocks containing
   fake SSNs) would prevent most of concern (1) from
   regressing. The scrubber is the kind of function that is
   cheap to test and expensive to debug after a bad
   training run.
7. **No changelog entry or model-card update.** If this
   scrubber runs for the next fine-tune, the resulting
   model card needs to record "PII-scrubbed corpus (regex
   v0 / library vN)" under the data-handling section. The
   model card lives at `models/<name>/MODEL_CARD.md` per the
   repo convention.

**Discussion — worth raising, not blocking:**

8. **Scope.** The PR description says "fixes the data team's
   request from last week's incident", but the incident
   write-up (link?) is not referenced. Linking the postmortem
   gives the next reader context on *why* this approach was
   chosen and what's intentionally out of scope (e.g., names,
   addresses, dates — all of which the current regex set
   misses). This is the "why not what" rule from
   `modules/mod-210-technical-leadership/SOLUTION.md` §PR
   reviews.
9. **Architectural fit.** The scrubber is in
   `training/preprocess.py`. If an inference-time prompt
   contains PII, is that the inference layer's problem? If
   the answer is "yes, a different scrubber there" then we
   should factor the scrubber into a shared module
   (`common/pii`) in this PR or the next, to avoid
   divergence. Not blocking this PR; raising it so we don't
   lose the decision.

### 2c. How those comments land — a model

The review above satisfies the rubric that
`modules/mod-210-technical-leadership/SOLUTION.md` sets out
for a senior-engineer PR review:

- **Correctness** — points 1, 2, 6 (behaviour, fingerprint,
  test).
- **Architectural fit** — points 7, 9 (model-card, shared
  module).
- **Performance / operational** — points 3, 4 (reproducible
  build, admission compatibility).
- **Security** — point 4 (supply-chain, ML06), point 1
  (data handling), point 8 (threat context).
- **Documentation** — points 7, 8 (model card, incident
  link).

Each comment names the line, states the concern, explains
the *why*, and points to the standard or repo convention the
reviewer is holding the author to.

### 2d. What a bad review of the same PR looks like

For contrast — the senior-engineer failure mode:

> `preprocess.py:4`: "nit: can we alphabetise the regex
> constants?"
> `preprocess.py:7`: "nit: missing trailing newline."
> Approves.

This review misses every blocking issue, transmits no
engineering standard, and trains the author to expect PR
review to be cosmetic.

## 3. Validation steps

1. Count the review comments. A senior review of a
   ~300-line PR typically has 5-10 substantive comments
   across the categories above. More than ~20 is a signal to
   split into a PR-level comment plus line-level ones; fewer
   than 3 is a signal the review didn't engage with the
   change.
2. For each blocking comment, check it names a line, states
   the concern, and gives the *why*. Comments without a
   *why* are opinions the author will not learn from.
3. Run the comment set against the five-point review
   checklist in §2c (correctness / fit / performance /
   security / documentation). Any category with zero
   comments either means the PR genuinely didn't touch that
   concern or the reviewer missed it; name which.
4. Read the PR description after writing the review. The
   two should line up — the review should not be about
   issues the description promised to ship. If it is, the
   PR should either be split or the description updated.
5. Would an on-caller woken by a failure from this code at
   3am be able to understand it from the logs, metrics, and
   messages? If the answer is "no", the review didn't look
   at it.

## 4. Rubric / review checklist

Use this rubric when grading a submitted PR review (or
self-grading your own):

| # | Check | Weight |
|---|---|---|
| R1 | Review names at least one *correctness* concern (behavioural, boundary, or test-gap) or explicitly notes none applies, with evidence | 15 |
| R2 | Review names at least one *architectural-fit* concern (abstraction altitude, placement, duplication) or notes none applies | 10 |
| R3 | Review includes at least one ML-specific concern from data contract, reproducibility, or supply chain (OWASP ML Top 10) | 15 |
| R4 | Every blocking comment states *why*, not only *what*; the author can learn the standard from the comment alone | 15 |
| R5 | Comments reference specific lines or hunks, not the PR as a whole | 10 |
| R6 | Severity is explicit (blocking / non-blocking / discussion), so the author can prioritise without guessing | 10 |
| R7 | Tone is direct and specific; no personal-preference framing ("I would have done X" without a trade-off) | 10 |
| R8 | Review covers the five canonical dimensions (correctness, tests, fit, performance, security, docs) with a comment or an explicit "n/a — reason" | 15 |

Pass threshold: **75/100**. R3 and R4 are must-haves —
a review without the ML-specific lens or without
*why*-grounded comments does not transmit the senior-level
bar this module exists to teach.

## 5. Common mistakes

1. **Nit storm.** 30 whitespace / alphabetisation comments,
   no architectural comment. Signals the reviewer scanned
   the diff instead of reading it. If the codebase needs
   those nits caught, a formatter / linter should do it,
   not a human.
2. **Approval without review.** Clicking approve because
   CI is green. The human review exists to catch what CI
   cannot — architecture, intent, data-contract drift.
3. **"Needs tests" with no specifics.** A useful comment
   says *what* behaviour needs a test (boundary input,
   failure mode, regression for a specific prior bug), not
   just that tests are missing.
4. **Reviewing the author, not the code.** Comments like
   "you always do X" belong in a 1:1, not a PR. The PR
   review is a public artefact that teaches the whole team.
5. **Blocking on preference.** "I'd use a dataclass here
   instead of a dict" with no trade-off attached. Either
   the PR's choice violates a standard (cite it) or it's a
   style preference (don't block on it).
6. **Missing the ML-specific lens entirely.** Reviewing a
   model-training PR the same way as a CRUD-endpoint PR.
   Data lineage, reproducibility, model supply chain, and
   threat model are first-class in this codebase.
7. **No follow-up tracking.** A non-blocking comment that
   says "we should factor this out eventually" with no
   issue filed just becomes repo folklore. Either file the
   issue in the comment or move the concern to blocking.
8. **Confusing the PR author's skill level with the
   standard.** The standard the review enforces is the
   codebase's, not the author's. Explain, don't lower the
   bar.

## 6. References

Primary (official project documentation and standards):

- OWASP Machine Learning Security Top 10 (ML06 — AI Supply
  Chain Attacks is the direct basis for the image-digest and
  dependency-pinning comments in §2b) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS (adversary techniques against ML systems —
  informs the threat-model lens in §1 and §2a) —
  <https://atlas.mitre.org/>
- NIST AI Risk Management Framework 1.0
  (MANAGE-4.1 reproducibility is the basis for the dataset-
  fingerprint comment) —
  <https://www.nist.gov/itl/ai-risk-management-framework>

Local context:

- Module-level rationale and failure modes:
  `modules/mod-210-technical-leadership/SOLUTION.md`
  ("The PR review checklist (exercise 02)" paragraph
  describes the five canonical review dimensions, and
  "Common mistakes graders see" §2 is the nit-storm failure
  mode this lab exercises against).
- Supply-chain and admission policy context for the
  image-digest comment: `modules/mod-209-security-compliance/SOLUTION.md`
  §2f (`require-image-digests`, `verify-image-signatures`).
- Engineer-tier implementation reference for the underlying
  training-pipeline code:
  `engineer-solutions/mod-106 ex-14` (per module README
  cross-reference).
