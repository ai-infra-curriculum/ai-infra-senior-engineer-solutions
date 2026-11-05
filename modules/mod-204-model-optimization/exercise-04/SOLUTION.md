# exercise-04 — Lab 04: Comprehensive Inference Benchmarking

Reference solution for the learning lab
`lessons/mod-204-model-optimization/exercises/lab-04-benchmarking.md`.

## 1. Solution overview

The lab asks the learner to build a benchmarking harness that can
compare inference engines (PyTorch eager, `torch.compile`, ONNX
Runtime, TensorRT, vLLM) *fairly*. The senior-level bar is **honest
reporting**: numbers that another engineer can reproduce, that are
not inflated by measurement artifacts, and that are presented with
the context (hardware, software versions, batch sizes) required to
interpret them.

The reference solution has three deliverables:

1. A harness (`bench/`) that runs a backend-agnostic measurement loop.
2. A results schema (JSONL) that includes every field needed to
   compare results across runs and hardware.
3. A report format (Markdown) that pairs latency with accuracy and
   calls out the operating conditions.

## 2. Worked implementation

### 2.1 Measurement contract

Every benchmark run must satisfy, in order:

1. **Fix the input.** The same batch of inputs is reused across all
   backends. Random inputs are generated once and saved to disk.
2. **Warm up.** `n_warmup` iterations (≥ 20) are run and discarded.
   This covers CUDA graph capture, kernel autotuning, and JIT caches.
3. **Synchronize.** GPU benchmarks use `torch.cuda.synchronize()` or
   CUDA events at the measurement boundary. CPU benchmarks use
   `time.perf_counter()` with no further synchronization needed.
4. **Repeat.** `n_iter` measurements (≥ 100) are taken. p50/p95/p99
   are reported, not just the mean.
5. **Pin the environment.** Every run records the GPU model, driver,
   CUDA / cuDNN / TensorRT versions, and the backend package version.

### 2.2 Harness core

```python
# bench/core.py
import time, json, platform, subprocess
from dataclasses import dataclass, asdict
from typing import Callable, Any
import numpy as np
import torch

@dataclass
class BenchResult:
    backend: str
    model: str
    batch_size: int
    seq_len: int | None
    warmup: int
    iters: int
    p50_ms: float
    p95_ms: float
    p99_ms: float
    throughput_qps: float
    gpu: str
    driver: str
    cuda: str
    pkg_versions: dict

def _gpu_info():
    out = subprocess.check_output(
        ["nvidia-smi", "--query-gpu=name,driver_version",
         "--format=csv,noheader"]
    ).decode().strip().split(", ")
    return {"name": out[0], "driver": out[1]}

def run(
    fn: Callable[[Any], Any],
    inputs,
    *,
    backend: str,
    model: str,
    batch_size: int,
    seq_len: int | None = None,
    warmup: int = 20,
    iters: int = 200,
    sync: Callable[[], None] = torch.cuda.synchronize,
    pkg_versions: dict | None = None,
) -> BenchResult:
    sync()
    for _ in range(warmup):
        fn(inputs)
    sync()

    samples = []
    for _ in range(iters):
        t0 = time.perf_counter()
        fn(inputs)
        sync()
        samples.append((time.perf_counter() - t0) * 1000.0)

    arr = np.asarray(samples)
    gpu = _gpu_info()
    return BenchResult(
        backend=backend,
        model=model,
        batch_size=batch_size,
        seq_len=seq_len,
        warmup=warmup,
        iters=iters,
        p50_ms=float(np.percentile(arr, 50)),
        p95_ms=float(np.percentile(arr, 95)),
        p99_ms=float(np.percentile(arr, 99)),
        throughput_qps=batch_size / (arr.mean() / 1000.0),
        gpu=gpu["name"],
        driver=gpu["driver"],
        cuda=torch.version.cuda or "",
        pkg_versions=pkg_versions or {},
    )

def append_jsonl(path: str, result: BenchResult) -> None:
    with open(path, "a") as f:
        f.write(json.dumps(asdict(result)) + "\n")
```

### 2.3 Backend adapters

Each adapter is a thin wrapper that returns a `fn(inputs) -> outputs`
callable and the `sync` function to use.

```python
# bench/adapters/torch_eager.py
import torch
def build(model_name):
    import torchvision
    m = getattr(torchvision.models, model_name)(weights="DEFAULT").eval().cuda()
    def fn(x): 
        with torch.inference_mode():
            return m(x)
    return fn, torch.cuda.synchronize
```

```python
# bench/adapters/torch_compile.py
import torch
def build(model_name):
    import torchvision
    m = getattr(torchvision.models, model_name)(weights="DEFAULT").eval().cuda()
    m = torch.compile(m, mode="reduce-overhead")
    def fn(x):
        with torch.inference_mode():
            return m(x)
    return fn, torch.cuda.synchronize
```

```python
# bench/adapters/onnxruntime_cuda.py
import onnxruntime as ort
import numpy as np
def build(onnx_path):
    sess = ort.InferenceSession(
        onnx_path,
        providers=["CUDAExecutionProvider", "CPUExecutionProvider"],
    )
    iname = sess.get_inputs()[0].name
    def fn(x_np):
        return sess.run(None, {iname: x_np})
    def sync(): pass   # ORT run is synchronous on return
    return fn, sync
```

