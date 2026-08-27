# Index Maintenance and Synchronization

[Data Ingestion](data-ingestion.md) builds a retrieval index from a source snapshot. Real sources change, so a production system also needs to discover changes, update derived records, and prove that the index has not fallen behind.

## The Problem

Suppose an editor changes one paragraph in a 200-page handbook. The CMS or file store remains the source of truth. The sparse index, vector index, and RAG citations are derived data.

If the derived data is not updated, retrieval can return a policy the source no longer contains. If it is updated carelessly, old chunks can remain, a failed job can leave one index half updated, or an older event can overwrite a newer version. The design question is both what to recompute and how to keep serving useful results while the update happens.

## Mental Model

Treat retrieval indexes as rebuildable materialized views of source content. Synchronization is a feedback loop, not a one-time event handler.

```text
Source of truth changes
          ↓
Events, polling, or scheduled scans
          ↓
Reindex work queue
          ↓
Sparse and vector indexes
          ↓
Freshness checks and reconciliation
          └─────────────── repair or backfill
```

The usual goal is eventual consistency with a defined freshness target. A public help center may tolerate a few minutes. A permission revocation or pricing change may need a faster path.

## How It Works

1. **Detect the change.** Use a webhook, change-data-capture feed, application outbox, polling cursor, or scheduled scan. Record the document ID, source version, and operation type.
2. **Create durable work.** Put the change on a queue or job store so a worker can retry it and the system can measure what is still pending.
3. **Read authoritative state.** Fetch the current source document rather than trusting a partial client payload. Extract and normalize it using the same [Data Ingestion](data-ingestion.md) rules.
4. **Build the desired derived state.** Create the new chunk set, compare stable IDs and content hashes, and identify changed, new, and deleted chunks.
5. **Apply the update safely.** Upsert changed chunks into every retrieval index, delete chunks that no longer belong, and use the source version and a stable operation identity to make retries idempotent.
6. **Measure and repair.** Record the latest source version seen and the latest version fully indexed. Reconcile source and derived state periodically, then create repair jobs or backfills for gaps.

```text
Document version 42
          ↓
Desired chunks for version 42
          ↓
Upsert changed chunks + delete stale chunks
          ↓
Indexes represent version 42
```

## Important Concepts

### Identities, Versions, and Hashes

Use separate identities for the source document and its chunks.

```text
document_id = employee-handbook
document_version = 42
chunk_id = employee-handbook:parental-leave:2
content_hash = hash(normalized chunk content)
```

The exact chunk ID scheme is a design choice. It must identify the same logical chunk on a retry and preserve the parent-child relationship. Store the source version, ingestion version, embedding model, and chunking strategy with each record.

A content hash lets the pipeline skip re-embedding unchanged content. It does not replace document versioning because a metadata or permission change may require an update even when the text is identical.

### Incremental and Full Reindexing

Incremental reindexing updates only the affected document or chunks. It is faster and cheaper for ordinary edits, but it is safe only when the pipeline can identify all affected records. If a change shifts section boundaries, rebuild the affected section or the whole document instead of assuming one paragraph maps to one vector.

Full reindexing rebuilds all derived data. Use it when the embedding model, chunking strategy, normalization rules, index metric, or permission representation changes. These changes alter the meaning of the whole index.

### Upserts, Deletes, and Permission Changes

An upsert creates a missing chunk or replaces an existing chunk with the same stable ID. A delete removes a chunk that no longer exists, belongs to a deleted document, or has lost its searchable permission scope. Deletion is a first-class operation. Updating only new chunks leaves stale vectors behind.

For multi-index systems, apply the same source version to the sparse and vector indexes. Query-time code can detect or avoid mixed versions when strict consistency is required. Permission changes must propagate even when the text and embedding do not change.

### Freshness and Eventual Consistency

The source may show version 42 while retrieval still serves version 41. Expose that lag in monitoring using the age of pending work and the difference between source and indexed versions. Event-driven updates reduce lag, but events can be delayed, duplicated, reordered, or lost. A slower reconciliation scan provides an independent way to find missed changes.

### Reconciliation, Repair, and Backfills

Reconciliation compares desired source state with derived state. It can check document counts, source versions, content hashes, permission metadata, and samples of source documents. A repair job reprocesses an affected document. A backfill processes a larger historical set after adding a connector, changing a rule, or restoring missed events.

