# SOLUTION — Lab 02: GPU Profiling and Performance Optimization

> Read this after you have attempted the lab. The module-level
> framing in [`../SOLUTION.md`](../SOLUTION.md) applies — in
> particular Decisions 3, 4, and 5 (roofline-driven, ncu for tuning,
> warm-up + statistics).

## 1. Solution overview

A senior-tier profiling pass is a procedure, not a metric collection.
The reference workflow is:

1. **Scope**: define what "faster" means — training step time, tokens/sec
   per GPU, serving p99 latency, or cost per request. Pick one; it
   decides every downstream choice.
2. **Start high**: profile the whole program with Nsight Systems
   (`nsys`) to find which kernels and which CPU gaps actually matter.
3. **Compute a roofline** for the top-k hot kernels: peak compute,
   peak DRAM bandwidth, achieved % of each. Classify each as
   compute-bound, bandwidth-bound, or launch/overhead-bound.
4. **Drill in**: profile the hot kernel with Nsight Compute (`ncu`).
   Read the hardware counters named in the module SOLUTION
   (`sm__throughput`, `dram__throughput`, occupancy) — not wall clock.
5. **Change one thing at a time**: tile size, dtype, memory layout,
   kernel fusion, launch amortization. Re-measure.
6. **Stop when the kernel is within ~10% of the roofline ceiling**
   or when the whole-program share no longer justifies further work.

The reference exercise applies this to a toy transformer-attention
block to reproduce a canonical "GPU bound → bandwidth bound → launch
bound" progression as the kernel is tuned.

## 2. Worked answer or implementation

### 2.1 Top-down profile with Nsight Systems

```bash
# Record a single step (export NCCL_P2P_DISABLE=0 for multi-GPU work).
nsys profile \
  --trace=cuda,nvtx,osrt,cudnn,cublas \
  --output=step \
  --force-overwrite=true \
  python train_step.py --iters 20

# Inspect in the Nsight Systems GUI; or summarize on the CLI:
nsys stats step.nsys-rep --report cudaapisum --report gputrace
```

What to look for:

- **GPU idle gaps** between kernels. Gaps > a few hundred microseconds
  almost always mean a Python / dataloader / CPU-side synchronization
  stall. The fix is upstream of the kernel.
- **Top-10 kernels by wall-clock share.** Everything below the tenth
  is noise unless the aggregate matters. The decision rule from the
  module SOLUTION applies: a kernel that is 3% of runtime is not
  worth 30 minutes of tuning.
- **Launch overhead**: many sub-microsecond kernels in a row indicate
  a case for CUDA Graphs. Note it; address in a separate step.

### 2.2 Roofline calculation

For each hot kernel compute:

- **Arithmetic intensity** `I = FLOPs / Bytes moved` (FLOPs and bytes
  from the algorithm, not the implementation).
- **Peak compute** `P` (FLOP/s) and **peak bandwidth** `B` (byte/s)
  from the GPU spec.
- **Roofline ceiling** `min(P, I * B)`.
- **Achieved %** = measured FLOP/s / ceiling.

Example (fp16 GEMM on an NVIDIA A100 80GB SXM4):

| Quantity | Value | Source |
|---|---|---|
| Peak fp16 Tensor Core throughput | 312 TFLOP/s (dense) | NVIDIA A100 datasheet |
| Peak HBM2e bandwidth | 2039 GB/s | NVIDIA A100 datasheet |
| GEMM `(8192, 8192) x (8192, 8192)` arithmetic intensity | ~2730 FLOP/B | `2 * M * N * K / (2 * (M*K + K*N + M*N))` for fp16 |
| Ceiling | compute-bound at `min(312 TFLOP/s, 2730 * 2039 GB/s)` → 312 TFLOP/s | derived |

