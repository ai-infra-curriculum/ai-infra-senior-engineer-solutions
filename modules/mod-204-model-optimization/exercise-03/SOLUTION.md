# exercise-03 — Lab 03: LLM Deployment with vLLM

Reference solution for the learning lab
`lessons/mod-204-model-optimization/exercises/lab-03-llm-vllm.md`.

## 1. Solution overview

The lab asks the learner to stand up an LLM behind a production-shaped
inference server using vLLM, and to make deliberate choices about
batching, KV-cache budget, parallelism, and quantized loading. The
senior-level bar is that the deployment is *tuned to a target SLO* and
the tuning choices are explained.

The reference solution has four pieces:

1. A vLLM OpenAI-compatible server running a chosen model.
2. A load profile (concurrent clients, prompt / completion lengths)
   that mirrors a realistic workload.
3. A tuning sweep over the knobs that matter: `--max-num-seqs`,
   `--max-num-batched-tokens`, `--gpu-memory-utilization`,
   `--tensor-parallel-size`, and the quantization flag.
4. A short write-up explaining the chosen operating point — why that
   batch size, why that TP degree, why that quant — and what the SLO
   is.

## 2. Worked implementation

### 2.1 Serve the model

The reference starts with the stock OpenAI-compatible server. Do not
roll a custom HTTP layer for this lab — the serving layer isn't what
is being graded.

```bash
# server.sh
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3-8B-Instruct \
  --host 0.0.0.0 --port 8000 \
  --tensor-parallel-size 1 \
  --gpu-memory-utilization 0.90 \
  --max-num-seqs 256 \
  --max-num-batched-tokens 8192 \
  --dtype auto
```

For a quantized model, swap in an AWQ or GPTQ artifact and pass the
quantization flag (see §2.3). The tokenizer travels with the model
directory.

### 2.2 Load generator

The load generator must produce realistic prompts, not fixed strings.
Use a mix of short (QA) and long (summarization) prompts drawn from a
public eval set or synthetic prompts of calibrated length.

```python
# load.py (shape only)
import asyncio, time, random
import httpx

PROMPTS = load_prompts()        # a list of realistic prompts
TARGET_QPS = 20
DURATION_S = 60

async def one(client, prompt):
    t0 = time.perf_counter()
    r = await client.post(
        "http://localhost:8000/v1/chat/completions",
        json={
            "model": "meta-llama/Llama-3-8B-Instruct",
            "messages": [{"role": "user", "content": prompt}],
            "max_tokens": 256,
        },
        timeout=60,
    )
    r.raise_for_status()
    out = r.json()
    return {
        "ttft_ms": None,                  # vLLM streams; measure TTFT via /v1/chat/completions?stream=true
        "e2e_ms": (time.perf_counter() - t0) * 1000.0,
        "input_tokens": out["usage"]["prompt_tokens"],
        "output_tokens": out["usage"]["completion_tokens"],
    }
```

Report four numbers per run: **TTFT p50/p95**, **E2E p50/p95**,
**input tokens/sec**, **output tokens/sec**. TTFT is the user-perceived
latency; E2E matters for pipelines; tokens/sec is the cost metric.

### 2.3 Quantized loading

For an AWQ artifact (produced in exercise-02):

```bash
python -m vllm.entrypoints.openai.api_server \
  --model ./llama3-8b-awq \
  --quantization awq \
  --gpu-memory-utilization 0.90 \
  --max-num-seqs 256
```

The quant flag must match the artifact format (`awq`, `gptq`, `fp8`,
etc.). vLLM will refuse to load with the wrong flag — that is a
feature, not a bug.

### 2.4 Tuning sweep

Hold the load profile fixed. Sweep *one knob at a time*:

- `--gpu-memory-utilization`: controls how much VRAM the KV cache can
  claim. Too low → more preemption → higher TTFT variance. Too high →
  OOM on long prompts.
- `--max-num-seqs`: cap on concurrent sequences. Larger → higher
  throughput, longer TTFT. Smaller → lower TTFT, lower QPS ceiling.