These are normal parts of an asynchronous system. Any pipeline that can lose connectivity or worker capacity needs a way to prove and restore completeness.

### Safe Rebuilds

For a large full reindex, build a new generation beside the current one. Validate its counts, sample retrieval quality, and permissions, then switch an alias or configuration pointer.

```text
search alias ──> index-v1

build and validate index-v2

search alias ──> index-v2
```

This blue-green replacement avoids serving a partially rebuilt index. Keep the old generation briefly for rollback, then remove it deliberately.

## Where It Fits

Index maintenance is the update path around the ingestion pipeline. Its outcome determines how current [RAG](../rag/retrieval-augmented-generation.md) answers can be.

```text
Author edits source
          ↓
Synchronization detects the change
          ↓
Reindexing updates derived chunks and indexes
          ↓
Retrieval and RAG serve the new version
```

The source remains authoritative. The derived index should be disposable and rebuildable.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Chunk-level update | Low cost for isolated edits | Harder identity and boundary logic |
| Document-level update | Simpler correctness model | More embedding work for small changes |
| Event-driven updates | Low freshness lag | Duplicate, ordering, and loss handling |
| Scheduled reconciliation | Finds missed or stale state | Extra reads and delayed repair |
| Full rebuild | Clean migration for global changes | Time, cost, and operational coordination |
| In-place update | Fewer index generations | Partial-state and rollback risk |
| Blue-green rebuild | Consistent cutover and easy rollback | Temporary duplicate storage and orchestration |

## What Can Go Wrong

**Stale or orphaned vectors.** An edit or deletion updates the new chunks but leaves old records searchable. Keep deletion explicit and reconcile derived IDs with source state.

**Partial index updates.** The sparse index and vector index reach different versions, so retrieval is inconsistent. Record versions per index and make mixed state visible or temporarily avoid it.

**Out-of-order or duplicate work.** An older event finishes after a newer one, or a retry creates duplicates. Use source versions, stable operation IDs, and idempotent writes.

**Permission drift.** Text is unchanged, but a user or tenant loses access and the metadata update never reaches the index. Treat permission changes as retrieval-affecting events.

**Unbounded freshness lag.** A queue grows while the system still reports healthy source writes. Track pending age and source-versus-indexed versions, then apply backpressure or repair.

**Configuration drift.** Query embeddings use a different model, metric, or normalization rule from the index. Version these inputs and rebuild when a global representation changes.

**No independent repair path.** A lost webhook looks like success because nothing compares source and derived state. Schedule reconciliation even when the event path appears reliable.

## Example

A CMS publishes version 42 of a regional leave policy. Its event places the document ID and version on a durable queue. A worker reads the current document, rebuilds the affected section, embeds only chunks whose content hashes changed, updates both indexes, and records the indexed version.

If the event is duplicated, the same operation identity makes the second run harmless. If it is missed while the worker is offline, a nightly reconciliation scan finds the version mismatch and creates a repair job. Search may have been temporarily stale, but the system has a defined path back to correctness.

If the company changes its chunking strategy for every handbook, it builds a new index generation, validates retrieval and permissions, and switches traffic only after the rebuild is complete.

## Interview Takeaways

- Retrieval indexes are derived data, not the source of truth.
- Stable document and chunk identities make upserts, deletes, retries, and reconciliation possible.
- Incremental updates save work, but shifted boundaries, metadata changes, and permission changes can widen the update scope.
- Event-driven updates reduce lag; reconciliation detects missed work and restores completeness.
- Full reindexing and blue-green cutovers are appropriate when a global representation rule changes.

## Next

Next: [Retrieval-Augmented Generation](../rag/retrieval-augmented-generation.md). It explains how current indexed chunks become a grounded LLM answer.

## References

- [Elasticsearch documentation: index aliases and zero-downtime reindexing](https://www.elastic.co/guide/en/elasticsearch/reference/current/aliases.html)
- [Debezium: change data capture](https://github.com/debezium/debezium)
- [Debezium PostgreSQL connector: insert, update, and delete events](https://debezium.io/documentation/reference/stable/connectors/postgresql.html)
- [Amazon Bedrock: synchronize a data source](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-sync.html)
