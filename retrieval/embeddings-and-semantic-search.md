# Embeddings and Semantic Search

This builds on [LLM Fundamentals](../foundations/llm-fundamentals.md). An LLM generates language. An embedding model represents the meaning of language so a system can find related content.

## The Problem

Keyword search works when the user's words match the document's words. It struggles when equivalent ideas use different language.

Imagine a help center article titled "Recover access after changing your phone." A user searches for "I cannot log in with my new device." The exact words barely overlap, but the article is relevant.

Semantic search addresses this gap. It retrieves content that is similar in meaning, not just spelling.

## Mental Model

An embedding is a list of numbers that places a piece of content in a high-dimensional map. Content with related meaning tends to land near each other on that map.

```text
"cannot log in after changing phone"  ──┐
                                         ├── nearby region: account recovery
"recover access on a new device"     ──┘

"download an invoice"                ───── different region: billing records
```

The coordinates are not individually meaningful. What matters is the relationship between vectors. A vector near another vector is a candidate for related meaning.

An embedding is not a database lookup, a fact, or a complete understanding of a document. It is a useful representation for ranking candidates.

## How It Works

Semantic search usually follows these steps:

1. Split source content into searchable units, often document chunks.
2. Use an embedding model to convert each unit into a vector.
3. Store each vector with an ID and metadata that points back to the source content.
4. Convert the user's query with the same embedding model.
5. Find the nearest stored vectors using a chosen similarity measure and return their source content.

```text
Document chunks ──> embedding model ──> vectors + metadata ──> vector index

User query      ──> embedding model ──> query vector ────────> nearest results
```

The embedding model must be compatible on both sides. Mixing vectors created by unrelated models makes the distances meaningless.

## Important Concepts

### Embedding Spaces

An embedding model learns a particular coordinate system, called an embedding space. It is trained so content that appears in similar contexts or expresses related ideas tends to receive nearby vectors.

Modern systems commonly embed a sentence, passage, image, or query rather than individual words. The unit matters. A whole 50-page document can hide the one paragraph that answers a question, while a tiny fragment may lose the surrounding meaning. Chunking is covered in detail in Data Ingestion.

Semantic similarity is useful, but it is not perfect synonym detection. A query for an invoice may return billing-policy content even when the user needed a specific invoice record. Search quality depends on the embedding model, the indexed content, the chunk boundaries, and the ranking strategy.

### Similarity Measures

Search needs a rule for deciding what "near" means. The common choices are:

| Measure | Intuition | Search direction |
| --- | --- | --- |
| Cosine similarity | Compare vector direction | Higher is more similar |
| Dot product | Multiply matching coordinates and add them | Higher is more similar |
| Euclidean distance, or L2 | Measure straight-line distance | Lower is closer |

Cosine similarity ignores vector length and compares direction. Its formula is:

```text
cosine(x, y) = (x · y) / (||x|| × ||y||)
```

When vectors are normalized to length 1, cosine similarity is their dot product. In that case, ranking by cosine, dot product, or L2 distance gives equivalent ordering. Without normalization, dot product also reflects vector length, so it is not automatically cosine similarity.

Use the metric the embedding model and index expect. Do not compare raw scores across models or treat a universal score threshold as proof of relevance.

### Nearest-Neighbor Search

Given a query vector, nearest-neighbor search returns the top `k` stored vectors according to the chosen metric.

**Exact search** compares the query with every stored vector. It has perfect recall: if the closest result exists in the index, exact search finds it. This is simple and often a good choice for small collections, batch jobs, or a correctness baseline.

At larger scale, comparing every query to every vector becomes expensive. **Approximate nearest-neighbor search**, or ANN, searches a smaller promising subset. It is faster, but it may miss a true nearest neighbor.

Recall measures this tradeoff. If an exact top-10 search has ten results and an ANN search returns nine of them, recall at 10 is 90%. Measure recall against exact results on representative queries, not only on random vectors.

### Vector Indexes and Vector Databases

A vector index is the data structure that makes nearest-neighbor search efficient. A flat index performs exact, brute-force comparison. An approximate index uses extra structure to avoid considering every vector.

A vector database is a system that manages vectors and usually also manages IDs, metadata, filtering, persistence, and an index. It can be a dedicated service, an extension to an existing database, or a search library paired with another store. The label matters less than the behavior you need: correct filtering, durable data, index operations, and acceptable latency.

### Metadata Filtering

