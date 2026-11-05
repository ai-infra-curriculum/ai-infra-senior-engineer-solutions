# SOLUTION — Lab 01: Writing Custom CUDA Kernels for ML Operations

> Read this after you have attempted the lab. The goal is to show *when*
> a custom kernel is warranted, *how* a correct and well-structured one
> looks, and *how* to decide whether the result is good enough to ship.
> Module-level framing in [`../SOLUTION.md`](../SOLUTION.md) applies.

## 1. Solution overview

The senior-level answer is less about writing a specific kernel than about
operating a kernel workflow. For every candidate operation:

1. **Decide whether to write a kernel at all.** Prefer a library op
   (cuBLAS, cuDNN, cuBLASLt, cuSPARSE) unless profiling shows the op is
   both (a) a measurable share of end-to-end latency and (b) misaligned
   with any library op's assumed shape, dtype, or access pattern.
2. **Pick the implementation tier.** Default to Triton; drop to CUDA
   C++ only when Triton cannot express the algorithm (see module
   Decision 1).
3. **Write the kernel against a reference.** The reference is a PyTorch
   eager-mode implementation. The kernel must match it bit-for-bit for
   integer ops and within a stated tolerance for float ops.
4. **Autotune block sizes, num warps, num stages.** Empirically — the
   module-level SOLUTION explains why.
5. **Benchmark against the library baseline and the eager reference**
   using the warm-up and statistics protocol from the module-level
   SOLUTION. If the custom kernel does not clearly beat both, do not
   ship it.

The reference solution exercises this flow on two worked kernels:

- **Element-wise fused op** (e.g., GELU on the forward path) —
  bandwidth-bound; demonstrates coalesced loads, vectorized memory
  access, and how roofline analysis sets the ceiling.
- **Softmax over the last axis** — a numerically delicate reduction
  that benchmarks competitively against `torch.nn.functional.softmax`
  because the fused Triton version keeps the row in registers across
  the max, exp, and divide passes.

## 2. Worked answer or implementation

### 2.1 Triton — fused GELU forward

```python
# gelu_kernel.py
# Fused GELU (tanh approximation) forward in Triton.
# Reference: https://pytorch.org/docs/stable/generated/torch.nn.GELU.html
# Triton docs: https://triton-lang.org/

import torch
import triton
import triton.language as tl


@triton.autotune(
    configs=[
        triton.Config({"BLOCK": 1024}, num_warps=4),
        triton.Config({"BLOCK": 2048}, num_warps=4),
        triton.Config({"BLOCK": 2048}, num_warps=8),
        triton.Config({"BLOCK": 4096}, num_warps=8),
    ],
    key=["n_elements"],
)
@triton.jit
def _gelu_fwd(
    x_ptr, y_ptr, n_elements,
    BLOCK: tl.constexpr,
):
    pid = tl.program_id(0)
    offsets = pid * BLOCK + tl.arange(0, BLOCK)
    mask = offsets < n_elements
    x = tl.load(x_ptr + offsets, mask=mask, other=0.0).to(tl.float32)
    # GELU tanh approximation per Hendrycks & Gimpel; also what
    # torch.nn.GELU(approximate="tanh") computes.
    k0 = 0.7978845608028654       # sqrt(2/pi)
    k1 = 0.044715
    inner = k0 * (x + k1 * x * x * x)
    y = 0.5 * x * (1.0 + tl.math.tanh(inner))
    tl.store(y_ptr + offsets, y.to(x_ptr.dtype.element_ty), mask=mask)


def gelu_fwd(x: torch.Tensor) -> torch.Tensor:
    assert x.is_cuda and x.is_contiguous()
    y = torch.empty_like(x)
    n = x.numel()
    grid = lambda meta: (triton.cdiv(n, meta["BLOCK"]),)
    _gelu_fwd[grid](x, y, n)
    return y
```

Design notes:

- **Load → fp32 → store** keeps numerics stable even when the tensor is
  fp16/bf16. The op is bandwidth-bound, so the extra fp32 ALU work is
  free.
- **Autotune key is `n_elements`** because the only shape dimension that
  matters here is total length. Different shapes do not deserve different
  tilings.
- **No special last-block handling**: `mask` already handles the tail.

### 2.2 Triton — row-wise softmax

