# SOLUTION — Lab 03: GPU Sharing with Multi-Instance GPU (MIG)

> Read this after you have attempted the lab. MIG is the hard-
> partitioning option; this solution also explains when MPS or
> time-slicing is the correct answer instead — the senior-level
> call is "which sharing mechanism", not "configure MIG".

## 1. Solution overview

MIG lets a single supported NVIDIA GPU present itself as up to
seven hardware-isolated GPU instances, each with its own slice of
SMs, L2 cache, and memory channels. The feature exists on A100 and
newer datacenter parts; the current generations most often used are
A100 (SM80) and H100 (SM90)
<!-- author-note: confirm the exact current list of MIG-capable datacenter GPUs against the NVIDIA MIG User Guide before shipping to users -->.

The senior-level answer is a decision framework followed by the
mechanical configuration:

- **Decide the sharing mode** from workload shape:
  - *MIG*: strong isolation (tenant / SLO separation), predictable
    latency, independent failure. Required when fault isolation
    matters — one instance faulting does not touch another.
  - *MPS* (Multi-Process Service): co-located processes that trust
    each other and want to overlap small kernels; no hardware
    isolation.
  - *Time-slicing* (Kubernetes device plugin feature): best-effort
    sharing for dev or CI; no isolation at all, kernels serialize.
  - *No sharing* (one workload per GPU): training steady-state or
    any workload large enough to saturate a full GPU.
- **Pick an instance profile** that fits the memory and compute
  footprint of the workload, from the published profiles for that
  GPU generation.
- **Reflect the profile in the cluster**: on bare metal, via
  `nvidia-smi mig`; on Kubernetes, via the NVIDIA GPU Operator with
  a MIG strategy that matches the chosen profile.
- **Observe isolation by test**: the lab's acceptance criterion is a
  reproducible demonstration that a workload on one MIG instance
  cannot affect latency on another.

The reference solution walks through enabling MIG on an A100 with a
"1g.10gb × 7" (seven smallest instances) layout, exposing them to
Kubernetes via the GPU Operator, and showing isolation under load.

## 2. Worked answer or implementation

### 2.1 Decision rationale

| Workload | Recommended mode | Why |
|---|---|---|
| Multi-tenant inference with per-tenant SLOs | MIG | Hardware isolation keeps a noisy tenant from blowing another tenant's p99. |
| Many small inference replicas of one model | MIG or time-slicing | MIG if latency SLO matters; time-slicing if cost per replica matters more. |
| CI / smoke-test runners sharing a GPU | Time-slicing | Short jobs, no SLO, cheapest to operate. |
| Jupyter notebooks / developer sandbox | MIG (1g profile) | Caps a noisy notebook's blast radius to its own slice. |
| Interactive serving at low QPS | MIG | Idle capacity on other instances is reclaimable by another workload. |
| Full-model training | None (one workload per GPU) | Saturates the GPU; sharing strictly loses throughput. |
| Many co-operating processes in one model-server | MPS | Low-latency IPC-style overlap; no isolation needed. |

### 2.2 Enabling MIG on the host (A100 / H100)

```bash
# 1. Confirm the GPU supports MIG.
nvidia-smi -q | grep -A2 "MIG Mode"

# 2. Enable MIG. The GPU must be idle (no process, no X).
#    Reboot may be required on some platforms; -r forces a re-init.
sudo nvidia-smi -mig 1                                    # enable on all GPUs
sudo nvidia-smi -i 0 -mig 1                               # or a specific GPU

# 3. List GPU instance profiles available on this GPU.
nvidia-smi mig -lgip

# 4. Create 7 x 1g.10gb instances on GPU 0 (A100 40GB default profile layout).
#    The profile IDs come from -lgip output; names like "1g.10gb" are also accepted.
sudo nvidia-smi mig -i 0 -cgi 1g.10gb,1g.10gb,1g.10gb,1g.10gb,1g.10gb,1g.10gb,1g.10gb -C

# 5. List the resulting instances and their UUIDs.
nvidia-smi -L
```

Design notes:

- The exact profile menu depends on the GPU's SKU (A100 40GB vs
  A100 80GB vs H100 80GB). Always run `nvidia-smi mig -lgip` on
  the host — the counts of GPCs / memory per slice change per
  generation
  <!-- author-note: table of (GPU model, profile name, SM/GPC share, memory GB) should be sourced from the NVIDIA MIG User Guide for the exact generations deployed; do not transcribe from third-party posts -->.
- The number of *compute instances* ("CI") per *GPU instance*
  ("GI") is configurable. For the simple one-CI-per-GI case,
  `-C` after `-cgi` creates a default compute instance per GI.
- `nvidia-smi -mig 1` requires the GPU to be idle. If a kernel
  is running (or a dangling CUDA context exists), the command
  fails — unload anything using the GPU first.

### 2.3 Exposing MIG instances to Kubernetes

Use the NVIDIA GPU Operator (which bundles the device plugin and
MIG manager):

```yaml
# gpu-operator-values.yaml — Helm values for a MIG-enabled node pool.
# NVIDIA GPU Operator docs: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/
migManager:
  enabled: true
  config:
    name: default-mig-parted-config
    default: all-1g.10gb
# Single or Mixed strategy on the device plugin:
#   single: one MIG profile per node; GPUs expose as "nvidia.com/gpu".
#   mixed:  multiple profiles per node; GPUs expose as
#           "nvidia.com/mig-1g.10gb", "nvidia.com/mig-2g.20gb", etc.
devicePlugin:
  config:
    name: mig-strategy
    default: mixed
```

```yaml
# inference.yaml — a Pod requesting one 1g.10gb slice.
apiVersion: v1
kind: Pod
metadata:
  name: tenant-a-inference
spec:
  containers:
    - name: server
      image: ghcr.io/example/inference:1.0
      resources:
        limits:
          nvidia.com/mig-1g.10gb: 1
```

Design notes:

- **`single` strategy** is simpler to operate; the whole node
  publishes one profile and pods request `nvidia.com/gpu`.
- **`mixed` strategy** lets different pods on the same node
  request different profiles; the scheduler sees each profile
  as a distinct extended resource. Required when tenants have
  different memory footprints.
- The MIG manager is responsible for reconfiguring profiles.
  Draining the node and letting the manager apply the new
  layout is the correct procedure; manual `nvidia-smi mig`
  edits on a managed node will be reverted.

### 2.4 Demonstrating isolation

The grader-facing acceptance test is a "noisy neighbor"
reproduction:

```bash
# Pin a saturation workload to MIG instance 0.
CUDA_VISIBLE_DEVICES=MIG-<uuid-0> python saturate_gpu.py &  # 100% SM util

# On MIG instance 1, run a latency-sensitive workload.
CUDA_VISIBLE_DEVICES=MIG-<uuid-1> python measure_latency.py

# Expected: measure_latency.py reports p99 within ~X% of its
# standalone baseline. Record both numbers.
```

The isolation is at the hardware level (SM partitioning, memory
channel partitioning, dedicated L2 slice), so the expected signal
is that the noisy neighbor's effect on the latency workload is
below measurement noise — in contrast to MPS or time-slicing,
where the noisy neighbor dominates.

### 2.5 Operational guardrails

- **Reconfiguration drains the node.** Changing the profile
  requires destroying the existing MIG instances and recreating
  them; any workload holding a slice must be moved first.
- **MIG does not fractionally share across instances.** Unused
  SMs in instance 0 are not available to instance 1. Right-size
  the profile; do not oversubscribe expecting workload average.
- **Instance UUIDs change on reconfiguration.** Any pinning
  (`CUDA_VISIBLE_DEVICES`) outside the device plugin must be
  updated.
- **Nsight tooling has caveats per instance.** Some performance
  counters require elevated privileges and some are per-GPU, not
  per-GI. Plan profiling at provisioning time
  <!-- author-note: confirm the current ncu/nsys MIG support matrix against the Nsight Compute release notes before quoting specifics -->.

## 3. Validation steps

