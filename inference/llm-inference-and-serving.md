# LLM Inference and Serving

[Sandboxed Execution](../security/sandboxed-execution.md) establishes where model-proposed code can run safely. This page looks at the other expensive part of an AI request: turning a prompt into model output reliably, quickly, and at a sustainable cost.

## The Problem

Calling a hosted model API hides this machinery. That is often the right choice. But predictable latency, high traffic capacity, data control, or self-hosting require an inference service.

An LLM may read thousands of prompt tokens before producing the first output token, then generate one token at a time. Each active request consumes accelerator memory. One very long prompt can delay many interactive users if the server schedules work poorly.

Inference engineering serves a chosen model so quality, latency, throughput, availability, and cost meet product needs. It does not make a weak model more capable.

## Mental Model

Think of an inference server as a shared kitchen with limited, expensive workstations.

```text
Requests arrive
      ↓
Scheduler admits work that fits in compute and memory
      ↓
Prefill: read each prompt
      ↓
Decode: generate one token at a time
      ↓
Stream or return the result, then free request memory
```

The scheduler is balancing two different user experiences:

- **Time to first token (TTFT):** how long the user waits before seeing a response begin.
- **Time per output token (TPOT):** how quickly later tokens arrive once generation has started.

More parallel work can improve total **throughput**, but make an individual request wait longer. Set explicit targets for both, rather than optimizing one tokens-per-second number.

## How It Works

1. **Load the model.** The server loads model weights onto one or more accelerators and reserves remaining memory for request state.
2. **Process the prompt.** In *prefill*, the model reads the input and computes attention state for generation. Long prompts increase TTFT and use more request memory.
3. **Keep useful request state.** The server stores the attention keys and values for processed tokens in a **KV cache**. It can then generate the next token without recomputing the entire prompt's attention state.
4. **Schedule active requests.** In *decode*, the server generates the next token for a batch, removes finished requests, and admits more work. Modern servers do this continuously rather than wait for a fixed batch to finish.
5. **Enforce limits and release state.** The server applies token, deadline, cancellation, and memory rules. A completed or cancelled request releases its cache.

```text
Request A: long prompt ── prefill ── decode ── done
Request B: short prompt ───── prefill ─ decode ─ done
                              ↑
                    scheduler changes the active batch
                    as requests arrive and finish
```

## Important Concepts

### Prefill and Decode Have Different Shapes

Prefill processes the whole prompt together and mainly affects TTFT. Long retrieved context, conversations, and tool definitions make it expensive. Decode produces output incrementally and mainly affects TPOT. A model can have good average throughput while an interactive request still waits behind a large prefill job.

Measure TTFT, TPOT, total latency, prompt and output length, queue wait, and cancellations separately. “Tokens per second” alone is not enough to size a user-facing service.

### The KV Cache Is a Memory Budget

The KV cache is request-specific memory that grows with processed tokens, unlike fixed model weights. It avoids repeating work during generation, but many concurrent long requests can exhaust accelerator memory even when the model fits.

Engines commonly store it in fixed-size blocks, reducing waste from differently sized requests. The practical implication is simple: a context-window limit is also a capacity decision. Higher input or output limits lower the concurrency a fixed machine can support.

### Continuous Batching and Admission Control

A fixed batch waits for its slowest request before the next batch begins. With **continuous batching**, the scheduler removes completed requests and adds eligible new ones at each generation step. This keeps accelerators busy when requests have different lengths.

Large prompt jobs can still interfere with decoding, and over-admission causes eviction, recomputation, or failures. Set queue and concurrency limits, reserve memory headroom, and choose policies based on service targets. Separate prefill and decode pools can protect strict targets at scale, but add complexity.

### Deployment Choices Change the Bottleneck

One replica can serve a model that fits in one accelerator. Replicas increase capacity. Splitting weights across accelerators enables larger models but adds coordination overhead.

Quantization uses fewer bits for some model values. It can reduce memory use and improve speed, but can change quality and compatibility. Validate it on the same [evals](../evaluation/evals.md) that justified the original model.

## Where It Fits

Inference is the model-serving layer below application logic. Whether a provider or application team runs it, its limits affect every feature above it.

