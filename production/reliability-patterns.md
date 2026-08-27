# Reliability Patterns for AI Systems

[Evals](../evaluation/evals.md) show whether an AI system is useful and safe under known conditions. Reliability patterns address a different question: what should the system do when a model provider, tool, queue, network, or worker fails while real work is in progress?

## The Problem

An AI application depends on remote, fallible systems. A model call can be rate limited. A search service can time out. A tool may accept an action before your worker records success.

The last case is especially dangerous. If an assistant retries a timed-out `create_refund` call, did the refund fail, or did it already happen? A system that cannot answer that question can accidentally duplicate a real-world action.

Reliability is making the system fail predictably, avoid making an outage worse, and leave work repairable.

## Mental Model

Treat every dependency call as one of three outcomes:

```text
Known success       → record result and continue
Known failure       → return, repair, or ask for help
Unknown outcome     → reconcile before repeating a side effect
```

A timeout means only that the caller stopped waiting. It does not prove the dependency did nothing. Do not start with "retry everything."

## How It Works

1. **Set a deadline.** Decide how long a request or background step may wait. Pass the remaining time downstream.
2. **Classify the failure.** A rate limit, temporary network error, malformed request, denied permission, and invalid model response need different handling. Only temporary failures are candidates for retry.
3. **Retry a bounded number of safe operations.** Wait longer between attempts and add random variation, called *jitter*, so many workers do not retry together. Stop when the deadline or retry budget is exhausted.
4. **Make writes repeat-safe.** Give each consequential action a stable operation ID or idempotency key. Record it before the call. The same key must not create a second action.
5. **Degrade or defer safely.** When repeated calls to one provider fail, stop sending it normal traffic for a short time. Return a clear degraded result, use a safe fallback, or queue work for later. Move repeatedly failing background work to a visible repair queue with its input, operation ID, attempts, and error history.

```text
Request or workflow step
          ↓  deadline
Call model, retrieval service, or tool
   ├── success  → record result
   ├── temporary failure → bounded retry with backoff
   ├── repeated failure  → fallback, queue, or fail clearly
   └── timeout after write → reconcile using operation ID
```

## Important Concepts

### Deadlines, Timeouts, and Cancellation

A **deadline** is the total time available for useful work. A **timeout** is the limit for one wait. In a chain of calls, give a retrieval service only the remaining time, not a new full timeout after the model used most of the budget.

When a request is cancelled or its deadline expires, stop downstream work when practical. Otherwise an overloaded system spends capacity producing answers or tool results that nobody can use.

Interactive and background work need different budgets. A document-indexing task can wait longer in a queue, but still needs a maximum duration and an owner for failure.

### Selective Retries

Retry transient failures such as a connection reset, short outage, or explicit rate limit. Do not retry invalid input, failed authorization, unsupported tool arguments, or a schema-validation failure without changing something first.

Use exponential backoff with jitter and a small attempt limit. Backoff reduces pressure on a struggling dependency; jitter prevents thousands of requests from returning at the same instant. Keep retries in one layer whenever possible. Otherwise they multiply latency, cost, and provider calls.

Retrying a transport error may be reasonable. Retrying a valid but unhelpful model answer is a quality decision, not an infrastructure retry. Evaluate it and give it its own budget.

### Idempotency and Unknown Outcomes

An operation is **idempotent** when repeating it has the same effect as doing it once. Reads are often naturally idempotent. Sending an email, charging a card, granting access, or creating a ticket usually is not unless the receiving system supports a key that identifies the intended operation.

For a write, generate a stable key from the durable task and action, such as `case-104:issue-refund`. Store it before calling the service. If a timeout leaves the outcome unknown, ask the service or your own operation record what happened for that key. Do not create a new key and try again.

If the target cannot deduplicate and duplication would cause harm, reconcile from authoritative records or send the case for review. No retry setting can remove this ambiguity.

### Circuit Breakers and Safe Degradation

A **circuit breaker** watches a dependency's recent failures. When failures cross a threshold, it opens: calls fail quickly rather than occupying workers until they time out. After a short pause, it allows test calls. If they succeed, normal traffic resumes.

