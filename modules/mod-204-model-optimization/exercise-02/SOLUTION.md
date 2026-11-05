# exercise-02 — Lab 02: Quantization Implementation

Reference solution for the learning lab
`lessons/mod-204-model-optimization/exercises/lab-02-quantization.md`.

## 1. Solution overview

The lab asks the learner to quantize a transformer model and report
the (speed, memory, accuracy) triple honestly. At senior level the
pass bar is not "I ran a quantization recipe" — it is *the right
recipe for this model* and *a defensible choice on the precision
ladder*.

The reference workflow follows the module decision framework
(`modules/mod-204-model-optimization/SOLUTION.md`):

1. Start with **post-training quantization (PTQ)**: static INT8 with
   representative calibration data.
2. Verify accuracy on the same evaluation set as the baseline. If the
   drop is inside budget, ship it. If not, continue.
3. Try **weight-only INT4** (AWQ for LLMs) when the model is too
   activation-sensitive for INT8. This is cheaper to run but still
   needs calibration.
4. Fall back to **quantization-aware training (QAT)** only when
   PTQ/AWQ cannot preserve accuracy and the training budget is
   available.

The exercise should demonstrate at least two of these levels, not
just one.

## 2. Worked implementation

### 2.1 Baseline measurement

Before touching the quantization code, lock down the baseline. Record
FP16 (or FP32) latency, memory footprint, and task accuracy on the
exact same hardware and dataset the quantized runs will use. Every
later number is a *delta* against this baseline — without it, the
report is unfalsifiable.

```python
# baseline.py
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

model_name = "meta-llama/Llama-3-8B-Instruct"  # or any instruct model
tok = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name, torch_dtype=torch.float16, device_map="cuda"
)

mem_gb = torch.cuda.max_memory_allocated() / 1024**3
# run eval loop here and record accuracy / perplexity on the eval set
```

### 2.2 PTQ with INT8 (weight + activation, static)

For non-LLM models (vision, small NLP), the PyTorch native PTQ flow
is the reference. It uses observers to collect activation statistics
over a calibration pass, then converts the graph.

```python
# ptq_int8.py (shape only)
import torch
from torch.ao.quantization import get_default_qconfig_mapping
from torch.ao.quantization.quantize_fx import prepare_fx, convert_fx

qconfig_mapping = get_default_qconfig_mapping("fbgemm")   # x86 server
# or get_default_qconfig_mapping("qnnpack") for ARM

model_prep = prepare_fx(model.eval(), qconfig_mapping, example_inputs=example)
for batch in calibration_loader:
    model_prep(batch)                                     # calibrate

model_int8 = convert_fx(model_prep)
torch.save(model_int8.state_dict(), "model_int8.pt")
```

For transformer blocks with activation outliers, this flow often
loses accuracy. Document the drop; do not hide it.

### 2.3 INT4 weight-only for LLMs (AWQ)

For LLMs, the production reference is **weight-only INT4 via AWQ**
(see module `SOLUTION.md` §Decision 2). AWQ is calibration-based and
preserves outlier channels at higher precision.

```python
# awq_int4.py
from awq import AutoAWQForCausalLM
from transformers import AutoTokenizer

model_path = "meta-llama/Llama-3-8B-Instruct"
quant_path = "llama3-8b-awq"

model = AutoAWQForCausalLM.from_pretrained(model_path, device_map="cuda")
tok = AutoTokenizer.from_pretrained(model_path, trust_remote_code=True)

quant_config = {
    "zero_point": True,
    "q_group_size": 128,
    "w_bit": 4,
    "version": "GEMM",
}

model.quantize(tok, quant_config=quant_config)
model.save_quantized(quant_path)
tok.save_pretrained(quant_path)
```

### 2.4 QAT fallback (sketch)

QAT is only expected when PTQ and INT4 both miss the accuracy budget.
The reference does not implement it in this lab; it documents when it
would be used, with a code pointer to a known-good recipe.

### 2.5 Evaluation protocol

For every quantized artifact:

- Measure **memory**: `torch.cuda.max_memory_allocated()` after the
  model is resident on-device.
- Measure **latency**: p50/p95 at the batch size and sequence length
  the deployment target uses.
- Measure **throughput**: tokens/sec at a batch size large enough to
  saturate the GPU, not batch=1.