If the kernel reports <70% of 312 TFLOP/s it has headroom; if it
reports >90% the ceiling is near and further work is wasted. Values
for H100 and consumer parts differ; always re-run the numbers from
the datasheet of the GPU you measured on
<!-- author-note: confirmed H100 SXM5 fp16 Tensor Core and HBM3 bandwidth figures should be pulled from the current NVIDIA H100 datasheet before quoting -->.

### 2.3 Kernel drill-down with Nsight Compute

```bash
# Collect the recommended "full" set for an initial look.
ncu --set full \
    --target-processes all \
    -k regex:"my_hot_kernel|aten::_scaled_dot_product_attention" \
    -o kernel_profile \
    python train_step.py --iters 2

# Headline metrics on the CLI, no GUI needed:
ncu --import kernel_profile.ncu-rep --page details --metrics \
    sm__throughput.avg.pct_of_peak_sustained_elapsed,\
    dram__throughput.avg.pct_of_peak_sustained_elapsed,\
    launch__occupancy_limit_registers,\
    launch__occupancy_limit_shared_mem,\
    sm__warps_active.avg.pct_of_peak_sustained_active
```

Reading rules:

- **sm__throughput high, dram__throughput low** → compute-bound.
  Room to improve only through algorithmic changes (fusion, lower
  precision, Tensor Core-friendly tile shapes).
- **dram__throughput high, sm__throughput low** → bandwidth-bound.
  Fuse adjacent kernels, change layout for coalesced access, or move
  to a smaller dtype.
- **Both low** → launch, occupancy, or synchronization is the
  bottleneck. Check occupancy limits and look for serialization
  (`cudaDeviceSynchronize`, implicit syncs on `item()` /
  `.cpu()` / `print`).
- **Register / shared-memory limited occupancy** → reduce register
  pressure (break the kernel up, lower unrolling) or shared-mem
  usage (smaller tiles).

### 2.4 One canonical optimization pass

The reference pass on a transformer block:

1. **Baseline** (eager PyTorch): nsys shows attention as 42% of step
   time; `aten::matmul` is next at 17%; the rest is small.
2. **Switch attention to `torch.nn.functional.scaled_dot_product_attention`**
   (which dispatches to FlashAttention on supported hardware):
   attention drops to 15% of step time, step time drops ~2x.
3. **Enable mixed precision** (`torch.cuda.amp.autocast` + `GradScaler`):
   the remaining matmuls move to Tensor Core-friendly dtypes;
   `sm__throughput` on the main matmul jumps from ~35% to ~75% of
   peak. These relative gains are expected on current NVIDIA GPUs
   (A100/H100 Tensor Cores)
   <!-- author-note: specific percentage-of-peak numbers should be reproduced on the target GPU; cite measured values in the write-up, not these illustrative ones -->.
4. **Capture with CUDA Graphs**: eliminates small launch overheads
   between adjacent kernels. Useful only when the kernel sequence
   is static and launch overhead is a measurable share (nsys will
   show it as GPU idle between tiny kernels).
5. **Stop**: the hot kernel is now within the ~10% band of the
   roofline ceiling; further work shifts to I/O and dataloader.

Report each step's effect as a before/after table with p50, p95,
p99, and the roofline-% achieved.

### 2.5 Report template

Every optimization write-up includes:

```
# Optimization pass — <workload name>, <GPU model>, <date>

## Baseline
- Workload: <description, batch, sequence, model>
- GPU: <A100-80GB SXM4 | H100 SXM5 | ...>
- Step time (p50 / p95 / p99): ...
- Top-5 kernels by share: <table>

## Roofline
- Hot kernel: <name>
- Arithmetic intensity: <value> FLOP/B
- Ceiling: <value> TFLOP/s or GB/s
- Achieved: <value> (<pct>% of ceiling)

## Changes
1. <change>: step time <before> -> <after>; roofline % <before> -> <after>.
2. ...

## Decision
- Ship / do not ship.
- Reason: <within X% of ceiling | shifted bottleneck to Y>.
```

## 3. Validation steps

