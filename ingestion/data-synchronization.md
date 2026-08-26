# Data Synchronization

[Editable Content and Reindexing](editable-content-and-reindexing.md) defines how one source change should update derived retrieval data. Data synchronization is the larger operational problem: continuously discover every change, process it reliably, and detect when the index has fallen behind.

## The Problem

Source systems change independently of the retrieval system. A CMS editor may publish an update, a database record may be deleted, a permission may change, or an ingestion worker may be offline at the wrong time.

Even a well-designed reindex operation is not enough if it is never triggered, receives duplicate events, or misses a delete. Derived AI data must be treated as rebuildable state that can become stale.

## Mental Model

Synchronization is a feedback loop between source-of-truth state and derived retrieval state.

```text
Source changes
     ↓
Events, polling, or scheduled scans
     ↓
Reindex work queue
     ↓
Derived indexes
     ↓
Reconciliation checks
     └─────────────── backfill or repair when needed
```

The goal is usually eventual consistency with a known freshness target, not an unrealistic promise that every source write is instantly visible in search.

## How It Works

1. **Detect changes.** Use events, change-data-capture feeds, webhooks, polling, or scheduled source scans.
2. **Create durable work.** Put a document ID, source version, and operation type on a queue or job store.
3. **Apply idempotently.** Re-running the same update must reach the same derived state without duplicates.
4. **Order by version.** Ignore or supersede an older update that arrives after a newer source version.
5. **Measure freshness.** Record the latest source version seen and the latest version fully indexed.
6. **Reconcile and repair.** Periodically compare source and derived state, backfill missed work, and remove orphaned chunks.

## Important Concepts

### Change Detection

Choose the trigger that fits the source. These are familiar integration techniques, so the important question here is how each one keeps derived retrieval data current.

| Technique | Use it when |
| --- | --- |
| Polling | The source has no event API. Compare a cursor, timestamp, or content hash. |
| Scheduled scan | You need a periodic refresh, reconciliation, or repair. |
| Webhook | A SaaS product or CMS can promptly notify your application of a change. |
| Change data capture (CDC) | A database is the source of truth and committed row changes should drive updates. |

Event-driven updates can use a source-system webhook, an application outbox, or CDC. CDC reads committed database changes and produces change events for inserts, updates, and deletes.

Events reduce freshness lag, but delivery is not a guarantee of correctness by itself. They can be delayed, duplicated, reordered, or lost before the consumer records them. Scheduled scans compare source versions or content hashes and provide a second path for finding missed changes.

For a small, low-change corpus, a periodic full scan may be enough. For frequently edited content, combine a fast event-driven path with a slower reconciliation scan.

### Eventual Consistency and Freshness

The source may show version 42 while retrieval still serves version 41. This is eventual consistency.

Define whether that lag is acceptable. A public help center may tolerate several minutes. A permission revocation or pricing change may need a faster, stronger path. Expose freshness in monitoring so the team can see the age of the oldest pending update and the difference between source and indexed versions.

### Reconciliation, Repair, and Backfills

Reconciliation compares the desired derived state with the actual state. It can check document counts, source versions, content hashes, permission metadata, and samples of source documents.

When it finds a gap, a repair job reprocesses the affected document. A backfill processes a larger historical set, for example after adding a new source connector or restoring a failed month of events.

These are normal operations, not signs that the original event path was a mistake. Any asynchronous pipeline needs a way to prove and restore completeness.

### Deletions and Permission Changes

Synchronization must handle more than new text. A deleted document requires deletion of its chunks. A renamed document may need only metadata updates. A permission change must update filterable metadata promptly, even if the text and embedding are unchanged.

Treat permission changes as retrieval-affecting events. Waiting for a later full content refresh can leak data through search or RAG in the meantime.

## Where It Fits

Synchronization operates around the [Data Ingestion](data-ingestion.md) pipeline and the reindexing write path. Its outcome determines how current [RAG](../rag/retrieval-augmented-generation.md) answers can be.

```text
Author edits source
     ↓
Synchronization detects the change
     ↓
Reindexing updates derived chunks and indexes
     ↓
Retrieval and RAG serve the new version
```

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Event-driven updates | Low freshness lag | Event infrastructure and duplicate handling |
| Scheduled refresh | Simplicity and broad coverage | Staler results and unnecessary scans |
| CDC | Reliable committed database changes | Source-specific operational setup |
| Reconciliation jobs | Detects missed or stale derived state | Extra reads and delayed repair |
| Immediate permission propagation | Reduces access-control risk | More urgent update path and coordination |

## Failure Modes

- **Lost event:** A source change never reaches the reindex worker.
- **Duplicate event:** A retry creates duplicate chunks because processing is not idempotent.
- **Out-of-order event:** An old version overwrites a newer one.
- **Backlog:** Updates wait too long and the index misses its freshness target.
- **Silent deletion failure:** A source is removed but its vectors remain searchable.
- **No reconciliation:** The system has no independent way to discover these gaps.

## Example

A CMS publishes a new regional leave policy. Its update event places the document ID and version on a durable queue. A worker reindexes the document and records the indexed version. A nightly reconciliation scan compares every published document's version with its indexed version.

If an event was missed while the worker was offline, the scan detects the version mismatch and creates a repair job. Search and RAG may have been temporarily stale, but the system has a defined path back to correctness.

## Interview Takeaways

- Synchronization keeps derived retrieval data aligned with source-of-truth data over time.
- Event delivery should be treated as at-least-once and potentially out of order, so update processing must be idempotent and version-aware.
- Event-driven updates reduce lag; scheduled reconciliation detects missed work.
- Deletions and permission changes are as important as text edits.
- Freshness lag and repair success are production metrics, not implementation details.

## Next

Return to [Retrieval-Augmented Generation](../rag/retrieval-augmented-generation.md) to see how freshness and retrieval quality affect the final answer.

## References

- [Debezium: change data capture](https://github.com/debezium/debezium)
- [Debezium PostgreSQL connector: insert, update, and delete events](https://debezium.io/documentation/reference/stable/connectors/postgresql.html)
- [Amazon Bedrock: synchronize a data source](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-sync.html)