- Measure **task accuracy**: on the *same* eval set the baseline
  used, with the *same* generation config (temperature, top-p).

Result table shape:

| Variant         | Mem GB | p50 ms | tok/s | Task metric |
|-----------------|--------|--------|-------|-------------|
| FP16 baseline   | ...    | ...    | ...   | ...         |
| INT8 PTQ        | ...    | ...    | ...   | Δ vs base   |
| INT4 AWQ        | ...    | ...    | ...   | Δ vs base   |

Fill in on the learner's hardware. The reference intentionally does
not pre-populate numbers — see "no fabricated metrics" policy below.

## 3. Validation steps

1. Baseline script reports a memory, a latency, and a task-metric
   number on the target GPU.
2. INT8 PTQ script runs calibration and produces a saved artifact.
3. INT4 AWQ script runs quantization and produces a save directory
   that `AutoAWQForCausalLM.from_quantized(...)` can load.
4. Result table is filled for at least baseline + one quantized
   variant, with all three columns present (no missing cells).
5. Report answers: *which variant would you deploy?* and *what was the
   accuracy cost?*

## 4. Rubric / review checklist

| Dimension                       | Weight | Pass condition |
|---------------------------------|--------|----------------|
| Baseline captured correctly     | 15%    | Memory, latency, accuracy all on the same hardware |
| PTQ run succeeds                | 15%    | Calibration data is representative; artifact loads |
| INT4 (AWQ) run succeeds         | 15%    | Calibration done; saved model loads with the same tokenizer |
| Results reported as a triple    | 20%    | (speed, memory, accuracy) for each variant |
| Precision choice defended       | 15%    | Writeup names *why* INT8 vs INT4 vs QAT for *this* model |
| No fabricated numbers           | 10%    | All numbers traceable to a reproducible script |
| Reproducibility                 | 10%    | Pinned versions of `torch`, `transformers`, `autoawq` |

## 5. Common mistakes

1. **Reporting only perplexity.** Perplexity tracks downstream task
   quality imperfectly. Always pair it with a task metric (e.g.,
   MMLU, HumanEval, or a domain eval).
2. **Calibrating on training data, evaluating on held-out — and
   celebrating.** PTQ calibration must come from the *inference*
   distribution.
3. **Measuring batch-1 throughput.** INT8 and INT4 wins show at
   larger batches; batch-1 is often memory-latency-bound and the
   quantized model looks no faster.
4. **Quantizing embeddings and lm_head at the same precision as the
   body.** These are sensitive layers; keep them at higher precision
   unless accuracy on the eval set shows no regression.
5. **Confusing weight-only INT4 with INT4 activations.** AWQ / GPTQ
   quantize weights, not activations. Don't claim "INT4 inference"
   as if every tensor is INT4.
6. **Testing on `greedy` generation and deploying with `nucleus`.**
   The accuracy delta can differ substantially across decoding
   strategies — match the deployment config.
7. **Skipping the baseline.** Every deployment decision is a delta;
   without a baseline the whole exercise is vibes.

## 6. References

Official project documentation:

- PyTorch quantization guide
  — https://pytorch.org/docs/stable/quantization.html
- PyTorch FX graph-mode PTQ tutorial
  — https://pytorch.org/tutorials/prototype/fx_graph_mode_ptq_static.html
- ONNX Runtime quantization
  — https://onnxruntime.ai/docs/performance/model-optimizations/quantization.html
- Hugging Face `transformers` quantization overview
  — https://huggingface.co/docs/transformers/main/en/quantization/overview
- NVIDIA TensorRT INT8 / FP8 calibration
  — https://docs.nvidia.com/deeplearning/tensorrt/developer-guide/index.html#enable_int8_c

Research-anchor references (methods used above):

- Lin et al., *AWQ: Activation-aware Weight Quantization for LLM
  Compression and Acceleration* — https://arxiv.org/abs/2306.00978
- Frantar et al., *GPTQ: Accurate Post-Training Quantization for
  Generative Pre-trained Transformers* — https://arxiv.org/abs/2210.17323

Local module context:

- `modules/mod-204-model-optimization/SOLUTION.md` — Decision 1
  (PTQ before QAT) and Decision 2 (AWQ over GPTQ for INT4 LLMs).
- `modules/mod-204-model-optimization/exercise-01/SOLUTION.md` —
  TensorRT INT8 calibration, reused shape for engine builds.
