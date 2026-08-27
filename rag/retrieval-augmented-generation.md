# Retrieval-Augmented Generation

[Data Ingestion](../ingestion/data-ingestion.md) turns source content into searchable chunks. [Sparse, Dense, and Hybrid Retrieval](../retrieval/sparse-dense-and-hybrid-retrieval.md) selects the most relevant ones. Retrieval-augmented generation, or RAG, uses those selected chunks as evidence for an LLM response.

## The Problem

An LLM can answer many questions from its training, but it does not automatically know a company's current policies, private documentation, or a recent product change. Adding all of that information to every prompt is expensive and distracting. Fine-tuning is also a poor fit for content that changes frequently or needs citations.

RAG gives the model a small, relevant, source-backed reading packet at request time.

## Mental Model

RAG is an open-book response, not a better memory inside the model.

```text
Question: "What is the parental-leave policy?"
                  ↓
Retrieve the relevant handbook passages
                  ↓
Give those passages to the LLM with instructions
                  ↓
Answer with links to the handbook
```

The model still generates the answer probabilistically. Retrieval gives it evidence, but does not force it to use that evidence correctly.

## How It Works

A practical RAG request usually follows this path:

```text
User question
      ↓
Query processing
      ↓
Candidate retrieval
      ↓
Permission filtering and reranking
      ↓
Context construction
      ↓
LLM
      ↓
Grounded answer + sources
```

1. **Process the question.** Apply user and tenant context, then optionally rewrite the query for search.
2. **Retrieve candidates.** Search the sparse and vector indexes built during [Data Ingestion](../ingestion/data-ingestion.md).
3. **Filter and rerank.** Enforce permissions, then use the [hybrid retrieval](../retrieval/sparse-dense-and-hybrid-retrieval.md) pipeline to select the strongest passages.
4. **Construct context.** Fit the selected passages, their source details, and answer instructions into the LLM's context budget.
5. **Generate an answer.** Ask the LLM to answer from the supplied evidence and to say when the evidence is insufficient.
6. **Attach citations.** Return links or source identifiers that let the user inspect the passages behind the answer.

## Important Concepts

### Retrieval Quality Comes First

The LLM cannot ground an answer in a passage it never received. Retrieval should favor evidence that is relevant, current, permitted, and specific enough to answer the question.

This is why RAG inherits its quality from the earlier pages. Poor extraction or chunking from [Data Ingestion](../ingestion/data-ingestion.md), a weak embedding match, or a missing exact-term result can make the final answer fail before generation starts.

### Context Construction

Retrieved results are candidates, not a prompt. Context construction chooses what the LLM will actually see.

Include the smallest useful set of passages, clear separators, source identifiers, and instructions such as: "Answer only from the provided sources. If they do not answer the question, say so."

More context is not automatically better. It consumes tokens, adds latency, and can bury the best evidence among loosely related material. Context is a budget, as described in [LLM Fundamentals](../foundations/llm-fundamentals.md).

### Grounding and Citations

Grounding means the answer is based on supplied evidence rather than unsupported model recall. Citations expose that evidence to the user.

For useful citations, carry source metadata from ingestion through retrieval to generation. A chunk should have a URL, document ID, page, or section path that the application can present. Do not ask the model to invent source links.

A citation proves which source was retrieved, not that the statement is correct. The application should validate citation IDs and test whether cited passages actually support the answer.

### Retrieval Failures and Generation Failures

RAG has two distinct failure classes:

| Failure | Symptom | Likely fix |
| --- | --- | --- |
| Retrieval failure | The right evidence is absent from the context | Improve ingestion, filters, recall, query handling, or reranking |
| Generation failure | The evidence is present, but the answer is wrong or unsupported | Improve context instructions, model choice, answer format, or validation |

Diagnose these separately. Changing the prompt cannot recover a missing policy passage, and raising retrieval `k` cannot make a model faithfully summarize conflicting evidence.

## Where It Fits

RAG is a composition of the earlier concepts, not a replacement for them.

```text
Source of truth
     ↓
Data ingestion
     ↓
Sparse and vector indexes
     ↓
Hybrid retrieval
     ↓
Context for the LLM
     ↓
Answer with sources
```

For static content, this pipeline can be built once. For changing content, the retrieval indexes must be kept synchronized with their sources. See [Index Maintenance and Synchronization](../ingestion/index-maintenance-and-synchronization.md).

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| More retrieved passages | Chance of including evidence | More noise, tokens, cost, and latency |
| Reranking | Top-passage relevance | Extra request time and model cost |
| Strict answer-from-context instruction | Grounded behavior | More "I don't know" responses when evidence is incomplete |
| Citations | User trust and debuggability | Source tracking and evaluation work |
| Query rewriting or multi-query retrieval | Recall for ambiguous phrasing | Extra latency and possible query drift |

## What Can Go Wrong

**No answer in the index.** The source was not ingested, was stale, or cannot be found by the retrieval pipeline.

**Irrelevant context.** The model receives related passages that do not answer the question.

**Unsupported synthesis.** The model adds a claim that is not in the supplied sources.

**Citation mismatch.** The answer cites a real source that does not support the associated claim.

**Permission leak.** Unauthorized material enters the context or a citation reveals its existence.

**Stale answer.** Retrieval finds an old policy because the derived index has not caught up with its source.

## Example

An employee asks, "Can I carry unused vacation into next year?"

The system retrieves policy chunks, filters them to the employee's region and employment type, reranks them, and places the best two passages in the prompt with their section URLs. The LLM summarizes the carryover rule and cites the policy section.

If no passage describes the employee's region, the correct response is not a generic policy guess. The system should say that the available sources do not establish the answer.

## Interview Takeaways

- RAG provides relevant external evidence at request time; it does not update the model's weights.
- The system should distinguish retrieval failures from generation failures.
- Context construction is a relevance and token-budget decision, not a blind dump of search results.
- Citations require source metadata to survive ingestion, retrieval, and generation.
- RAG quality depends on retrieval, grounding, permissions, freshness, and evaluation together.

## Next

Next: [Structured Outputs and Tool Calling](../tools/structured-outputs-and-tool-calling.md). It explains how an LLM can safely turn its response into application data and proposed actions.

## Go Deeper

### Query Rewriting and Multi-Query Retrieval

Users often ask vague questions. A query rewrite can turn "how much time do I get?" into a search-oriented version that includes the relevant subject from the conversation. Multi-query retrieval searches several interpretations and merges their results.

Both techniques can improve recall, but they can also drift from the user's intent. Keep the original question in the final prompt and evaluate rewritten queries separately.

### Long Documents and Parent-Child Retrieval

Small child chunks improve precise retrieval. Their larger parent section gives the LLM enough surrounding context to interpret the result. A system can retrieve the child, then include its parent or a controlled neighboring window in the final context.

### Context Compression and Multi-Step Retrieval

When candidate passages are long, a smaller model or deterministic extractor can compress them before generation. For questions that need multiple facts, a system may retrieve once, inspect the gap, and retrieve again. Both patterns add latency and must preserve source attribution.

## References

- [Lewis et al., *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*](https://arxiv.org/abs/2005.11401)
- [Karpukhin et al., *Dense Passage Retrieval for Open-Domain Question Answering*](https://arxiv.org/abs/2004.04906)
- [Nogueira and Cho, *Passage Re-ranking with BERT*](https://arxiv.org/abs/1901.04085)
