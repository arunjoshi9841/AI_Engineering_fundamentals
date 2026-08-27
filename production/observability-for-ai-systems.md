# Observability for AI Systems

[Reliability Patterns for AI Systems](reliability-patterns.md) keeps failures from becoming duplicate actions, outages, or lost work. Observability lets an engineer answer the next question: what actually happened inside this request, and is the system getting better or worse?

## The Problem

An AI request can return HTTP 200 while still failing the user. It may retrieve the wrong policy, exceed its budget, use an outdated prompt, ask an unnecessary tool question, or produce an unsupported answer. An ordinary application log that says "request completed" cannot explain which of those occurred.

The system also crosses boundaries. One request may select context, call a model twice, invoke a tool, wait for a queue worker, and continue in another process. Without a shared record, an engineer sees disconnected events rather than one story.

Observability provides enough evidence to ask both known questions, such as "is latency rising?", and unexpected questions, such as "why did this version fail only for Spanish support requests?"

## Mental Model

Treat observability as a flight recorder for a system, not a warehouse for every conversation.

```text
One user task
     ↓ trace ID
Request → retrieve → model → tool → model → response
     ↓        ↓         ↓       ↓       ↓
   shared context, timings, outcomes, versions, and safe references
     ↓
Traces explain one case; metrics reveal patterns; evals judge quality.
```

A **trace** is the connected record for one task. A **span** is one timed operation inside it, such as retrieval, a model call, or a tool call. Metrics aggregate many requests. Logs and events preserve selected details, especially errors and safety decisions.

Monitoring uses these signals to alert on known user-impacting problems. Observability also supports investigation when the question is not known in advance.

## How It Works

1. **Start one trace at the system boundary.** Assign a trace ID when a request or workflow begins. Propagate it through services, queues, callbacks, and background workers.
2. **Create spans at meaningful boundaries.** Record the start, finish, status, and parent span for retrieval, model calls, tool calls, authorization checks, and workflow steps.
3. **Attach safe context.** Include the application and prompt version, model and provider, request mode, retrieval configuration, tool name, outcome code, and token or cost usage. Use document and operation IDs instead of raw content where possible.
4. **Aggregate the signals.** Build dashboards for traffic, latency, errors, saturation, tokens, cost, retries, and tool outcomes. Break them down by the versions and request types that can explain a change.
5. **Connect production signals to quality evidence.** Record user feedback, review outcomes, and eval results using the trace or request ID when available. Use them to find cases for the [eval suite](../evaluation/evals.md), not as a substitute for a reviewed quality measure.
6. **Govern the telemetry.** Redact secrets and personal data before export, limit who can inspect traces, define retention, and sample intentionally. Capture raw prompts or outputs only when their debugging value and data policy justify it.

```text
Trace: tr_42
  ├── retrieve       120 ms  12 candidates
  ├── model          840 ms  model=v3, 1,240 input tokens
  ├── tool: lookup    90 ms  allowed, result=found
  └── model          410 ms  answer returned

Metrics: p95 latency, error rate, cost/request, tool-denial rate
```

## Important Concepts

### End-to-End Traces

Instrument only meaningful units of work. A useful trace normally has a root span for the user task and child spans for its major decisions and remote calls. Keep the same trace ID when work crosses a process boundary. For queued work, link the producer that created the job to the consumer that processes it later.

Trace names should describe stable operations, such as `retrieve_knowledge` or `provision_access`, not include user text or randomly generated IDs. This makes related requests easy to group.

An error status tells you that an operation failed technically. It does not prove that the final answer was wrong. Conversely, a successful model call does not prove that its answer was supported. Traces explain execution; [evals](../evaluation/evals.md) and reviewed outcomes assess behavior.

### Decision Context and Versioning

For an AI request, the important question is usually not just "which endpoint was slow?" It is "which system version made this decision?"

Record the identifiers needed to compare like with like:

- application, prompt, and tool-schema versions
- model, provider, decoding mode, and relevant limits
- retrieval index, embedding model, filter result, and selected document IDs
- tool name, authorization outcome, idempotency key reference, and result class
- token counts, cache status, retry count, and final stop reason

Prefer references, hashes, counts, and classifications over full prompts, retrieved passages, tool arguments, and model output. A secure internal capture of selected raw content may still be needed for an investigation, but it should be intentional, access-controlled, and short-lived.

### Metrics, Outcomes, and Alerts