```bash
# Prereqs: CUDA 12+, NVIDIA driver supporting your profiler versions,
# nsight-systems and nsight-compute installed (ship with the CUDA toolkit).
nsys --version
ncu --version
nvidia-smi

# 1. End-to-end profile (fast, broad)
nsys profile --output=baseline python train_step.py --iters 20
nsys stats baseline.nsys-rep --report gputrace --report cudaapisum

# 2. Kernel-level drill-down (slow, narrow)
ncu --set full -k regex:"<hot_kernel>" -o kernel_profile python train_step.py --iters 2

# 3. Apply one change; repeat both. Commit the diff only if BOTH
#    (a) step time improved beyond noise AND
#    (b) a hardware counter explains the improvement.
```

## 4. Rubric or review checklist

- [ ] **Scope stated**: what metric the optimization is targeting.
- [ ] **Baseline captured** with `nsys` before anything else.
- [ ] **Top-k kernels enumerated** with wall-clock share, not cherry-
      picked.
- [ ] **Roofline computed** for the hot kernel, not estimated.
- [ ] **Bound classification stated**: compute / bandwidth / launch.
- [ ] **`ncu` counters used** to confirm the classification before
      any change is made.
- [ ] **One change at a time**; each change paired with a before/after
      measurement.
- [ ] **Distribution reported** (p50/p95/p99), not single wall-clock.
- [ ] **`cudaDeviceSynchronize` and warm-up handled** in any custom
      timing.
- [ ] **Stopping condition stated**: within X% of ceiling, or
      bottleneck has shifted, or budget exhausted.
- [ ] **No "Nx speedup" claim** without the measurement protocol
      attached.
- [ ] **No optimization of a kernel under ~5% of end-to-end** unless
      the aggregate share justifies it.

## 5. Common mistakes

1. **Starting with `ncu` instead of `nsys`.** You cannot pick the
   right kernel to tune without the top-down view.
2. **Timing with Python `time.time()` in the training loop.** The
   kernel launches are async; the loop's wall clock is measuring the
   launch, not the work, until the next synchronization.
3. **Optimizing a kernel that is 3% of runtime.** Even if you double
   it, the end-to-end win is 1.5%.
4. **Changing two things at once.** You cannot attribute the delta.
5. **No roofline.** "It's faster" is not a bound; the next GPU can
   regress it.
6. **Reading `sm__throughput` without `dram__throughput`.** Either
   number alone is uninterpretable.
7. **Confusing "GPU utilization" (from `nvidia-smi`) with occupancy
   or throughput.** `nvidia-smi` utilization is a coarse "the GPU
   had a kernel running during the sample window" bit, not a
   performance metric.
8. **Ignoring launch overhead.** Many tiny kernels in a row can be
   50% of step time even when each one is well-tuned.
9. **Benchmarking only on one shape and one dtype.** The ranking of
   optimizations changes with shape and dtype.

## 6. References

- NVIDIA Nsight Systems — https://docs.nvidia.com/nsight-systems/
- NVIDIA Nsight Compute — https://docs.nvidia.com/nsight-compute/
- NVIDIA CUDA Best Practices Guide — https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/
- NVIDIA A100 datasheet (for peak compute and bandwidth figures) —
  https://www.nvidia.com/en-us/data-center/a100/
- NVIDIA H100 datasheet —
  https://www.nvidia.com/en-us/data-center/h100/
- PyTorch profiler — https://pytorch.org/docs/stable/profiler.html
- Williams, Waterman, Patterson. *Roofline: An Insightful Visual
  Performance Model for Multicore Architectures.* (The canonical
  roofline paper; use for the model, not for GPU-specific
  numbers.) <!-- author-note: confirm preferred public citation link before shipping; the paper is widely available -->

Local context:

- Module-level SOLUTION: [`../SOLUTION.md`](../SOLUTION.md) — see
  Decisions 3, 4, and 5.
- Companion engineer-tier module: `engineer-solutions/mod-107`.
