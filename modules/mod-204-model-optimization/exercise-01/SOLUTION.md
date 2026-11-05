# exercise-01 — Lab 01: TensorRT Optimization

Reference solution for the learning lab
`lessons/mod-204-model-optimization/exercises/lab-01-tensorrt-optimization.md`.

## 1. Solution overview

The lab asks the learner to take an existing PyTorch (or ONNX) model
and produce a TensorRT engine that is demonstrably faster on an NVIDIA
GPU while preserving accuracy within an acceptable budget. The senior-
level cut of the exercise is not "get a faster number" — it is
*justifying the conversion path*, *reporting results honestly*, and
*reasoning about when TensorRT is the wrong choice*.

A complete solution contains three parts:

1. A working PyTorch → ONNX → TensorRT pipeline with FP16 and INT8
   engine variants.
2. A benchmark harness that reports latency, throughput, and accuracy
   against the same baseline PyTorch model on the same hardware.
3. A short write-up answering: which precision wins for this workload,
   why, and under which conditions (batch size, sequence length, GPU
   generation) the answer would change.

The reference approach favors the officially supported export path
(`torch.onnx.export` → `trtexec` / TensorRT Python API) rather than
plugin-heavy custom paths, because the goal is a result the next
engineer on the team can reproduce.

## 2. Worked implementation

### 2.1 Export the model to ONNX

Use the stable `torch.onnx.export` entry point. Fix the opset to a
version TensorRT advertises support for in its release notes rather
than the latest ONNX opset. Export with dynamic axes on the batch
dimension so one engine can serve multiple batch sizes.

```python
# export_onnx.py
import torch
from torchvision.models import resnet50, ResNet50_Weights

model = resnet50(weights=ResNet50_Weights.DEFAULT).eval().cuda()
dummy = torch.randn(1, 3, 224, 224, device="cuda")

torch.onnx.export(
    model,
    dummy,
    "resnet50.onnx",
    input_names=["input"],
    output_names=["logits"],
    dynamic_axes={"input": {0: "batch"}, "logits": {0: "batch"}},
    opset_version=17,
    do_constant_folding=True,
)
```

Validate the ONNX file before touching TensorRT:

```bash
python -m onnx.checker resnet50.onnx
python -m onnxruntime.tools.check_onnx_model_mobile_usability resnet50.onnx || true
```

If ONNX cannot load the graph, TensorRT will not be able to parse it
either — fix export issues here, not later.

### 2.2 Build the TensorRT engine

Use `trtexec` as the first-pass tool. It is shipped with TensorRT, is
scriptable, and produces the same engine as the Python API for the
options the lab uses.

```bash
# FP16 engine — the default senior-level starting point on A100/H100
trtexec \
  --onnx=resnet50.onnx \
  --saveEngine=resnet50_fp16.plan \
  --fp16 \
  --minShapes=input:1x3x224x224 \
  --optShapes=input:8x3x224x224 \
  --maxShapes=input:32x3x224x224 \
  --workspace=4096 \
  --verbose
```

For INT8, you need a calibration dataset. The lab uses a 500-image
calibration set drawn from the training distribution. Build with a
`IInt8EntropyCalibrator2` implementation (Python example below) or
pass `--int8 --calib=<cache>` once a cache exists.

```python
# build_int8.py (abbreviated — full file in the lab template)
import tensorrt as trt
from calibrator import ImageNetCalibrator  # user-provided

logger = trt.Logger(trt.Logger.WARNING)
builder = trt.Builder(logger)
network = builder.create_network(
    1 << int(trt.NetworkDefinitionCreationFlag.EXPLICIT_BATCH)
)
parser = trt.OnnxParser(network, logger)
with open("resnet50.onnx", "rb") as f:
    assert parser.parse(f.read()), "ONNX parse failed"

config = builder.create_builder_config()
config.set_memory_pool_limit(trt.MemoryPoolType.WORKSPACE, 4 * 1024**3)
config.set_flag(trt.BuilderFlag.INT8)
config.int8_calibrator = ImageNetCalibrator(
    cache_file="resnet50_int8.cache",
    data_dir="/data/calibration",
    batch_size=32,
)

engine = builder.build_serialized_network(network, config)
with open("resnet50_int8.plan", "wb") as f:
    f.write(engine)
```

### 2.3 Benchmark harness

The harness runs the three engines (FP32 PyTorch baseline, FP16 TRT,
INT8 TRT) through the **same** inputs on the **same** GPU, with
warm-up, synchronization, and repeat counts.

```python
# bench.py (shape only — fill in loader and engine init)
import time
import numpy as np
import torch

def bench(fn, inputs, n_warmup=20, n_iter=200):
    torch.cuda.synchronize()
    for _ in range(n_warmup):
        fn(inputs)
    torch.cuda.synchronize()
    t = []
    for _ in range(n_iter):
        start = time.perf_counter()
        fn(inputs)
        torch.cuda.synchronize()
        t.append(time.perf_counter() - start)
    arr = np.array(t) * 1000.0
    return {
        "p50_ms": float(np.percentile(arr, 50)),
        "p95_ms": float(np.percentile(arr, 95)),
        "p99_ms": float(np.percentile(arr, 99)),
        "throughput_qps": inputs.shape[0] / (arr.mean() / 1000.0),
    }
```