Metadata narrows the search to content a result is allowed or expected to contain. Examples include tenant ID, document type, language, date, and user permissions.

```text
Query: "reset my password"
Filter: tenant_id = "acme" AND visibility = "employee"
```

Filtering is part of retrieval correctness, especially for access control. With approximate indexes, a filter applied only after candidate search may return too few allowed results. Highly selective filters may need a different plan, such as filtering first, partitioning the index, or searching more candidates before filtering.

## Where It Fits

Embeddings convert unstructured content into a searchable representation. Retrieval later chooses how to combine semantic search with keyword search and reranking.

```text
Source content
     ↓
Chunk and embed
     ↓
Vector index
     ↓
Query vector + metadata filter
     ↓
Candidate passages
     ↓
Retrieval pipeline or RAG context
```

The original text and metadata remain important. The vector is an indexable representation, not a replacement for the source of truth.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Exact search | Perfect recall and easy debugging | Work grows with every stored vector |
| Approximate search | Lower query latency at large scale | Some true neighbors can be missed |
| Larger `k` | More chances to include a useful candidate | More downstream work and noise |
| Smaller chunks | More precise matches | Less context per result and more vectors |
| Metadata filters | Relevance and isolation | More complex query planning; can lower ANN recall |

## Failure Modes

- **Model mismatch:** Index vectors and query vectors come from incompatible embedding models or versions.
- **Metric mismatch:** The index uses a distance rule that does not match the vectors or normalization strategy.
- **Semantic but unhelpful results:** The vectors capture a related topic rather than the exact entity, code, date, or policy the user needs.
- **Missing evidence:** The right text was never indexed, was chunked badly, or was excluded by a filter.
- **Permission leaks:** Filtering is missing or applied after unauthorized content has already entered an application path.
- **Overconfident thresholds:** A score that worked for one corpus or model is treated as a universal relevance guarantee.

## Example

A company indexes its employee help articles. Each paragraph receives an embedding plus metadata for `tenant_id`, `audience`, and article URL.

For the query "my new phone cannot receive login codes," the application embeds the query, filters to the employee's tenant and audience, and retrieves the closest paragraphs. The top result may be a paragraph about changing an MFA device even though the query never used the phrase "multi-factor authentication."

The application still needs to show the actual article and may later combine this semantic result with keyword search. It should not infer that the user is authorized to reset another person's account merely because the vectors are similar.

## Interview Takeaways

- Embeddings turn content into vectors whose relative positions support semantic ranking.
- Search the query and stored content in the same compatible embedding space and with the correct metric.
- Exact nearest-neighbor search is the quality baseline; ANN trades some recall for latency and scale.
- A vector index accelerates search, while a vector database adds data-management concerns around the index.
- Metadata filtering is both a relevance feature and an access-control requirement.

## Next

Next: Sparse, Dense, and Hybrid Retrieval. It explains why semantic search should often be combined with keyword search and reranking.

## Go Deeper

### HNSW

Hierarchical Navigable Small World, or HNSW, stores vectors in a multi-layer graph. Search starts in sparse upper layers to move quickly toward the query's region, then explores a denser lower layer for nearby candidates.

HNSW often provides a strong recall-latency tradeoff, but graph links consume additional memory and index construction can be slower. Query-time exploration can be increased to improve recall at the cost of latency.

### IVF and IVFFlat

An inverted file index, or IVF, groups vectors into clusters. At query time it identifies the closest clusters and searches only vectors assigned to those clusters. IVFFlat keeps full vectors in each selected cluster, so comparisons inside selected clusters are exact.

IVF can use less memory and build faster than HNSW, but it requires a representative dataset to train its clusters. Searching more clusters improves recall and increases latency. If every cluster is searched, IVFFlat becomes exhaustive again.

## References

- [Mikolov et al., *Distributed Representations of Words and Phrases and their Compositionality*](https://arxiv.org/abs/1310.4546)
- [Faiss: metrics and distances](https://github.com/facebookresearch/faiss/wiki/MetricType-and-distances)
- [Faiss index overview](https://github.com/facebookresearch/faiss/wiki/Faiss-indexes)
- [Malkov and Yashunin, *Efficient and Robust Approximate Nearest Neighbor Search Using Hierarchical Navigable Small World Graphs*](https://arxiv.org/abs/1603.09320)
- [pgvector documentation: exact search, HNSW, IVFFlat, and filtering](https://github.com/pgvector/pgvector)
