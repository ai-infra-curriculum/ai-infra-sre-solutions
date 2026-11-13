# Exercise 04 — vLLM / SGLang / TRT-LLM Engine-Knob Cheatsheet

> Reference solution. The learner is asked to produce an operator-grade
> cheatsheet of the most important configuration knobs for the three major
> open-source / NVIDIA inference engines, with (a) what each knob controls,
> (b) how it interacts with the Four Golden Signals, and (c) what an AI SRE
> should check first when a workload is hot on that engine.
>
> This is a **literacy** exercise, not a tuning guide. The deliverable is
> "can you read a vLLM / SGLang / TRT-LLM config and know what will happen",
> not "memorize the exact best value."

## 1. Solution overview

All three engines share the same conceptual knob set — they just name things
differently and prioritize different workloads:

| Capability | vLLM | SGLang | TRT-LLM (via TensorRT-LLM / Triton) |
|---|---|---|---|
| Serving style | PagedAttention + continuous batching. | RadixAttention (prefix-cache tree) + continuous batching. | Static or in-flight batching over TRT-compiled engines. |
| Language | Python. | Python + custom runtime. | C++ runtime + Python bindings, usually behind Triton. |
| Primary strength | Throughput-first OSS LLM serving. | Latency / prefix-reuse for structured prompting (RAG, agents). | Lowest latency on NVIDIA GPUs via pre-compiled engines, quantization, speculative decoding. |

Given that convergence, the cheatsheet below is organized **by concern**
(memory, batching, concurrency, decoding, quantization, observability), with
each engine's name for the knob and the signal it moves. Numbers are omitted
intentionally — exact values depend on model size and hardware.

> Knob names below are current to the publicly documented interfaces of each
> engine. Engines evolve fast; always cross-check against the version you are
> actually running — flag names reflect documented interfaces as of authoring
> and should be reconfirmed against the exact engine versions used in the
> lesson lab.

## 2. Implementation — the cheatsheet

### 2.1 Memory and KV-cache

Memory sizing is the first knob to get right; everything downstream (batch
size, concurrency, prefix caching) is bounded by it.

| Concern | vLLM knob | SGLang knob | TRT-LLM knob | Signal it moves |
|---|---|---|---|---|
| Fraction of GPU memory used for KV cache | `--gpu-memory-utilization` | `--mem-fraction-static` | `kv_cache_free_gpu_mem_fraction` (runtime config) | Saturation (KV usage %); indirectly Latency (if too low, concurrency drops). |
| Explicit KV block size | `--block-size` | n/a (radix-tree managed) | `tokens_per_block` (build / runtime) | Memory fragmentation and TPOT. |
| Max tokens in a prompt | `--max-model-len` | `--context-length` | `max_input_len` (build-time) | Error rate (OOM / 4xx) and memory headroom. |
| CPU / NVMe swap of KV | `--swap-space` (CPU) | n/a (prefix tree lives on GPU by default) | not supported in the same shape | Latency spikes on eviction; Errors under pressure. |
| Prefix caching | on by default; `--enable-prefix-caching` | core feature (RadixAttention) | prefix-reuse via `enable_kv_cache_reuse` | Latency (TTFT drop on cache hit); Saturation (reduces KV churn). |

**Operator reading:**
- On vLLM / SGLang, under-sizing `gpu-memory-utilization` / `mem-fraction-static`
  caps throughput; over-sizing causes OOMs under bursty prompts. Watch the
  engine's KV-usage metric (vLLM: `vllm:gpu_cache_usage_perc`).
- On TRT-LLM, `max_input_len` is **build-time**; changing it means rebuilding
  the engine. Capacity is less elastic than vLLM/SGLang.
- Prefix caching changes incident shape — a cache-eviction storm looks like a
  TTFT spike without any obvious upstream change.

### 2.2 Batching and scheduling