```text
Application: RAG, tools, agents, policy
                 ↓
Request policy: model choice, token limits, deadlines
                 ↓
Inference service: queue, scheduler, model, KV cache
                 ↓
Accelerators and model replicas
                 ↓
Traces, latency, saturation, token, and cost signals
```

[Model Routing](../production/model-routing.md) chooses an eligible model and [Streaming AI Responses](../production/streaming-ai-responses.md) presents output. Inference determines whether that model can meet the promised experience. [Observability](../production/observability-for-ai-systems.md) reveals the cause of regressions.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Larger active batch | Total throughput and accelerator use | Queueing delay and slower per-request experience |
| Lower concurrency limit | More predictable latency and less memory pressure | Lower peak capacity and higher cost per served request |
| Longer context and output limits | More room for evidence and detailed answers | Higher TTFT, KV-cache use, and tail-latency risk |
| More replicas | Capacity and availability | More hardware cost; does not make one request faster |
| Multi-accelerator sharding | Lets larger models run | Inter-device communication, failure domains, and operational complexity |
| Quantized model | Lower memory use and possibly higher capacity | Potential quality or compatibility regression |
| Separate prefill and decode pools | Better control of TTFT and TPOT under heavy load | Cache transfer, extra capacity planning, and more moving parts |

## What Can Go Wrong

**Sizing by average tokens per second.** The service passes a throughput test while interactive users see poor TTFT.

**Unbounded context.** Long conversations or retrieved documents consume KV cache and push ordinary requests into a queue or out-of-memory failure.

**No admission control.** The server accepts more active work than memory permits, then evicts or recomputes work under load.

**One queue for incompatible work.** Large batch jobs delay short interactive requests that need a strict latency target.

**Treating cancellation as only a client event.** The browser stops listening while the server continues generating and holding memory.

**Blind quantization rollout.** A smaller artifact fits the hardware but breaks structured output or a language slice.

**Version drift.** Model weights, tokenizer, runtime, or serving configuration change without comparable evaluation and rollback.

**Ignoring warmup.** A new replica receives traffic before weights are loaded and health checks prove it can serve the expected request shape.

## Example

An internal knowledge assistant is self-hosted because documents cannot leave its environment. After a policy-index update, prompts become longer and TTFT rises sharply. Model quality, output length, and GPU utilization look normal.

Tracing separates queue wait, prefill, and decode. It shows that large retrieved prompts delay short questions. The team reduces the retrieved-context budget using [RAG evaluation](../evaluation/rag-evaluation.md), then adds a maximum queue wait and a low-priority path for bulk summaries. More replicas would hide the prompt-growth problem and raise cost.

## Interview Takeaways

- LLM serving has a prompt-processing prefill phase and a token-by-token decode phase; TTFT and TPOT measure different parts of the experience.
- KV cache saves repeated attention computation but grows with active context, so context limits and concurrency are linked.
- Continuous batching improves utilization by changing the active batch as requests finish, but needs admission control to protect latency and memory.
- Replicas add capacity; multi-accelerator sharding lets a larger model run but introduces communication costs.
- Benchmark and evaluate the full serving configuration with representative prompt and output lengths, not only a model or a hardware type in isolation.

## Next

Next: [Production Architecture for AI Systems](../production/production-architecture.md). It combines the handbook's components into a system with clear boundaries, control paths, and operational ownership.

## References

- [Kwon et al., *Efficient Memory Management for Large Language Model Serving with PagedAttention*](https://arxiv.org/abs/2309.06180)
- [Hugging Face: Continuous Batching from First Principles](https://huggingface.co/blog/continuous_batching)
- [Zhong et al., *DistServe: Disaggregating Prefill and Decoding for Goodput-optimized LLM Serving*](https://www.usenix.org/system/files/osdi24-zhong-yinmin.pdf)
- [NVIDIA TensorRT-LLM: Parallelism in TensorRT LLM](https://github.com/NVIDIA/TensorRT-LLM/blob/main/docs/source/features/parallel-strategy.md)
- [Leviathan, Kalman, and Matias, *Fast Inference from Transformers via Speculative Decoding*](https://arxiv.org/abs/2211.17192)