A fallback must preserve the product's safety promise. A research assistant might return a cached answer marked as possibly stale. A scheduling assistant might save a draft and say booking is unavailable. It must not invent a confirmation or bypass approval.

### Queues, Repair, and Backpressure

Use a queue when work can happen later, such as indexing a document or generating a report. Limit its size and worker concurrency. An unlimited queue hides overload until latency, storage, or cost becomes the next failure.

After a defined number of failed attempts, move an item to a repair queue, often called a dead-letter queue. Alert on it, inspect the cause, and replay only when it remains safe and relevant.

## Where It Fits

Reliability surrounds the components already covered in this handbook. It does not decide whether an answer is grounded or a tool call is authorized. It makes the path through them survive ordinary operational failure.

```text
User request or durable workflow
              ↓
Deadline and admission control
              ↓
Retrieval ── Model ── Controlled tool
    ↓          ↓              ↓
 fallback   retry policy   idempotent action
              ↓
       record outcome or repair work
```

[Structured Outputs and Tool Calling](../tools/structured-outputs-and-tool-calling.md) keeps application actions controlled. [Long-Running and Durable AI Workflows](../workflows/long-running-and-durable-ai-workflows.md) keeps their state recoverable. Reliability patterns protect the calls between them.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Short deadline | Protects latency and capacity | May abandon work that could have finished |
| Automatic retry | Recovers transient failures | More latency, cost, and possible duplicate effects |
| Idempotency key | Safe recovery of consequential writes | Requires a durable operation record and target support |
| Circuit breaker | Prevents dependency failures from spreading | Can temporarily reject calls that might have succeeded |
| Fallback or degraded response | Keeps some user value during an outage | Lower quality or freshness; must be clearly communicated |
| Queue and repair path | Absorbs delayed work and preserves evidence | Adds operational state, monitoring, and replay discipline |

## Failure Modes

- **Retrying every error:** Invalid requests and permission denials become noisy, expensive loops.
- **Retries at several layers:** A browser, API, agent, and tool client each retry three times, multiplying one request into dozens of calls.
- **Timeout treated as failure:** The original write succeeded, but the retry creates a second refund, ticket, or permission grant.
- **No deadline propagation:** A backend spends resources on work after the original request has already failed.
- **Fallback that lies:** The application labels an action complete when it only saved a draft or returned cached information.
- **Open circuit with no recovery check:** A healthy dependency remains unnecessarily unavailable.
- **Ignored repair queue:** Repeatedly failing work is preserved but never diagnosed, repaired, or replayed.
- **Reliability confused with quality:** Retries make an unreliable model response arrive consistently, but do not make it correct, grounded, or safe.

## Example

An employee asks an IT assistant to grant a contractor access to a project. The assistant retrieves the relevant policy, obtains the required approval, and starts the provisioning step in a durable workflow.

Before calling the identity provider, the workflow stores the operation ID `access-721:grant-contractor`. The provider times out after receiving the request. Instead of granting access again, the workflow queries the provider with that ID, finds that access was created, and records the result.

If the provider continues failing, the circuit breaker opens. New requests are saved as pending rather than pretending access was granted. Cases that still fail move to the repair queue with their approval and operation IDs.

## Interview Takeaways

- A timeout creates an unknown outcome; it does not prove an external action failed.
- Retry only transient, repeat-safe operations, with bounded exponential backoff and jitter.
- Protect consequential writes with idempotency keys and durable operation records; reconcile before repeating unknown work.
- Circuit breakers, load limits, and honest fallbacks prevent one unhealthy dependency from consuming the whole system.
- Background failures need visible ownership and a repair path, not endless retries or silent loss.

## Next

Next: [Observability for AI Systems](observability-for-ai-systems.md). It makes request paths, model behavior, tool outcomes, and reliability safeguards visible enough to operate and improve the system.

## References

- [Google SRE Book: Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [AWS Prescriptive Guidance: Retry with Backoff](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/retry-backoff.html)
- [AWS Prescriptive Guidance: Circuit Breaker](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/circuit-breaker.html)
- [Amazon SQS: Using Dead-Letter Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