| Concern | vLLM knob | SGLang knob | TRT-LLM knob | Signal it moves |
|---|---|---|---|---|
| Max concurrent sequences in a batch | `--max-num-seqs` | `--max-running-requests` | `max_batch_size` (build) + `max_num_tokens` (runtime) | Throughput ↑, p99 Latency ↑ as it grows. |
| Max tokens per batch step | `--max-num-batched-tokens` | `--max-total-tokens` | `max_num_tokens` | Throughput ↑ at the knee; above knee Latency explodes. |
| Prefill/decode scheduling | `--enable-chunked-prefill` | `--chunked-prefill-size` | in-flight batching (`enable_inflight_batching`) | ITL / TPOT stability when long prompts arrive mid-stream. |
| Scheduler policy | default FCFS | default FCFS, priority via `--schedule-policy` | request-level priorities via Triton config | Fairness and tail latency. |

**Operator reading:**
- The "knee" is the batch size at which the GPU is fully utilized. Below it,
  adding batch raises throughput without raising per-request latency. Above
  it, throughput plateaus but p99 latency climbs fast — this is the bimodal
  curve the SLO has to protect against.
- **Chunked prefill** is the single most impactful knob for mixed-length
  workloads (a long summarization next to short chat turns): without it, the
  long prefill blocks the batch; with it, short turns keep making progress.
- TRT-LLM's in-flight batching is the equivalent — build-time flag.

### 2.3 Concurrency, tensor parallelism, pipeline parallelism

| Concern | vLLM knob | SGLang knob | TRT-LLM knob | Signal it moves |
|---|---|---|---|---|
| Tensor parallelism (intra-node) | `--tensor-parallel-size` | `--tp-size` | `tensor_parallel_size` (build) | Latency ↓ for bigger models; Errors on NCCL faults. |
| Pipeline parallelism (inter-node) | `--pipeline-parallel-size` | `--pp-size` | `pipeline_parallel_size` (build) | Scales to models too big for one node; introduces bubble latency. |
| Expert parallel (MoE) | `--enable-expert-parallel` | MoE-specific flags | TRT-LLM MoE plugins | Throughput on MoE models; routing-induced tail latency. |

**Operator reading:**
- TP < model fits on one GPU: unused. TP = 2/4/8 up to the # GPUs on a node;
  beyond the node boundary use PP (interconnect bandwidth matters a lot).
- NCCL / collective timeouts are a *paging* failure class — be prepared.
- Mismatched TP between build and runtime on TRT-LLM = immediate load failure.

### 2.4 Decoding and sampling

| Concern | vLLM knob | SGLang knob | TRT-LLM knob | Signal it moves |
|---|---|---|---|---|
| Speculative decoding | `--speculative-model` + related | `--speculative-algorithm` (EAGLE/Medusa variants) | `speculative_decoding` config | Latency ↓ for compatible workloads; Quality risk if draft diverges. |
| Guided decoding / structured output | `--guided-decoding-backend` | SGLang's structured generation | logits post-processor via custom plugin | Quality ↑ (schema compliance); Latency ↑ on strict modes. |
| Max output tokens | per-request `max_tokens`; server-side cap configurable | per-request; server-side defaults | runtime param | Cost + Latency. |
| Beam search | supported but discouraged for prod | supported | supported, uncommon in prod | Latency ↑ a lot; usually not worth it for serving. |

**Operator reading:**
- Speculative decoding is a Latency lever, not a free one — if the draft
  model quality is low the acceptance rate drops and you've paid for a GPU
  with no latency gain.
- Guided decoding (JSON / regex) stabilizes output shape; good for agents
  and tool-call APIs. Verify it matches the schema the app actually parses.

### 2.5 Quantization and precision