Report at batch sizes {1, 8, 32} at minimum. Single-batch latency and
large-batch throughput are different optimization problems; a solution
that reports only one is incomplete.

### 2.4 Accuracy evaluation

Re-run the same validation set through each engine and report top-1
accuracy (or the task-specific metric). Pair every speedup number with
the accuracy delta. A correct report format is a table shaped like:

| Engine            | Batch | p50 ms | QPS   | Top-1 |
|-------------------|-------|--------|-------|-------|
| PyTorch FP32      | 8     | ...    | ...   | ...   |
| TensorRT FP16     | 8     | ...    | ...   | ...   |
| TensorRT INT8     | 8     | ...    | ...   | ...   |

Numeric cells are left blank intentionally: the learner must fill them
on their own hardware. Solutions pasting fabricated numbers are
rejected.

### 2.5 Decision rationale (the senior-level part)

The write-up must answer each of these in 2–4 sentences:

- **Why FP16 before INT8?** FP16 usually gives most of the speedup
  with near-zero accuracy loss and no calibration dataset. INT8 is
  only worth the calibration work and the accuracy risk when FP16 is
  not fast enough.
- **When would you *not* ship INT8 even if it is fastest?** When
  calibration drift between calibration data and production traffic
  would silently degrade accuracy; when the regulator or the SLO
  disallows a quality regression; when the model has activation
  outliers (e.g., certain transformer blocks) that INT8 cannot
  represent without major accuracy loss.
- **Why not skip ONNX and use `torch.compile` / `torch-tensorrt`
  directly?** For the reference workflow, ONNX is the portable
  artifact: it lets the serving team keep the engine build out of
  training infrastructure. Direct-import flows are valid but couple
  the serving step to the exact PyTorch and CUDA versions used at
  training time.

## 3. Validation steps

A graded solution should reproduce the following locally:

1. `python export_onnx.py` produces a `resnet50.onnx` file and
   `onnx.checker` passes.
2. `trtexec --onnx=resnet50.onnx --fp16 --saveEngine=...` completes
   and prints a non-zero throughput on the target GPU.
3. INT8 build reports a non-empty calibration cache and the engine
   file is smaller than the FP16 engine.
4. `python bench.py` produces three rows of results with p50/p95/p99
   latencies and matched accuracy on the same validation set.
5. The write-up's "decision rationale" paragraphs are present and
   answer the three questions above.

## 4. Rubric / review checklist

| Dimension                         | Weight | Pass condition |
|-----------------------------------|--------|----------------|
| Export reproducible               | 10%    | Script runs end-to-end from a clean checkout; opset is pinned |
| FP16 engine builds and runs       | 15%    | `trtexec` log shows FP16 layers and non-zero throughput |
| INT8 engine builds and runs       | 15%    | Calibrator is implemented; cache is produced; engine runs |
| Benchmark harness is honest       | 20%    | Warm-up, `torch.cuda.synchronize()`, p50/p95/p99, repeats ≥ 100 |
| Accuracy pairing                  | 15%    | Every latency number ships with its matching accuracy number |
| Decision rationale                | 15%    | Answers "why FP16 first", "when not INT8", "why ONNX" |
| Reproducibility notes             | 10%    | GPU model, driver, CUDA, TensorRT version are recorded |

An automatic fail: speedup claims without accuracy numbers, or
numbers reported without stating the hardware and TensorRT version
they were produced on.

## 5. Common mistakes

1. **Comparing a cold PyTorch run to a warmed-up TensorRT engine.**
   Both must be warmed with the same number of iterations.
2. **Measuring host-only time.** Without `torch.cuda.synchronize()`
   (or a CUDA event pair), you are timing kernel launches, not
   execution.
3. **Reporting batch-1 throughput as "the speedup".** TensorRT's
   advantage usually grows with batch size. Pick a batch that reflects
   production, not the one that looks best.
4. **Using a calibration dataset that is not representative.** INT8
   calibration on toy data produces an engine that is fast and wrong.
5. **Rebuilding the engine for every run.** Engine build is slow and
   non-deterministic across TensorRT versions; serialize once, load
   many times.
6. **Shipping across GPU generations.** A TensorRT plan built on an
   A100 is not guaranteed to run on an H100 and will not run on a
   consumer card. Rebuild per target SKU.
7. **Quantizing layers that shouldn't be.** Softmax, layernorm, and
   embedding layers are commonly left at higher precision — override
   the per-layer precision when the global INT8 pass degrades
   accuracy.

## 6. References

Official project documentation:

- NVIDIA TensorRT Developer Guide
  — https://docs.nvidia.com/deeplearning/tensorrt/developer-guide/
- NVIDIA TensorRT `trtexec` reference
  — https://docs.nvidia.com/deeplearning/tensorrt/developer-guide/index.html#trtexec
- ONNX operator and opset reference
  — https://onnx.ai/onnx/operators/
- PyTorch `torch.onnx` documentation
  — https://pytorch.org/docs/stable/onnx.html
- MLPerf Inference benchmark rules (for honest reporting conventions)
  — https://mlcommons.org/en/inference-datacenter/

Local module context:

- `modules/mod-204-model-optimization/SOLUTION.md` — module-level
  decision framework (precision ladder, pairing speedup with
  accuracy).
- `modules/mod-203-gpu-computing/SOLUTION.md` — kernel-level
  implications of reduced-precision formats.
