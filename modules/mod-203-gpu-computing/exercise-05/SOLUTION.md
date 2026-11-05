# SOLUTION — Lab 05: GPU Fleet Strategy at Senior Scale

> This is a strategy exercise. The deliverable is a defensible
> multi-year plan, not a code artifact. The grader is evaluating
> trade-off reasoning, cost arithmetic, and the honesty of the
> assumptions behind the plan.

## 1. Solution overview

A senior-tier GPU fleet strategy answers five questions explicitly:

1. **What workloads does the fleet support?** Training, serving,
   research, each with its own utilization and SLO profile.
2. **What generations of GPU do we run, and for how long?**
   Rarely a single generation. Usually "current for training,
   previous for serving, oldest for research/sandbox".
3. **What is the sourcing mix?** Owned (reserved), on-prem /
   co-located, cloud reserved, cloud on-demand, cloud spot /
   preemptible. Each has different cost and reliability curves.
4. **What is the cost model, and what does it predict next year?**
   Cost per training token, cost per served request, idle cost.
5. **How does the fleet fail safely?** Capacity to handle a
   failure of the largest sourcing tier for the SLO commitment.

The reference answer below walks through a single defensible
posture for a mid-sized organization; the rubric scores how well a
student's answer defends their *own* posture, not whether it
matches this one.

## 2. Worked answer or implementation

### 2.1 Workload segmentation

| Segment | Load shape | Right sourcing |
|---|---|---|
| Baseline training | Continuous; large models; needs top-tier GPUs | Reserved / owned |
| Experimental training | Bursty; variable scale | Reserved + cloud on-demand for peaks |
| Production serving | SLO-bound; predictable daily curve; tolerant to right-sizing | Reserved + autoscaled cloud |
| Batch inference | Flexible timing; large volume | Spot / preemptible |
| Research sandbox | Unpredictable; single-user; cheap-per-hour is the main goal | Older generations, previous-gen hardware, or cloud spot |
| CI / smoke tests | Short, parallel, isolated | Time-sliced or MIG on serving nodes |

Segmenting matters because *every* subsequent decision depends on
which bucket a workload lands in. A design that uses one sourcing
posture for everything is leaving money on the table on the low end
and risking SLO on the high end.

### 2.2 Multi-generation fleet

A one-generation fleet is a procurement mistake. The target steady-
state is:

- **Current generation** for all production training. Short depreciation
  horizon; expect full utilization at all times.
- **Previous generation** for the bulk of production serving and for
  experimental training. The hardware is still fast enough for serving
  and research but no longer the most efficient per-FLOP for training.
- **N-2 generation** for sandbox / CI / low-priority batch.

Rotation policy:

- Current-gen rotates in on roughly an 18–24 month cadence
  <!-- author-note: the specific cadence depends on vendor release rhythm and should be set from the organization's own history; do not quote a universal figure -->.
- Previous-gen does not get retired until the N-2 workloads shrink
  below its remaining depreciation.
- "Retired" means removed from production queues; it may still run
  in developer workstations or external donations.

### 2.3 Sourcing mix and the cost model

Three sourcing tiers with a worked cost arithmetic:

1. **Owned / reserved** (lowest unit cost, highest commitment):
   - Pros: no price variance; capacity available during cloud
     scarcity; data gravity stays local.
   - Cons: capital cost up-front; power and cooling are your
     problem; takes months to re-size.
   - Target utilization: ≥80%. Below that, the owned tier is more
     expensive than cloud.
2. **Cloud reserved** (middle unit cost, multi-year commitment):
   - Pros: scales up without buying hardware.
   - Cons: locks you into a vendor and a generation; difficult to
     exit mid-term.
   - Target utilization: 70-90%.
3. **Cloud on-demand / spot** (highest unit cost, zero commitment):
   - Pros: no commitment; useful for burst and experiments.
   - Cons: variable cost; capacity is not guaranteed; preemption
     risk for spot.
   - Use for: peaks, burst training runs, batch inference.

Cost accounting discipline:

- **Cost per training token** (or cost per step) should be tracked as
  a first-class metric per model, per generation. Not "GPU hours".
- **Idle cost**: hours of reserved capacity with no running workload.
  Any week above 20% idle is a signal that the reservation is
  mis-sized
  <!-- author-note: the 20% threshold is heuristic; final write-up should state the organization's chosen SLA-adjusted threshold with its reasoning -->.
- **Cost per served request** by workload class. Autoscaling that
  reduces replica count during off-peak is table stakes.

### 2.4 Capacity planning

Capacity is sized for three distinct numbers:

- **P50 demand**: the normal week. Satisfied by reserved capacity.
- **P95 demand**: peak weeks. Satisfied by reserved + cloud
  reserved. If spot can cover any of this, use it for the lower-
  priority share.
- **P99 / event demand**: product launches, model releases. Burst
  to cloud on-demand with a pre-arranged quota and a tested fallback
  if the quota cannot be filled.

The write-up names the quota, the escalation contact, and the
manual-runbook step for each tier.

### 2.5 Fleet-level reliability

- **Supplier concentration**: a one-vendor fleet is an outage risk.
  The policy should name an acceptable concentration limit
  (e.g., ≤ 70% from the dominant supplier) and the alternative
  path to re-balance if a supplier fails.
- **Region / DC concentration**: at least one secondary region or
  colo capable of running the critical serving workload at reduced
  capacity.
- **Generation concentration**: avoid being stuck entirely on a
  retired generation; the rotation policy above prevents this.