| Concern | vLLM knob | SGLang knob | TRT-LLM knob | Signal it moves |
|---|---|---|---|---|
| Precision | `--dtype` (fp16, bf16, fp8 where supported) | `--dtype` | build-time `--dtype` + per-op precision | Throughput ↑ on lower precision, Quality risk. |
| Quantization format | `--quantization` (awq, gptq, fp8, int8 variants) | per-model quantization flag | TRT-LLM quantization toolkit at build | Memory ↓, Latency ↓, Quality must be re-evaluated. |

**Operator reading:**
- Every quantization decision is a *quality decision*. Changing the quant
  format without re-running eval is how quality burns happen. Treat quant
  changes like model promotions — go through the error-budget policy.

### 2.6 Observability

| Concern | vLLM knob | SGLang knob | TRT-LLM knob | Signal it moves |
|---|---|---|---|---|
| Prometheus metrics endpoint | on by default at `/metrics` | on by default | via Triton metrics (`:8002/metrics`) | enables TTFT / TPOT / queue / cache SLIs. |
| Per-request tracing | OpenTelemetry integration supported | logs + traces | Triton tracing | enables request-level debugging. |
| Request logging | `--disable-log-requests` to turn off | log level flags | Triton log level | privacy / volume trade. |

**Must-have histograms** (for the SRE's SLI set):
- TTFT histogram: `vllm:time_to_first_token_seconds` (vLLM), engine-exposed
  equivalents on SGLang / TRT-LLM.
- TPOT / ITL histogram: `vllm:time_per_output_token_seconds` (vLLM).
- Queue depth / waiting requests: `vllm:num_requests_waiting`.
- KV-cache utilization: `vllm:gpu_cache_usage_perc`.
- Prefix-cache hit rate: engine-specific.

Without these, the SLI catalog in Exercise 01 has no measurement source.

### 2.7 First-10-minutes operator checklist (per engine)

When a workload is paging or degrading, this is the order to walk the knobs.

**On vLLM:**
1. `vllm:num_requests_waiting` — is the queue growing? (Saturation signal.)
2. `vllm:gpu_cache_usage_perc` — is KV near full? (Memory pressure.)
3. TTFT vs. TPOT histograms — which one is moving? TTFT shift ⇒ prefill /
   admission; TPOT shift ⇒ batch / contention.
4. Any recent config change to `--max-num-batched-tokens`,
   `--gpu-memory-utilization`, `--enable-chunked-prefill`, or
   `--enable-prefix-caching`? Those are high-impact.
5. Model / weight version change? Prompt template change? (Points back at
   the error-budget policy quality-burn clause.)

**On SGLang:**
1. Requests waiting / running counts from the engine.
2. `mem-fraction-static` healthy? RadixAttention cache hit rate — a cliff in
   hit rate drives TTFT.
3. Chunked-prefill size vs. incoming prompt length distribution.
4. Any schedule-policy change?

**On TRT-LLM / Triton:**
1. Triton metrics (`:8002/metrics`) — batch sizes, queue, compute.
2. Was the engine rebuilt recently? Build-time params (max_input_len,
   tensor_parallel_size, quantization) are not hot-tunable; verify the
   deployed engine matches the config you think is running.
3. In-flight batching enabled?
4. NCCL / collective errors in the model-server logs?

### 2.8 Decision rationale

- **Organize by concern, not by engine.** The cross-engine table is the
  literacy outcome — once a learner sees that "KV memory fraction" is the
  same knob everywhere just named differently, the fear goes away.
- **Separate build-time from runtime knobs.** This is the single most
  surprising thing about TRT-LLM for operators coming from vLLM / SGLang.
  Missing it leads to "I changed the config, why didn't anything change?"
- **Link each knob to a golden signal.** The cheatsheet exists to support
  Exercise 01's SLIs and Exercise 02's policy actions. A knob with no signal
  linkage is just trivia.
- **Treat quantization and sampling as *quality* changes.** They move
  latency, yes, but any operator who doesn't re-evaluate quality after a
  quant change will cause a quality burn.
- **End with an operator checklist.** Cheatsheet without triage ordering is
  ten pages of flags. The 10-minute checklist is what gets taped above a
  monitor.

## 3. Validation steps

1. **All three engines are covered for every concern.** If a column is
   empty or "n/a" the learner has to explicitly explain why.
2. **Each row names a golden signal.** No knob is listed without its
   latency / throughput / saturation / error impact.
3. **Build-time vs. runtime is called out for TRT-LLM.** Specifically:
   `max_input_len`, `tensor_parallel_size`, quantization are build-time.
4. **Observability metrics are named.** TTFT histogram, queue, KV
   utilization at minimum — these map to the SLIs in Exercise 01.
5. **Operator checklist exists.** A triage order per engine — not just a
   grab-bag of knobs.
6. **Quantization treated as a quality decision.** Changing quant without
   re-eval is explicitly called a quality-burn risk.

## 4. Rubric / review checklist

Score each row independently; passing bar is **12/20** with no row at 0.

| Row | 0 | 2 | 4 |
|---|---|---|---|
| **Cross-engine coverage** | One engine only. | Two engines. | All three, with knob-equivalence noted. |
| **Signal linkage** | Flags listed without effect. | Partial. | Every flag ties to latency / throughput / saturation / errors. |
| **Build vs. runtime** | Not addressed. | Mentioned. | TRT-LLM build-time flags explicitly separated. |
| **Observability names** | None. | Prometheus endpoint only. | TTFT, TPOT, queue, KV usage, prefix hit (if applicable) named per engine. |
| **Operator checklist** | Missing. | Generic. | Per-engine 10-minute checklist. |

## 5. Common mistakes

- **Treating the three engines as interchangeable.** They aren't —
  SGLang's radix cache, vLLM's paged KV, and TRT-LLM's compiled engine are
  *different memory models*. The learner who writes "they all batch the
  same way" has missed the point.
- **Confusing build-time and runtime on TRT-LLM.** This produces the
  hardest-to-debug outage of the three: a config was changed, the engine
  reports "running", nothing actually changed because the engine binary is
  from the last build.
- **Memorizing numbers.** The exercise is about *reading* configs, not
  tuning. Learners who hard-code "set gpu-memory-utilization to 0.9" have
  learned trivia, not literacy.
- **Omitting metrics.** A cheatsheet without the histogram names is
  useless at 3 a.m.
- **Treating quantization as a latency knob only.** It is also a quality
  knob. Any changeover without re-eval will burn the quality SLO.
- **Forgetting speculative decoding's failure mode.** Low draft acceptance
  means all cost, no latency benefit — and the operator won't see it
  unless they check acceptance rate specifically.
- **Not linking back to the SLIs / policy.** The cheatsheet exists to let
  on-call act. If the names in this cheatsheet don't match the levers in
  Exercise 02, the operator has to translate at 3 a.m.

## 6. References

- vLLM documentation — <https://docs.vllm.ai/>. Serving flags:
  <https://docs.vllm.ai/en/latest/serving/openai_compatible_server.html>.
  Production metrics (histogram names used above):
  <https://docs.vllm.ai/en/latest/serving/metrics.html>.
- SGLang project documentation — <https://docs.sglang.ai/>.
  RadixAttention paper (Zheng et al., 2023), "SGLang: Efficient Execution of
  Structured Language Model Programs" — <https://arxiv.org/abs/2312.07104>.
- NVIDIA TensorRT-LLM documentation —
  <https://nvidia.github.io/TensorRT-LLM/>. In-flight batching,
  KV-cache reuse: `enable_kv_cache_reuse`,
  `kv_cache_free_gpu_mem_fraction`.
- NVIDIA Triton Inference Server documentation —
  <https://docs.nvidia.com/deeplearning/triton-inference-server/>.
- Google SRE Book, Ch. 6 "Monitoring Distributed Systems" (Four Golden
  Signals) — <https://sre.google/sre-book/monitoring-distributed-systems/>.
