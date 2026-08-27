# AI Engineering Handbook

A concise, sequential handbook for software engineers learning how to build AI systems.

## How to Use This Handbook

This handbook is for building a mental model, refreshing concepts, and getting started. It intentionally abstracts some implementation detail so the important engineering tradeoffs stay visible.

Take time to implement these ideas yourself. A small retrieval pipeline, tool call, agent loop, or durable workflow teaches more than reading about it. Libraries can remove boilerplate, but they do not remove the need to understand state, permissions, retries, evaluation, and failure recovery.

Useful tools and libraries to explore after learning the underlying concepts include:

- **Agent and workflow orchestration:** [LangGraph](https://langchain-ai.github.io/langgraph/) and [LlamaIndex](https://docs.llamaindex.ai/)
- **Retrieval and vector search:** [Qdrant](https://qdrant.tech/documentation/) and [pgvector](https://github.com/pgvector/pgvector)
- **Long-running durable workflows:** [Temporal](https://docs.temporal.io/) and [Azure Durable Functions](https://learn.microsoft.com/en-us/azure/durable-functions/)

These are examples, not required choices. Start with the smallest approach that lets you test the concept, then adopt an abstraction when it solves a problem you can explain.

## Concepts

Read the concepts from top to bottom. Each page introduces the next layer of an AI system and links forward when a prerequisite matters. The indented pages deepen the surrounding topic without changing the main path.

- [LLM Fundamentals](foundations/llm-fundamentals.md): How an LLM turns a bounded input into generated output. Covers tokens, prompts and messages, context windows, sampling, structured outputs, and the limits of probabilistic generation.
- [Embeddings and Semantic Search](retrieval/embeddings-and-semantic-search.md): How AI systems retrieve meaningfully related content. Covers embedding spaces, similarity measures, exact and approximate search, vector indexes, vector databases, and metadata filtering.
- [Sparse, Dense, and Hybrid Retrieval](retrieval/sparse-dense-and-hybrid-retrieval.md): How retrieval systems match exact terms and meaning together. Covers BM25, dense retrieval, rank fusion, reranking, and retrieval pipelines.
- [Data Ingestion](ingestion/data-ingestion.md): How source content becomes reliable retrieval data. Covers extraction, normalization, chunking, metadata, embeddings, indexing, and deduplication.
  - [Index Maintenance and Synchronization](ingestion/index-maintenance-and-synchronization.md): How source changes, reindexing, reconciliation, and repair keep derived retrieval data current.
- [Retrieval-Augmented Generation](rag/retrieval-augmented-generation.md): How retrieved evidence becomes a grounded LLM answer. Covers retrieval, context construction, citations, failure diagnosis, and RAG evaluation.
- [Structured Outputs and Tool Calling](tools/structured-outputs-and-tool-calling.md): How an LLM returns reliable data and proposes application actions. Covers schemas, validation, authorization, retries, idempotency, and human confirmation.
- [Agents](agents/agents.md): How a bounded LLM loop chooses actions from intermediate results. Covers state, planning, stopping conditions, handoffs, and when multi-agent systems help.
- [Context Engineering and Memory](agents/context-engineering-and-memory.md): How an AI application selects a useful working set for each model call and safely manages state across tasks and sessions. Covers context budgets, compaction, short-term state, long-term memory, retrieval, and memory lifecycle.
- [Long-Running and Durable AI Workflows](workflows/long-running-and-durable-ai-workflows.md): How AI work persists safely across restarts, waits, and background processing. Covers checkpoints, idempotent side effects, approval flows, workflow state, recovery, and versioning.
- [Evals: Measuring AI Systems](evaluation/evals.md): How to measure whether an AI system is useful, safe, and improving. Covers representative cases, graders, slices, baselines, and the feedback loop from production failures.
  - [Evaluating RAG Systems](evaluation/rag-evaluation.md): How to measure evidence retrieval, context quality, grounded answers, citations, permissions, and freshness separately.
  - [Evaluating Agents](evaluation/agent-evaluation.md): How to test multi-step tool use through controlled environments, final-state checks, traces, constraints, and repeated trials.
- [Reliability Patterns for AI Systems](production/reliability-patterns.md): How AI systems handle operational failure without duplicating actions, hiding outages, or losing repairable work. Covers deadlines, selective retries, idempotency, circuit breakers, safe degradation, queues, and repair.
- [Observability for AI Systems](production/observability-for-ai-systems.md): How to see and diagnose the full path of AI work in production. Covers traces, versioned decision context, metrics and quality signals, alerts, sampling, and privacy-aware telemetry.
- [Caching for AI Systems](production/caching.md): How to reuse AI work safely without leaking data or serving stale results. Covers scoped keys, freshness, invalidation, cache layers, prompt caching, semantic-cache limits, and stampede protection.
- [Model Routing](production/model-routing.md): How to select from approved models by task requirements, quality, latency, cost, and risk. Covers eligibility constraints, rules, learned routers, cascades, failover boundaries, and evaluation.
- [Security for AI Systems](security/security-for-ai-systems.md): How to establish data and action boundaries for AI applications. Covers prompt injection, permission-aware retrieval, least-privilege tools, output handling, and adversarial security evaluation.
- [Sandboxed Execution](security/sandboxed-execution.md): How to run generated or untrusted code with limited blast radius. Covers execution contracts, isolated runtimes, capabilities, restricted egress, validated output, and cleanup.
- [LLM Inference and Serving](inference/llm-inference-and-serving.md): How models turn prompts into output at sustainable latency and cost. Covers prefill and decode, KV-cache memory, batching, serving capacity, and deployment tradeoffs.
- [Streaming AI Responses](production/streaming-ai-responses.md): How to turn incremental model generation into a safe response lifecycle with typed events, cancellation, reconnection, validated tool output, and explicit terminal states.
- [Production Architecture for AI Systems](production/production-architecture.md): How retrieval, tools, workflows, security, inference, and operations fit into one production system. Covers system paths, component ownership, authoritative data, and operational boundaries.
- [System Design Case Studies](case-studies/system-design-case-studies.md): Six scenarios that apply the handbook's concepts to grounded retrieval, safe actions, durable workflows, research tasks, sandboxed execution, and medical RAG evaluation.
