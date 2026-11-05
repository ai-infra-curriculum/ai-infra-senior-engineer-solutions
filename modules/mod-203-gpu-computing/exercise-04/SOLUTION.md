# SOLUTION — Lab 04: GPU Cluster Design Exercise

> This is a design exercise. There is no single correct cluster; there
> is a defensible one. The grader is evaluating the reasoning chain,
> not the specific vendor choices.

## 1. Solution overview

A senior-tier GPU cluster design starts from the workload and backs
into topology, networking, storage, and scheduling. The reference
answer is a procedure:

1. **Characterize the workload** quantitatively: number of
   simultaneous jobs, parameter count, tokens/step, dataset size on
   disk, read bandwidth per step, write bandwidth per step, failure
   recovery requirements.
2. **Pick the GPU class** from compute and memory requirements, not
   from availability. For large-model training, 80GB class
   (H100 SXM or A100 80GB SXM) is the default; for small models or
   serving, a smaller class is correct.
3. **Decide the interconnect** by the model/parallelism plan:
   - Intra-node: NVLink + NVSwitch where available (HGX platforms).
   - Inter-node: InfiniBand (NDR for H100 generation) or
     high-speed Ethernet (RoCEv2) sized to the collectives traffic.
4. **Decide the storage tier**: a fast scratch tier for active
   datasets, a durable tier for archives, both reachable from the
   GPU nodes at the bandwidth the model actually consumes.
5. **Decide the scheduler and the queues**: training jobs are
   gang-scheduled; serving is autoscaled. They should not share the
   same queue and often not the same hardware.
6. **Fault domains**: at least one spare per 16 nodes, and the
   checkpoint store must survive a rack failure.

The worked example below sizes a cluster for a representative
transformer-training workload and a co-located inference workload.

## 2. Worked answer or implementation

### 2.1 Reference workload

The exercise assumes two workloads sharing one cluster:

| Workload | Notes |
|---|---|
| Training | One dense transformer, ~10B parameters, bf16/fp16, fits in 3D parallelism across 64 GPUs. Training cadence: continuous. Checkpoint every 30 minutes. Dataset: 2 TB, read-heavy. |
| Serving | ~30 replicas of a smaller model. Latency SLO p99 < 300 ms. QPS varies 10x across the day. |

### 2.2 GPU class and node shape

- **Training nodes**: 8-GPU SXM boards (HGX H100 or HGX A100 80GB).
  Reason: NVLink/NVSwitch inside the node gives bandwidth and
  topology that PCIe boards cannot match for tensor-parallel work.
- **Serving nodes**: PCIe GPUs of the same or previous generation.
  Reason: the serving replicas do not need intra-node NVLink; PCIe
  gives better cost-per-replica and allows MIG where the model
  fits a slice.
- **Head / login / build nodes**: CPU-only; GPUs are not scarce
  enough to waste on `tar` or `pip install`.

Why SXM over PCIe for training: SXM platforms expose a full NVLink
mesh via NVSwitch so every GPU-to-GPU hop inside the node is at
NVLink speed, not PCIe speed
<!-- author-note: current intra-node NVLink/NVSwitch bandwidth figures for H100 HGX should be cited from the HGX datasheet in the final write-up -->.

### 2.3 Inter-node fabric

- **Training fabric**: a rail-aligned InfiniBand (or RoCE) fabric
  where each GPU's NIC maps to a dedicated rail, and intra-rail
  traffic takes the shortest path. This layout is the reference
  topology for training clusters because allreduce traffic maps
  cleanly to rails.
- **NIC count**: typically one high-speed NIC per GPU on HGX
  training nodes (e.g., eight NICs on an 8-GPU node)
  <!-- author-note: exact NIC-per-GPU count depends on the reference HGX SKU; confirm against NVIDIA's reference designs before shipping -->.
- **Storage fabric**: separate or VLAN'd from compute fabric.
  Mixing dataset reads with collectives is a known tail-latency
  source.

### 2.4 Storage

Two tiers:

1. **Scratch** for active training data and checkpoints:
   a parallel filesystem (Lustre, GPFS, or a vendor equivalent)
   sized for the aggregate read bandwidth the step consumes.
   Example: a step that reads 50 GB/s across the cluster needs
   headroom of 2–3x that to survive bursts.
2. **Durable** for dataset archives and long-lived checkpoints:
   object storage (S3 or compatible), lifecycled, cheap-per-byte,
   not in the training-step hot path.

The write-up must state the step's read bandwidth and show the
scratch tier exceeds it with margin.

### 2.5 Scheduler and queues

- **Training**: Kubernetes with an AI-aware scheduler (Volcano,
  KAI/Kueue, or similar) that gang-schedules multi-node jobs.
  Alternative: Slurm — a defensible choice when the shop is HPC-
  shaped and already operates it. Document the choice; do not
  adopt both.
- **Serving**: Kubernetes with HPA on a serving-specific latency
  metric (tokens/sec, p99 per replica), not CPU utilization.
- **Separate queues** for training and serving. A slow training job
  must not preempt a live SLO.
- **Priority classes** for emergency serving ramps; a documented
  preemption policy so operators can predict what moves.

### 2.6 Fault domains and spares

- **Rack affinity labels** on nodes; jobs with replication requirements
  spread across rack labels.
- **Spare capacity**: at least one spare per 16 training nodes. A
  single DIMM, NIC, or GPU failure happens often enough that "fail
  the whole job" is not a sane recovery.
- **Checkpoint store** on durable storage in at least two failure
  domains. A checkpoint that lives only on the scratch volume is
  not a backup.
- **Head node redundancy**: at least two login hosts.

### 2.7 Observability

Minimum metrics shipped off-cluster:

- Per-GPU utilization, memory, temperature, ECC errors
  (`DCGM-Exporter` is the reference).
- NIC counters, including per-queue drops.
- Filesystem throughput and IOPS.
- Scheduler queue age (training) and pod startup time (serving).

The write-up names which alarms wake a human and which auto-
remediate.

### 2.8 Reference diagram

```
                       +----------- object store (durable) -----------+
                       |                                              |
                 +-----v------+   object/S3 tier                      |
                 |  archives   |                                      |
                 +-------------+                                      |
                                                                      |
  +----------------------- cluster fabric -----------------------+    |
  |                                                              |    |
  |  training pod                     serving pod                |    |
  |  (HGX H100 8-GPU, NVSwitch)       (PCIe GPUs, optional MIG)  |    |
  |  - gang scheduled                 - autoscaled               |    |
  |  - allreduce over IB/RoCE rail    - no RDMA required         |    |
  |                                                              |    |
  +------^-------------------------------------------------------+    |
         |                                                            |
         +------ parallel FS (Lustre/GPFS) scratch tier ---------------+
```

### 2.9 Cost posture and procurement

- **Right-size before scaling out.** A single HGX node often wins on
  utilization for workloads under ~30B parameters; multi-node
  training should be a decision, not a default.
- **Multi-generation fleet is normal.** Previous-generation GPUs
  stay useful for serving and smaller jobs long after they stop
  being the training default.
- **Reservations vs on-demand**: reserved for the always-on floor
  (24/7 training), on-demand or spot for burst — see Exercise 05
  for the fleet-level strategy.

## 3. Validation steps

This is a design exercise; the "validation" is a review protocol.

1. **Peer review**: present the design to a reviewer unfamiliar
   with the workload. Reviewer confirms each component answers a
   specific sentence from the workload characterization.
2. **Quantitative sanity checks**:
   - Scratch read bandwidth ≥ 2x the step's read bandwidth.
   - Collectives bandwidth demand (parameter size × iterations)
     fits within the fabric's per-rail bandwidth with margin.
   - Memory per GPU ≥ parameter shard + activation shard +
     optimizer-state shard with headroom.
3. **Blast-radius check**: enumerate a rack failure, a switch
   failure, a parallel-filesystem outage. For each, state what
   fails, what keeps running, and the recovery time objective.
4. **Observability check**: for each component, name the alarm
   that would fire first if it degraded.
5. **Dry-run the procurement**: at least one supplier-published
   SKU exists for every component class named.

## 4. Rubric or review checklist

- [ ] **Workload characterization is quantitative**, not adjectival.
- [ ] **GPU class choice is justified** against model size and
      parallelism plan.
- [ ] **Intra-node interconnect is explicit** (NVLink/NVSwitch for
      training nodes; PCIe acknowledged as the serving choice).
- [ ] **Inter-node fabric is sized** against the collective's
      bandwidth, not "fast enough".
- [ ] **Storage is two-tiered** (scratch + durable) with bandwidth
      arithmetic for the scratch tier.
- [ ] **Training and serving are on different queues**, with the
      policy for sharing hardware documented.
- [ ] **Fault domains are named** (rack labels, spares, checkpoint
      redundancy).
- [ ] **A rack/switch/FS failure story exists** with RTO/RPO.
- [ ] **Observability names specific exporters/dashboards** rather
      than "we will have monitoring".
- [ ] **Diagram or topology map accompanies the write-up**.
- [ ] **Multi-generation fleet is addressed** (serving on previous
      gen while training is on current).
- [ ] **No "we will use the best GPUs" claims** without memory and
      FLOP math that justifies them.

## 5. Common mistakes

1. **Starting from a vendor SKU, not the workload.** The first
   sentence of the design should be the workload, not "we chose
   H100".
2. **Mixing training and serving on the same queue.** One bad
   training job silently evicts live inference.
3. **Under-sizing the storage tier.** Training idle because the
   dataset reads cannot keep up is a very expensive failure.
4. **Forgetting checkpoint durability.** A scratch-only checkpoint
   does not survive a filesystem outage.
5. **PCIe where SXM/NVSwitch is needed.** Tensor-parallel work over
   PCIe is a measurable regression the first time you scale.
6. **One failure domain.** Any rack-switch failure that takes down
   the cluster is a design fault, not an incident.
7. **Observability deferred to "later".** Later means you cannot
   explain the first outage.
8. **Over-building for the peak.** A cluster sized for the busiest
   week spends 50 weeks of the year underused; see Exercise 05 for
   the burst strategy.
9. **Hardcoding today's GPU generation.** The cluster will outlive
   it; the design should name the upgrade path.
10. **No cost arithmetic at all.** A design that cannot answer
    "what does this cost per month" is incomplete.

## 6. References

- NVIDIA DGX / HGX reference architectures —
  https://www.nvidia.com/en-us/data-center/dgx-platform/
- NVIDIA NCCL (collectives library) —
  https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/
- NVIDIA DCGM and DCGM-Exporter —
  https://developer.nvidia.com/dcgm
- NVIDIA GPU Operator —
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/
- Kubernetes device plugin —
  https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/
- Kueue (Kubernetes batch queueing) —
  https://kueue.sigs.k8s.io/
- Volcano scheduler —
  https://volcano.sh/en/docs/
- Slurm —
  https://slurm.schedmd.com/
- Lustre —
  https://www.lustre.org/
- InfiniBand Architecture Specification (IBTA) —
  https://www.infinibandta.org/ibta-specifications-download/

Local context:

- Module-level SOLUTION: [`../SOLUTION.md`](../SOLUTION.md)
- Fleet-scale operational strategy: `../exercise-05/SOLUTION.md`
- Companion engineer-tier module: `engineer-solutions/mod-107`
