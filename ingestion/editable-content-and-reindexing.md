# Editable Content and Reindexing

[Data Ingestion](data-ingestion.md) describes how to build a retrieval index from a static snapshot. Real content changes. This page explains how the derived chunks and indexes must change with it.

## The Problem

Suppose a user edits one paragraph in a 200-page handbook. The original CMS or file store is the source of truth. The sparse index, vector index, and RAG citations are derived data.

If the derived data is not updated, retrieval can return a policy that the source no longer contains. If it is updated carelessly, old chunks can remain, newly written chunks can duplicate them, or a failed job can leave the index half updated.

The design question is: **what should be recomputed, and how can the application keep serving correct results while it happens?**

## Mental Model

Treat the retrieval indexes as rebuildable materialized views of source content.

```text
CMS / database / files
        ↓
   source of truth
        ↓  ingestion and reindexing
Sparse index + vector index
        ↓
  derived retrieval state
```

The vector database is usually not the source of truth. It is one representation that can be discarded and rebuilt from authoritative content.

## How It Works

When a source document changes, a robust incremental flow looks like this:

1. Receive or discover a change containing the document ID, version, and operation type.
2. Read the authoritative current document, not a partial client payload.
3. Extract and normalize it using the same [Data Ingestion](data-ingestion.md) rules.
4. Build the desired new chunk set and compare it with the existing derived state using stable IDs and content hashes.
5. Upsert new or changed chunks into every retrieval index, then delete chunks that no longer belong to the document.
6. Record the completed source version and make retries safe to run again.

```text
document update
      ↓
desired chunks for source version 42
      ↓
upsert changed chunks + delete stale chunks
      ↓
indexes now represent version 42
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

The exact chunk ID scheme is a design choice. It must be deterministic enough to identify the same logical chunk on a retry and must keep a parent-child relationship. Store the source version, ingestion version, embedding model, and chunking strategy with each record.

A content hash lets the pipeline skip re-embedding unchanged content. It does not replace document versioning: a metadata or permission change may require an update even when chunk text is identical.

### Incremental and Full Reindexing

Incremental reindexing updates only the affected document or chunks. It is faster and cheaper for ordinary edits.

It is only safe when the pipeline can identify the affected derived records. If an edit changes section boundaries or inserts text near the start of a document, later chunk boundaries may shift. Rebuild the affected section or entire document rather than assuming one paragraph maps to one vector.

Full reindexing rebuilds all derived data. Use it when the embedding model, chunking strategy, normalization rules, index metric, or permission representation changes. These configuration changes alter the meaning of the whole index, not one document.

### Upserts, Deletes, and Stale Vectors

An upsert creates a missing chunk or replaces an existing chunk with the same stable ID. A delete removes a chunk that no longer exists, has lost a user's permission scope, or belongs to a deleted document.

Deletion is a first-class operation. Updating only the new chunks leaves stale vectors behind, which can create obsolete answers or expose content that should no longer be visible.

For multi-index systems, apply the same source version to the sparse and vector indexes. Query-time code can then detect or avoid mixed versions when strict consistency is needed.

### Safe Rebuilds

For a large full reindex, build a new index generation alongside the current one. Validate its document count, sample retrieval quality, and permissions, then switch an application alias or configuration pointer to the new generation.

```text
search alias ──> index-v1

build and validate index-v2

search alias ──> index-v2
```

This blue-green replacement avoids a period in which search sees only a partially rebuilt index. Keep the old generation briefly for rollback, then remove it deliberately.

## Where It Fits

Reindexing is the write path for the retrieval system. [Data Synchronization](data-synchronization.md) explains how this work is triggered, retried, reconciled, and monitored over time.

```text
Source change
     ↓
Change detection
     ↓
Reindex affected derived data
     ↓
Sparse and vector indexes
     ↓
RAG retrieval
```

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Chunk-level update | Low cost for isolated edits | Harder identity and boundary logic |
| Document-level update | Simpler correctness model | More embedding work for small changes |
| Full rebuild | Clean migration for global changes | Time, cost, and operational coordination |
| In-place update | Fewer index generations | Partial-state and rollback risk |
| Blue-green rebuild | Consistent cutover and easy rollback | Temporary duplicate storage and more orchestration |

## Failure Modes

- **Stale vectors:** Old chunks remain after an edit or deletion.
- **Partial indexing:** One index is updated but another is not, producing inconsistent retrieval.
- **Out-of-order changes:** Version 41 finishes after version 42 and overwrites newer data.
- **Permission drift:** Content text is unchanged, but an access-control change was not propagated.
- **Configuration drift:** Query embeddings use a different model or metric than the rebuilt index.
- **Unsafe incremental update:** A changed boundary leaves later chunks pointing at the wrong text.

## Example

A benefits team changes the parental-leave section of its handbook. The pipeline reads handbook version 42, rebuilds that section's chunks, embeds only chunks whose content hashes changed, upserts them into both indexes, and deletes the previous section chunks. It records that version 42 completed.

If the company instead changes its chunking strategy for every handbook, it creates a new index generation for all documents and switches traffic only after validation.

## Interview Takeaways

- Retrieval indexes are derived data, not the source of truth.
- Stable document and chunk identities make upserts, deletes, and retries possible.
- Incremental updates save work but require careful handling of shifted chunk boundaries and metadata changes.
- Full reindexing is appropriate when a global representation rule changes.
- Blue-green index replacement protects readers from partial rebuilds.

## Next

Next: [Data Synchronization](data-synchronization.md). It explains how a system detects source changes and ensures reindexing eventually completes.

## References

- [Elasticsearch documentation: index aliases and zero-downtime reindexing](https://www.elastic.co/guide/en/elasticsearch/reference/current/aliases.html)
- [Elasticsearch documentation: reindex documents](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-reindex)
- [Debezium: change data capture](https://github.com/debezium/debezium)