```python
# bench/adapters/tensorrt.py
# Loads a prebuilt .plan file (see exercise-01) and binds host↔device
# with CUDA streams. Omitted here for brevity; use PyCUDA or
# TensorRT's Python runtime API.
```

For vLLM (exercise-03), the harness talks to the server over HTTP and
records TTFT and E2E rather than per-call latency.

### 2.4 Accuracy pairing

Every (backend, batch_size) row ships with a matching accuracy number
run through the same adapter on the same validation set.

```python
# bench/accuracy.py (shape only)
def accuracy(fn, loader, topk=(1,)):
    correct = total = 0
    for x, y in loader:
        pred = fn(x).argmax(dim=-1)
        correct += (pred == y).sum().item()
        total += y.numel()
    return correct / total
```

### 2.5 Report generator

Finally, a small script turns the JSONL log into a comparison table.
The report MUST include the operating conditions header; without it,
the numbers are unreadable.

```markdown
Hardware: NVIDIA L4, driver 535.x, CUDA 12.x
Software: torch 2.x.y, onnxruntime 1.x.y, TensorRT 10.x
Model: resnet50 (torchvision default weights), input 3×224×224

| Backend             | Batch | p50 ms | p95 ms | QPS   | Top-1 |
|---------------------|-------|--------|--------|-------|-------|
| torch.eager fp16    | 8     | ...    | ...    | ...   | ...   |
| torch.compile fp16  | 8     | ...    | ...    | ...   | ...   |
| onnxruntime cuda    | 8     | ...    | ...    | ...   | ...   |
| TensorRT fp16       | 8     | ...    | ...    | ...   | ...   |
| TensorRT int8       | 8     | ...    | ...    | ...   | ...   |
```

## 3. Validation steps

1. `python -m bench.run --backend torch_eager --model resnet50
   --batch 8` writes one JSONL row with p50/p95/p99 populated and a
   non-zero QPS.
2. Running the same command twice produces results within a tight
   band (see §5 "noise floor").
3. `python -m bench.report results.jsonl` emits a Markdown table with
   the operating-conditions header.
4. Every row in the table has a matching accuracy cell (no blanks).
5. A `env.txt` is saved alongside the JSONL with pinned package
   versions and the GPU identity.

## 4. Rubric / review checklist

| Dimension                       | Weight | Pass condition |
|---------------------------------|--------|----------------|
| Measurement contract honored    | 25%    | Warmup, sync, repeats, percentiles — all present |
| Backend adapters are fair       | 15%    | Same inputs, same batch, same precision ladder documented |
| Accuracy is paired with latency | 15%    | No row has a latency number without an accuracy number |
| Operating conditions reported   | 15%    | Hardware, driver, CUDA, package versions |
| Results are reproducible        | 10%    | `env.txt` + pinned requirements; two runs agree |
| Report is readable              | 10%    | Markdown table sorted sensibly; units in headers |
| Discussion of noise             | 10%    | Report names the p50 vs p95 gap and what it implies |

Automatic fail: a single mean reported as "the latency"; missing
operating-conditions header; identical numbers across backends
(a sign of a broken harness).

## 5. Common mistakes

1. **Measuring time without a synchronization.** CUDA is asynchronous.
   `time.perf_counter()` without `cuda.synchronize()` measures when
   the kernel was *enqueued*, not when it finished.
2. **Comparing engines across different GPUs.** A TensorRT run on an
   H100 vs a PyTorch run on an L4 is not a comparison, it is two
   unrelated numbers.
3. **Letting `torch.compile` autotune mid-measurement.** The first
   N calls include compile cost; pay for them in warmup, not in the
   timed loop.
4. **Reporting QPS computed from mean latency.** `QPS = batch / mean`
   understates tail behavior; always include percentile columns.
5. **Not pinning input batches.** If every iteration generates fresh
   random inputs, the host-side `torch.randn` call shows up in the
   timing as backend "work".
6. **Shared GPU, noisy host.** Other users on the same GPU, or a
   `top` process on the host CPU, both move the p95. Record whether
   the box was dedicated.
7. **Noise floor ignored.** Two consecutive runs that disagree by
   more than ~5% at p50 mean the harness is not yet trustworthy.

## 6. References

Official project documentation and standards:

- MLPerf Inference benchmarking methodology (reporting conventions,
  scenarios, latency targets)
  — https://mlcommons.org/en/inference-datacenter/
- NVIDIA Nsight Systems (profiling tool that validates harness
  measurements)
  — https://docs.nvidia.com/nsight-systems/
- PyTorch benchmarking recipes
  — https://pytorch.org/tutorials/recipes/recipes/benchmark.html
- ONNX Runtime performance tuning
  — https://onnxruntime.ai/docs/performance/
- NVIDIA TensorRT Developer Guide
  — https://docs.nvidia.com/deeplearning/tensorrt/developer-guide/

Local module context:

- `modules/mod-204-model-optimization/SOLUTION.md` — Decision 5
  (per-precision evaluation, always).
- `modules/mod-204-model-optimization/exercise-01/SOLUTION.md` —
  TensorRT engines consumed by the TRT adapter.
- `modules/mod-204-model-optimization/exercise-03/SOLUTION.md` —
  vLLM TTFT / E2E measurement shape reused here for the LLM backend.
