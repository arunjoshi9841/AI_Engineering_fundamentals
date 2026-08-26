# Event-Driven AI Systems

[Data Synchronization](../ingestion/data-synchronization.md) keeps derived AI data current when source content changes. Event-driven design is how a system can learn about that change promptly and start the right work without constantly checking every source.

## The Problem

An AI system needs to react when something changes: a document is uploaded, a permission is revoked, a support email arrives, or a user provides feedback. Repeatedly asking every source whether it changed is slow, wasteful, and can still leave long gaps before work begins.

Compare the two basic approaches:

```text
Polling:
"Has something changed?"

Webhook:
"Something changed."
```

Event-driven systems turn the second message into reliable background work. They are useful only if they handle the realities of distributed delivery: duplicates, delays, failures, and changing event formats.

## Mental Model

An event is a record that something happened. A producer publishes it. One or more consumers react to it.

```text
Document edited
      ↓
Event producer
      ↓
Event delivery
      ↓
Consumers: reindex, audit, notify, evaluate
```

An event is not the source of truth. It is a signal to process a change. For important work, a consumer should use the event's document ID and version to read the authoritative current record.

## How It Works

A safe webhook-driven AI workflow often follows this path:

```text
External system
      ↓
Webhook endpoint
      ↓
Verify and validate event
      ↓
Persist event / enqueue work
      ↓
Worker processes change
      ↓
Update derived AI state
```

1. A producer emits an event with an ID, type, source, time, subject, and payload.
2. The endpoint authenticates the sender and verifies any signature before trusting the payload.
3. The application validates the event shape, records its ID, and puts durable work on a queue.
4. A worker consumes the work, fetches authoritative data when needed, and performs the action.
5. The worker records the result, retries only safe transient failures, and makes duplicate processing harmless.
6. Other consumers can receive the same event for independent work such as auditing, notifications, or evaluation.

Do not perform a large embedding job or a long agent task directly inside the webhook request. Acknowledge validated delivery quickly, then let a worker perform durable background work.

## Important Concepts

### Events, Producers, and Consumers

An event should describe a fact in the past tense: `document.updated`, `permission.changed`, or `feedback.received`. The **producer** owns creation of the event. A **consumer** handles one use of that event.

Useful event metadata includes:

```text
event_id       unique event identity
event_type     document.updated
source         cms://handbook
subject_id     handbook-42
occurred_at    2026-08-26T14:03:00Z
version        17
payload        change details
```

Use the event ID for delivery deduplication and the source version for data correctness. An event can be delivered twice with the same ID. Two different events can also describe versions 16 and 17 of the same document.

### Webhook Intake

A webhook is an HTTP endpoint called by an external producer. Treat it as a public boundary.

Authenticate the sender and verify signatures over the original request body according to the producer's protocol. Then validate the payload's schema, enforce request-size limits, and record enough information to diagnose failures. A syntactically valid event is not necessarily an authorized event.

After verification, persist the event or enqueue work before returning success. Returning success before durable acceptance can lose the event if the process crashes a moment later.

### Duplicate Delivery, Ordering, and Idempotency

Most event systems provide at-least-once delivery. A consumer can receive the same event more than once, especially after a timeout or a retry.

Make handling idempotent: processing the same event twice should have the same final effect as processing it once. Store processed event IDs in a durable table with a unique constraint, or make the downstream operation an idempotent upsert.

Do not assume global ordering. For a document, version 17 may arrive before version 16. Compare versions or timestamps from the source, and reject a change that is older than the derived state already stored. If strict order is needed, partition work by the document or entity key and still design for retries.

### Queues, Fan-Out, and Filtering

A queue decouples fast event acceptance from slower processing. It absorbs bursts, lets workers retry, and protects the webhook endpoint from long-running work.

Fan-out sends one event to several consumers. A `document.updated` event might trigger reindexing, audit logging, and a notification to a content owner. Keep those consumers independent so a failed notification does not block reindexing.

Filter events before expensive work. An embeddings worker may handle only `document.created`, `document.updated`, `document.deleted`, and `permission.changed` events for its supported sources.

### Schemas and Evolution

An event schema is a contract between producers and consumers. Version it deliberately.

Additive changes, such as a new optional field, are usually easier for existing consumers to tolerate. Breaking changes, such as changing a field's meaning or type, need a new event type or version and a migration period where both versions can be consumed.

Keep event payloads focused. Large documents belong in the source system. A small event that names the changed resource and its version is easier to deliver, replay, and evolve.

### Retries and Dead-Letter Handling

Retry only failures likely to succeed later, such as a temporary network or service error. Do not retry a malformed payload or a permission denial until something changes.

After a bounded number of failed attempts, send the event to a dead-letter queue or equivalent repair path. Preserve the event, failure reason, and attempt count. A dead letter is not completed work. It needs alerts, inspection, and a safe replay procedure.

## Where It Fits

Events start asynchronous AI work and communicate results between systems.

```text
CMS edit or external webhook
     ↓
Validated event
     ↓
Queue and workers
     ├── reindex changed content
     ├── update retrieval state
     ├── start an evaluation
     └── notify another system
```

This pattern connects to [Editable Content and Reindexing](../ingestion/editable-content-and-reindexing.md): events tell the pipeline which source change may require a derived-index update.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Polling | Works without an event API | Delayed detection and repeated reads |
| Webhook event | Fast application-level response | Public endpoint security and delivery complexity |
| Queue | Burst handling and asynchronous work | Operational state, retry, and monitoring needs |
| Fan-out | Independent reactions to one change | More consumers and contract coordination |
| Idempotent consumer | Safe duplicates and replays | Durable IDs, versions, and state design |
| Strict ordering | Predictable per-entity updates | Lower parallelism and more coordination |

## Failure Modes

- **Unverified webhook:** An attacker or malformed sender triggers expensive or unsafe AI work.
- **Lost event:** The endpoint returns success before the event is durably accepted.
- **Duplicate processing:** A retry creates duplicate chunks, notifications, or side effects.
- **Stale overwrite:** An older document version arrives late and replaces newer derived state.
- **Poison event:** One malformed or permanently failing event blocks a worker without a dead-letter path.
- **Schema break:** A producer change silently makes consumers misinterpret payload data.

## Example

A CMS publishes a handbook update. It sends a signed `document.updated` webhook with the document ID and source version. The webhook endpoint verifies the signature, stores the event ID, and queues a reindex job.

The reindex worker fetches the latest document, sees version 17, and updates only the changed derived chunks. An audit consumer records the change independently. If the CMS retries the webhook, the event ID prevents duplicate work. If a version-16 event arrives later, the worker ignores it because version 17 is already indexed.

## Interview Takeaways

- Events are signals that something happened; consumers should fetch authoritative state when correctness matters.
- Webhook endpoints must authenticate, verify signatures, validate payloads, and durably accept work before acknowledging success.
- Assume at-least-once delivery and possible reordering. Use idempotency keys and source versions.
- Queues separate fast event intake from slow AI jobs and enable retries, fan-out, and recovery.
- Event schemas are long-lived contracts that need compatible evolution and dead-letter handling.

## Next

Next: Polling, Scheduling, Webhooks, and CDC. It compares the ways an AI system can detect change and choose the right trigger for each source.

## References

- [CloudEvents specification](https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md)
- [CloudEvents primer: event schema evolution](https://github.com/cloudevents/spec/blob/main/cloudevents/primer.md)
- [Stripe: receive webhook events](https://docs.stripe.com/webhooks)
- [Amazon SQS: at-least-once delivery](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html)