Track system health with the usual signals: traffic, latency, errors, and saturation. Add AI-specific measures that affect the user or the budget: input and output tokens, cost per request, time to first streamed token, retrieval empty-result rate, tool success and denial rates, retry rate, and queue age.

Look at distributions, not only averages. A low mean latency can hide a slow tail, and a low average cost can hide a small group of requests that exhausts the budget.

Alert on actionable symptoms, such as a sustained rise in failed user tasks, provider error rate, queue age, or a cost guardrail breach. Do not page someone merely because a model produced unusual text. That may be useful for offline review, but it is not necessarily an operational incident.

### Sampling, Privacy, and Retention

Full tracing is valuable during development and can be too expensive or sensitive at high volume. Keep metrics for all requests, then sample routine traces while retaining errors, high-latency requests, safety events, and user-reported failures at a higher rate. Sampling rules should preserve enough context to investigate the traces you keep.

Telemetry can contain user messages, documents, credentials passed to tools, and model output. Treat it as production data: minimize it, redact before it leaves the service, encrypt it, restrict access, and delete it on schedule. Never assume an observability vendor, log bucket, or debugging dashboard is automatically safe for raw model content.

## Where It Fits

Observability connects the runtime system to evaluation and operations. It makes a reliability event diagnosable, and it makes an observed production failure usable as a future regression case.

```text
Request, agent, or workflow
            ↓
  Trace IDs and spans across components
            ↓
Metrics, selected logs, and safe context
      ┌─────┴─────┐
      ↓           ↓
Alerts and repair  Trace review → reviewed cases → evals
```

It should be built into [RAG](../rag/retrieval-augmented-generation.md), tool calling, agents, and durable workflows rather than added only after an incident. The application, not the model provider alone, sees the complete path and owns the privacy policy.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Detailed traces | Diagnosis of individual failures | Storage, instrumentation work, and sensitive data exposure |
| Aggregated metrics | Fast detection of system-wide changes | Cannot explain one request by itself |
| Raw content capture | Direct debugging evidence | High privacy and retention risk |
| References and hashes | Safer correlation and lower cost | May require a controlled lookup to inspect content |
| Broad sampling | Lower telemetry cost | Can miss the exact trace needed for an investigation |
| Alerts on user symptoms | Useful on-call response | Requires clear service expectations and thresholds |

## Failure Modes

- **No shared trace ID:** Model, retrieval, tool, and worker logs cannot be assembled into one request story.
- **Only infrastructure metrics:** The system looks healthy while it retrieves no evidence or gives unsupported answers.
- **Only raw transcripts:** Storage becomes expensive, searches become noisy, and sensitive content is exposed without giving useful aggregates.
- **Missing version context:** A prompt or model change causes regressions that cannot be isolated from traffic changes.
- **Average-only dashboards:** Tail latency, rare tool failures, and high-cost requests remain hidden.
- **Alert fatigue:** People are paged for every odd model response and stop trusting alerts.
- **Telemetry as an ungoverned data store:** Prompts, outputs, secrets, or tenant data remain accessible longer than the application itself permits.

## Example

A customer-support assistant starts returning unsupported cancellation-policy answers after a release. HTTP success rate and model latency are normal, so an infrastructure dashboard shows no incident.

Trace review groups the affected requests by prompt version and retrieval index. The traces show that the new index returned no policy chunk for a particular locale, while the model call still completed successfully. The team rolls back the index, creates a representative regression case, and adds an alert for a sustained rise in empty retrieval results. The trace contains document IDs and counts, not the customer's full message or the policy text.

## Interview Takeaways

- A trace explains one end-to-end task; spans describe its major operations; metrics reveal patterns across many tasks.
- Technical success is not product success. Connect runtime telemetry to evaluated or reviewed outcomes.
- Record model, prompt, retrieval, tool, and configuration versions so changes can be compared and rolled back.
- Prefer safe references and structured metadata to raw prompts and outputs; telemetry needs its own privacy and retention design.
- Alert on actionable user impact and inspect traces to explain the cause.

## Next

Next: Caching. It reduces repeated AI work, but changes freshness, correctness, and observability requirements.

## References

- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [OpenTelemetry: Traces](https://opentelemetry.io/docs/concepts/signals/traces/)
- [OpenTelemetry GenAI Semantic Conventions](https://github.com/open-telemetry/semantic-conventions-genai)
- [OpenTelemetry GenAI Model Spans](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md)