- `--max-num-batched-tokens`: per-step token budget. Interacts with
  prompt length — sweep this when prompt mix changes.
- `--tensor-parallel-size`: only relevant when the model does not fit
  on a single GPU, or when the prefill is compute-bound at TP=1.

Record each sweep as one row in a results table.

### 2.5 Operating-point write-up

The write-up names:

- The SLO (e.g., *p95 TTFT ≤ 1.5 s at 20 concurrent users*).
- The chosen flag set (the specific CLI args).
- Which knob was the binding constraint.
- The accuracy / cost / latency trade-off that was made.

## 3. Validation steps

1. `server.sh` starts and `GET /v1/models` returns the configured
   model.
2. A sample `POST /v1/chat/completions` returns a coherent completion.
3. `load.py` runs for the stated duration and produces a JSONL log of
   per-request metrics.
4. At least three rows exist in the tuning-sweep table, with one knob
   varied at a time.
5. The write-up names the SLO, the operating point, and the binding
   constraint.

## 4. Rubric / review checklist

| Dimension                        | Weight | Pass condition |
|----------------------------------|--------|----------------|
| Server up, OpenAI-compatible     | 10%    | `/v1/models` responds; sample completion works |
| Realistic load profile           | 15%    | Prompt lengths and concurrency reflect a real workload |
| Metrics include TTFT             | 15%    | TTFT is measured via streaming, not inferred |
| Tuning sweep is single-variable  | 15%    | One knob changes per row; others held constant |
| Quantized variant tested         | 10%    | Same prompts, same load, documented quant flag |
| Operating point is defended      | 20%    | Writeup names the SLO and the binding constraint |
| Reproducibility                  | 15%    | Pinned vLLM version, model revision, GPU model |

Automatic fail: TTFT reported as "the full response time" (that is
E2E, not TTFT); or tuning rows where more than one knob moved.

## 5. Common mistakes

1. **Confusing TTFT with E2E latency.** TTFT is time to first token
   (the user-perceived delay). E2E is time to the final token. A
   model with low TTFT and slow decode is a different product from
   one with high TTFT and fast decode.
2. **Sweeping multiple knobs at once.** A table where
   `max-num-seqs` and `gpu-memory-utilization` both change tells you
   nothing about either.
3. **Reporting throughput at batch=1.** vLLM's continuous batching is
   the point; batch=1 numbers understate its advantage. Match the
   concurrency the SLO assumes.
4. **Running the load generator on the same GPU host.** CPU and
   network contention from the generator can poison the server's
   latency numbers.
5. **Forgetting the KV cache memory budget.** Long contexts eat
   KV cache; `--gpu-memory-utilization` and prompt-length mix
   interact. Tune for the worst-case prompt, not the median.
6. **Loading an AWQ artifact without `--quantization awq`.** The
   server may either refuse to start or silently dequantize —
   neither is what you want.
7. **Using `--enforce-eager` in production.** Disables CUDA graphs
   and tanks throughput. Only use it for debugging.

## 6. References

Official project documentation:

- vLLM documentation
  — https://docs.vllm.ai/
- vLLM OpenAI-compatible server
  — https://docs.vllm.ai/en/latest/serving/openai_compatible_server.html
- vLLM quantization overview
  — https://docs.vllm.ai/en/latest/features/quantization/index.html
- OpenAI Chat Completions API reference (for request / response shape
  the vLLM endpoint mirrors)
  — https://platform.openai.com/docs/api-reference/chat

Research-anchor references:

- Kwon et al., *Efficient Memory Management for Large Language Model
  Serving with PagedAttention* (vLLM paper)
  — https://arxiv.org/abs/2309.06180

Local module context:

- `modules/mod-204-model-optimization/exercise-02/SOLUTION.md` —
  the AWQ artifact loaded here is produced there.
- `modules/mod-204-model-optimization/exercise-04/SOLUTION.md` —
  the benchmarking harness used for operating-point selection.