```bash
# Prereqs: A MIG-capable NVIDIA GPU (A100 or newer datacenter part),
# NVIDIA driver and CUDA that match the GPU, Kubernetes >= 1.24,
# NVIDIA GPU Operator installed.

# 1. Enable MIG and create the planned layout on one node.
sudo nvidia-smi -mig 1
nvidia-smi mig -lgip
sudo nvidia-smi mig -i 0 -cgi 1g.10gb,1g.10gb,1g.10gb,1g.10gb,1g.10gb,1g.10gb,1g.10gb -C
nvidia-smi -L

# 2. Install the GPU Operator with MIG manager enabled; wait for the
#    device plugin to publish the mig-* extended resources.
kubectl get node <node> -o json | jq '.status.allocatable | with_entries(select(.key|startswith("nvidia.com/")))'

# 3. Deploy one Pod requesting nvidia.com/mig-1g.10gb and verify it binds
#    to the expected slice.
kubectl apply -f inference.yaml
kubectl exec tenant-a-inference -- nvidia-smi -L

# 4. Run the noisy-neighbor isolation test and record p99 delta.
```

Pass criteria:

- All seven MIG instances appear in `nvidia-smi -L`.
- Kubernetes allocatable resources report the expected
  `nvidia.com/mig-*` count.
- The noisy-neighbor latency test shows the sensitive workload
  within measurement noise of its standalone baseline.

## 4. Rubric or review checklist

- [ ] **Sharing mode decision is explicit** (MIG vs MPS vs time-
      slicing vs none) with workload-shape justification.
- [ ] **Profile choice is justified** against the workload's memory
      and compute footprint, not "whichever fits".
- [ ] **MIG enablement procedure is idempotent** — rerunning it on a
      node that is already configured does not break running pods.
- [ ] **Device plugin strategy is stated** (single vs mixed) and
      matches the profile layout.
- [ ] **Isolation is demonstrated**, not asserted — a noisy-neighbor
      test with before/after numbers.
- [ ] **Reconfiguration procedure documented**: drain the node,
      change layout, verify, re-admit workloads.
- [ ] **Failure modes listed**: what happens if MIG manager cannot
      apply the layout; where to find the error.
- [ ] **No oversubscription assumption**: the write-up acknowledges
      MIG instances do not share idle capacity.
- [ ] **Profiles use the names from `nvidia-smi mig -lgip` on the
      actual target GPU**, not copy-pasted from a different SKU.

## 5. Common mistakes

1. **Choosing MIG because "GPUs are expensive"** without checking
   whether the workload actually fits a small profile. A model that
   needs 24GB of memory cannot run on a 1g.10gb slice.
2. **Picking MIG for a workload that uses the whole GPU already.**
   Any sharing mode loses throughput here; the answer is "don't
   share".
3. **Confusing MIG with MPS.** MPS is cooperative scheduling on a
   shared GPU with no isolation; MIG is a hardware partition.
4. **Expecting unused capacity on one MIG instance to help another.**
   It cannot; they are hard-partitioned.
5. **Changing profiles under live workloads.** The layout change
   tears down and recreates instances; pods lose their device.
6. **Forgetting to enable MIG on every GPU in the node.** Mixed
   MIG/non-MIG GPUs on one host are allowed but complicate the
   device plugin configuration.
7. **Hardcoding MIG UUIDs** in deployment manifests. UUIDs change
   across reconfiguration; use the device plugin's resource names.
8. **No isolation test.** The write-up says "MIG provides isolation"
   but never proves it under load.
9. **Picking the smallest profile** by default. Many small inference
   replicas can be slower end-to-end than fewer larger ones because
   batch size and parallelism opportunities disappear.

## 6. References

- NVIDIA Multi-Instance GPU User Guide —
  https://docs.nvidia.com/datacenter/tesla/mig-user-guide/
- NVIDIA GPU Operator —
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/
- NVIDIA Kubernetes device plugin —
  https://github.com/NVIDIA/k8s-device-plugin
- NVIDIA MPS documentation —
  https://docs.nvidia.com/deploy/mps/
- NVIDIA A100 architecture whitepaper —
  https://images.nvidia.com/aem-dam/en-zz/Solutions/data-center/nvidia-ampere-architecture-whitepaper.pdf
- NVIDIA H100 datasheet —
  https://www.nvidia.com/en-us/data-center/h100/

Local context:

- Module-level SOLUTION: [`../SOLUTION.md`](../SOLUTION.md)
- Companion engineer-tier module: `engineer-solutions/mod-107`
