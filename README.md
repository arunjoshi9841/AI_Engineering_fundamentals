# AI Engineering Handbook

A concise, sequential handbook for software engineers learning how to build AI systems.

## Concepts

- [LLM Fundamentals](foundations/llm-fundamentals.md): How an LLM turns a bounded input into generated output. Covers tokens, prompts and messages, context windows, sampling, structured outputs, and the limits of probabilistic generation.
- [Embeddings and Semantic Search](retrieval/embeddings-and-semantic-search.md): How AI systems retrieve meaningfully related content. Covers embedding spaces, similarity measures, exact and approximate search, vector indexes, vector databases, and metadata filtering.
- [Sparse, Dense, and Hybrid Retrieval](retrieval/sparse-dense-and-hybrid-retrieval.md): How retrieval systems match exact terms and meaning together. Covers BM25, dense retrieval, rank fusion, reranking, and retrieval pipelines.
- [Data Ingestion](ingestion/data-ingestion.md): How source content becomes reliable retrieval data. Covers extraction, normalization, chunking, metadata, embeddings, indexing, and deduplication.
  - [Editable Content and Reindexing](ingestion/editable-content-and-reindexing.md): How derived indexes respond safely to source edits, deletes, and global representation changes.
  - [Data Synchronization](ingestion/data-synchronization.md): How event-driven updates, reconciliation, and repair keep derived retrieval data current.
- [Retrieval-Augmented Generation](rag/retrieval-augmented-generation.md): How retrieved evidence becomes a grounded LLM answer. Covers retrieval, context construction, citations, failure diagnosis, and RAG evaluation.
- [Structured Outputs and Tool Calling](tools/structured-outputs-and-tool-calling.md): How an LLM returns reliable data and proposes application actions. Covers schemas, validation, authorization, retries, idempotency, and human confirmation.
- [Agents](agents/agents.md): How a bounded LLM loop chooses actions from intermediate results. Covers state, planning, stopping conditions, handoffs, and when multi-agent systems help.