```python
# softmax_kernel.py
# Numerically stable row-wise softmax.
# Matches torch.nn.functional.softmax(x, dim=-1) within fp16 tolerance.

import torch
import triton
import triton.language as tl


@triton.jit
def _softmax_row(
    x_ptr, y_ptr,
    stride_x_row, stride_y_row,
    n_cols,
    BLOCK: tl.constexpr,
):
    row = tl.program_id(0)
    cols = tl.arange(0, BLOCK)
    mask = cols < n_cols
    x_row_ptr = x_ptr + row * stride_x_row + cols
    x = tl.load(x_row_ptr, mask=mask, other=-float("inf")).to(tl.float32)
    x = x - tl.max(x, axis=0)         # subtract max for stability
    num = tl.exp(x)
    den = tl.sum(num, axis=0)
    y = num / den
    y_row_ptr = y_ptr + row * stride_y_row + cols
    tl.store(y_row_ptr, y.to(x_ptr.dtype.element_ty), mask=mask)


def softmax(x: torch.Tensor) -> torch.Tensor:
    assert x.is_cuda and x.dim() == 2 and x.is_contiguous()
    n_rows, n_cols = x.shape
    # BLOCK must be a compile-time constant and >= n_cols.
    BLOCK = triton.next_power_of_2(n_cols)
    num_warps = 4 if BLOCK <= 1024 else 8 if BLOCK <= 4096 else 16
    y = torch.empty_like(x)
    _softmax_row[(n_rows,)](
        x, y,
        x.stride(0), y.stride(0),
        n_cols,
        BLOCK=BLOCK, num_warps=num_warps,
    )
    return y
```

Design notes:

- **One program per row** is the right granularity when `n_cols` fits in
  one block of registers. For very wide rows (e.g., vocab-size softmax),
  this needs to spill to shared memory or split across programs — call
  that out explicitly instead of hiding it.
- **`BLOCK = next_power_of_2(n_cols)`** forces a compile-time constant;
  Triton requires constexpr block dims and reductions are easier when
  the block is a power of two.
- **Subtract max before exp** is the standard numerical-stability trick;
  without it, large logits overflow in fp32.

### 2.3 Correctness harness

```python
# test_kernels.py
import torch
import pytest
from gelu_kernel import gelu_fwd
from softmax_kernel import softmax


@pytest.mark.parametrize("shape", [(1024,), (4095,), (1 << 20,)])
@pytest.mark.parametrize("dtype", [torch.float16, torch.bfloat16, torch.float32])
def test_gelu(shape, dtype):
    torch.manual_seed(0)
    x = torch.randn(*shape, device="cuda", dtype=dtype)
    got = gelu_fwd(x)
    want = torch.nn.functional.gelu(x, approximate="tanh")
    atol = {torch.float16: 1e-2, torch.bfloat16: 2e-2, torch.float32: 1e-5}[dtype]
    torch.testing.assert_close(got, want, atol=atol, rtol=1e-3)


@pytest.mark.parametrize("shape", [(128, 1024), (32, 4096), (8, 127)])
@pytest.mark.parametrize("dtype", [torch.float16, torch.bfloat16, torch.float32])
def test_softmax(shape, dtype):
    torch.manual_seed(0)
    x = torch.randn(*shape, device="cuda", dtype=dtype)
    got = softmax(x)
    want = torch.nn.functional.softmax(x, dim=-1).to(dtype)
    atol = {torch.float16: 1e-2, torch.bfloat16: 2e-2, torch.float32: 1e-6}[dtype]
    torch.testing.assert_close(got, want, atol=atol, rtol=1e-3)
```

### 2.4 Benchmark harness (warm-up + distribution)

```python
# bench.py
import time
import statistics
import torch
from gelu_kernel import gelu_fwd
from softmax_kernel import softmax


def bench(fn, *args, warmup=5, iters=50):
    for _ in range(warmup):
        fn(*args)
    torch.cuda.synchronize()
    samples = []
    for _ in range(iters):
        torch.cuda.synchronize()
        t0 = time.perf_counter()
        fn(*args)
        torch.cuda.synchronize()
        samples.append(time.perf_counter() - t0)
    samples.sort()
    p50 = samples[len(samples) // 2]
    p95 = samples[int(len(samples) * 0.95)]
    p99 = samples[int(len(samples) * 0.99)]
    return {"p50": p50, "p95": p95, "p99": p99, "mean": statistics.mean(samples)}


if __name__ == "__main__":
    x = torch.randn(4096, 4096, device="cuda", dtype=torch.float16)
    print("triton_gelu    ", bench(gelu_fwd, x))
    print("torch_gelu     ", bench(torch.nn.functional.gelu, x, approximate="tanh"))
    print("triton_softmax ", bench(softmax, x))
    print("torch_softmax  ", bench(lambda t: torch.nn.functional.softmax(t, dim=-1), x))
```

The reported metric is p50 with p95/p99 as the dispersion signal, per
the module-level SOLUTION's Decision 5.

### 2.5 Decision rationale