- **Reservation stacking**: do not stack the reservation renewals in
  the same quarter. A single bad vendor negotiation window becomes
  a cliff.

### 2.6 Governance and feedback

The fleet strategy is not a document that is written once. It is a
quarterly review with an owner. The review should at minimum ask:

- Are the segment definitions still true? New workloads?
- Is the utilization-per-tier still inside the target band?
- Is the cost-per-token trend what we projected?
- Are any generations approaching the retirement window?
- Is the burst runbook current (quotas tested, contacts correct)?

A missed quarterly review is the single most common reason fleets
drift into misalignment.

### 2.7 Framework alignment (where it applies)

While this exercise is primarily cost / capacity, two listed
official frameworks have touchpoints:

- **NIST AI Risk Management Framework (AI RMF)** — fleet decisions
  are part of the "Govern" and "Measure" functions: who decides
  capacity, which risk assumptions underpin reservations, how
  degradation is detected and reported.
- **OWASP Machine Learning Security Top 10** / **MITRE ATLAS** —
  touch the fleet strategy indirectly: a multi-tenant serving tier
  has to carry isolation and supply-chain assumptions through to
  the hardware layer (see Exercise 03 for the MIG decision; see
  the `mod-209-security-compliance` module for the full security
  posture).

Call out explicitly in the strategy document which of these the
fleet owns and which are owned elsewhere.

## 3. Validation steps

This is a strategy exercise; validation is a structured review.

1. **Workload audit**: for every workload currently running,
   confirm which segment it is assigned to. Unassigned workloads
   are a missed constraint.
2. **Sourcing audit**: count GPU-hours by tier in the last 90 days.
   Compare to the plan. Deltas > 10% need an explanation.
3. **Cost reconciliation**: finance's bill reconciles to the fleet
   plan within tolerance. Any surprise item is a plan failure.
4. **Burst runbook test**: at least once per quarter, exercise the
   on-demand burst path end-to-end. If the quota call fails, the
   plan has not been validated.
5. **Vendor-outage tabletop**: run the "primary supplier cannot
   deliver" scenario. The plan should produce a concrete action
   list, not "we would scale cloud".

Pass criteria: all five above have artifacts from the last quarter.

## 4. Rubric or review checklist

- [ ] **Workloads are segmented** into at least training vs serving
      vs research/sandbox, with utilization expectations.
- [ ] **Multi-generation posture is explicit**; retirement cadence
      is named.
- [ ] **Sourcing mix is named** (owned / reserved / on-demand / spot),
      with a utilization target per tier.
- [ ] **Cost metric is a rate**, not an absolute — cost per token,
      per request, per step.
- [ ] **Capacity is sized for P50/P95/P99 separately**.
- [ ] **Burst path is runbookable**: quotas, contacts, fallback.
- [ ] **Supplier/region concentration limit stated**.
- [ ] **Reservation renewals are staggered**.
- [ ] **Governance cadence** named (owner, meeting, deliverable).
- [ ] **No "we always use the biggest GPU" posture** without a
      cost-per-token story.
- [ ] **No "the cloud will have capacity"** assumption without a
      runbook that proves it.
- [ ] **Retirement is planned**, not accidental.

## 5. Common mistakes

1. **Single sourcing posture for all workloads.** The economics do
   not work; segments differ by orders of magnitude.
2. **One-generation fleet.** First rotation reveals the stranded
   depreciation.
3. **Reservation sized for peak demand.** The reserved tier is
   right-sized for the *floor*; peaks go to on-demand.
4. **Spot used for SLO-bound workloads.** Preemption of a live
   serving replica is an incident, not a cost saving.
5. **"Utilization" tracked only as nvidia-smi GPU utilization.**
   The right metric is tokens/sec per owned GPU-hour (training)
   and cost per served request (serving), not GPU busy %.
6. **No cost-per-unit metric.** "We spent more this quarter" is
   not a diagnosis; cost-per-token across months is.
7. **Burst path never tested.** The first time the on-demand quota
   is exercised is during an outage, not before.
8. **Vendor concentration not bounded.** A single vendor's delivery
   slip becomes an organizational problem.
9. **Strategy is a document, not a cadence.** Without a quarterly
   owner it ages into fiction.
10. **Cross-module hand-offs ignored.** Security (mod-209),
    observability (mod-207), and IaC/GitOps (mod-208) all have
    hooks into the fleet strategy. A plan that does not say who
    owns which of them is incomplete.

## 6. References

Official standards and project documentation:

- NIST AI Risk Management Framework —
  https://www.nist.gov/itl/ai-risk-management-framework
- OWASP Machine Learning Security Top 10 —
  https://owasp.org/www-project-machine-learning-security-top-10/
- MITRE ATLAS —
  https://atlas.mitre.org/
- NVIDIA data center product line (for generation comparisons) —
  https://www.nvidia.com/en-us/data-center/
- NVIDIA DCGM (for utilization metrics) —
  https://developer.nvidia.com/dcgm
- FinOps Foundation — https://www.finops.org/

Local context:

- Module-level SOLUTION: [`../SOLUTION.md`](../SOLUTION.md)
- Cluster-level design: [`../exercise-04/SOLUTION.md`](../exercise-04/SOLUTION.md)
- GPU sharing decisions: [`../exercise-03/SOLUTION.md`](../exercise-03/SOLUTION.md)
- Related senior-tier modules for the cross-module hand-offs listed
  in Common Mistakes:
  - `modules/mod-207-observability-sre`
  - `modules/mod-208-iac-gitops`
  - `modules/mod-209-security-compliance`