| Choice | Alternative | Why we chose it |
|---|---|---|
| Triton over raw CUDA C++ | Hand-written `__global__` with shared memory | Triton compiles to performant SASS for these ops and halves the code size (module Decision 1). Drop to CUDA only if Triton cannot express the pattern. |
| `@autotune` on GELU | Fixed BLOCK=1024 | One tiling does not survive both A100 and H100 (module Decision 2). |
| Load → fp32 → store for GELU | Compute in input dtype | GELU is bandwidth-bound; fp32 ALU work is free and prevents tanh overflow. |
| Row-per-program softmax | Split-row softmax | Simpler and faster for the common case (`n_cols <= 16k`). The caller is told, in the assertion, where the limit is. |

## 3. Validation steps

```bash
# Prereqs: CUDA 12+, PyTorch >= 2.1 with Triton, an SM80+ GPU (A100 or newer).
pip install "torch>=2.1" pytest
# 1. Correctness
pytest -q test_kernels.py
# 2. Benchmarks against torch reference
python bench.py
# 3. Hardware counters for the kernel under tuning (optional but expected
#    at senior level; see Lab 02 for the full profiling workflow).
ncu --set full -k _gelu_fwd python bench.py
```

Pass criteria:

- All `test_kernels.py` tests pass on at least one of fp16/bf16/fp32.
- `bench.py` reports p50/p95/p99 for Triton and Torch.
- The Triton kernel beats (or ties within noise) the Torch reference on
  the fp16 case at `(4096, 4096)` for softmax. GELU in Torch already
  calls a fused implementation, so matching Torch is acceptable; beating
  it is not expected without further work.

## 4. Rubric or review checklist

- [ ] **Correctness harness exists** and runs with multiple shapes and
      dtypes, not a single golden case.
- [ ] **Tolerance is dtype-aware**: fp16 uses a looser atol than fp32.
- [ ] **Benchmark reports a distribution**, not a single wall-clock.
- [ ] **Benchmark warms up** the GPU before timing.
- [ ] **`torch.cuda.synchronize()` is called** both before timing starts
      and before it stops.
- [ ] **Decision to write a custom kernel is justified** against a
      library baseline, with numbers.
- [ ] **Shapes benchmarked match production shapes**, not just powers
      of two.
- [ ] **Autotune configs are listed** and `key=` reflects the dims that
      actually change optimal tiling.
- [ ] **Numerical-stability choices are explicit** (subtract max before
      exp in softmax; fp32 accumulation for bandwidth-bound ops).
- [ ] **Write-up names the roofline ceiling** for the op and the % of
      peak the kernel achieves; if the kernel is within ~10% of the
      ceiling, the write-up says "further work is wasted."
- [ ] **No PTX inline** unless the write-up justifies it against the
      module-level trade-off list.

## 5. Common mistakes

1. **Timing with a single `time.time()` call and no sync.** The kernel
   launch is asynchronous; without `cuda.synchronize()` you are timing
   the launch, not the work.
2. **Only one shape benchmarked.** A kernel that wins at `(4096, 4096)`
   can lose at `(32, 127)`. Report multiple shapes, including
   non-power-of-two.
3. **Only one dtype benchmarked.** fp16 and fp32 can pick different
   optimal tilings; confirm both.
4. **Subtracting `max` after exp, not before.** Silent overflow for
   large logits; the test suite will catch it only if it has a large-
   logit case.
5. **`BLOCK` not a compile-time constant.** Triton requires `constexpr`
   block dims for reductions; failing to pass as `BLOCK=...` yields a
   confusing compile error.
6. **Beating the library on one shape and claiming victory.** The
   module-level SOLUTION lists this as an interview-grade mistake.
7. **No roofline calculation.** Without one, "it is faster" is not a
   decision — it is a coincidence that may regress on the next GPU.
8. **Writing a custom kernel at all** when the end-to-end profile shows
   the op is <5% of latency.

## 6. References

Official project documentation:

- NVIDIA CUDA C++ Programming Guide — https://docs.nvidia.com/cuda/cuda-c-programming-guide/
- NVIDIA CUDA Best Practices Guide — https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/
- NVIDIA Nsight Compute — https://docs.nvidia.com/nsight-compute/
- Triton language docs — https://triton-lang.org/
- PyTorch docs — https://pytorch.org/docs/stable/
- cuBLAS — https://docs.nvidia.com/cuda/cublas/
- CUTLASS — https://github.com/NVIDIA/cutlass
- CUB — https://nvidia.github.io/cccl/cub/

Local context:

- Module-level framing: [`../SOLUTION.md`](../SOLUTION.md)
- Companion engineer-tier module: `engineer-solutions/mod-107`
  (the module README lists it as the working implementation reference)
