# Sparse, Dense, and Hybrid Retrieval

In [Embeddings and Semantic Search](embeddings-and-semantic-search.md), an embedding placed related text near each other in a vector space. That gave us **dense retrieval**, or semantic search: a way to find a relevant passage even when it uses different words.

Semantic similarity is valuable, but it does not replace exact-term matching. This page adds **sparse retrieval**, which searches the literal terms in a query, and then combines both methods into hybrid retrieval.

## The Problem

Embeddings allow a search system to recognize that "recover account access" and "reset my login" express a related idea. But a vector search ranks semantic neighbors, not necessarily the document containing an exact string.

For `ERR_CONNECTION_RESET`, an invoice number, or a product SKU, the exact terms are the strongest signal. For "my internet connection keeps dropping," a document that says "network interruptions" may be relevant even though it shares none of the user's words.

Dense retrieval handles the second query well. Lexical, or sparse, retrieval handles the first well. No single method handles both cases reliably, so a production system often preserves exact matches while also handling paraphrase and broader meaning.

## Mental Model

The prior lesson's embedding index powers the dense reviewer. Sparse retrieval adds a separate reviewer that reads the query literally.

```text
Query: "database connection timeout"

Sparse reviewer: "I found pages containing those exact terms."
Dense reviewer:  "I found pages about an application losing its database connection."
                 ↓
             Combine candidates
                 ↓
        Optional careful second review
```

Hybrid retrieval asks both reviewers, merges their ranked lists, and may rerank the best candidates with a more precise model.

## How It Works

A typical hybrid pipeline has five stages:

1. Build the dense vector index from [Embeddings and Semantic Search](embeddings-and-semantic-search.md) and a sparse index over the same searchable units.
2. Run the query against both indexes, with the same metadata and permission filters.
3. Merge the two ranked candidate lists.
4. Optionally rerank the merged candidates with a model that reads each query-document pair together.
5. Return the best passages to search results or to a later RAG context builder.

```text
                    ┌── sparse search ──┐
Query + filters ────┤                  ├── merge ──> rerank ──> top passages
                    └── dense search  ──┘
```

The first stage optimizes for recall: include likely evidence. The reranker optimizes for precision: put the most useful evidence first.

## Important Concepts

### Sparse Retrieval and BM25

Sparse retrieval represents a document by the terms it contains. Most of the possible vocabulary has a value of zero for a given document, hence "sparse." It is lexical retrieval: it rewards overlap between query terms and document terms.

BM25 is a common sparse ranking algorithm. At a conceptual level, it gives more credit when a query term appears in a document, more weight to rare terms, and a length adjustment so long documents do not win simply because they contain more words.

Sparse retrieval is excellent for exact names, identifiers, error messages, technical terms, and fresh vocabulary that an embedding model may not represent well. It does not naturally recognize that "reset my login" and "recover account access" may be the same intent.

### Dense Retrieval

Dense retrieval is the semantic search introduced in [Embeddings and Semantic Search](embeddings-and-semantic-search.md): it embeds a query and each document into vectors, then ranks the closest vectors. It can retrieve paraphrases and conceptually related text without matching the same words.

It is useful for natural-language questions and varied wording. It can be weak when one exact token carries the meaning, such as a version number, a legal clause, a person's name, or an error code. Similar meaning is not always the same as the correct answer.

### Hybrid Retrieval and Fusion

Hybrid retrieval combines sparse and dense candidates. The simplest reliable approach is often to merge ranks rather than raw similarity scores, because BM25 and vector scores do not share a natural scale.

Reciprocal rank fusion, or RRF, gives a document credit for appearing near the top of each list:

```text
RRF(document) = Σ 1 / (constant + rank in a result list)
```

A document that ranks well in both searches rises. A document that only one method finds can still appear. The constant prevents a single rank-one result from overwhelming the merge.

Score fusion is another option, but it requires careful normalization and evaluation. Do not add BM25 and cosine values simply because both are numbers.

### Reranking

Retrievers must search thousands or millions of candidates quickly. A reranker can spend more compute on a much smaller candidate set, often tens or hundreds of passages.

A common reranker is a cross-encoder. Unlike a dense retriever, which embeds the query and document separately, it reads the query and one candidate together and produces a relevance score. This lets it notice details such as negation or whether a passage actually answers the question.

Rerankers improve ordering, not recall. If the relevant passage was absent from the initial candidate set, a reranker cannot recover it.

### Retrieval Pipelines

The useful unit is the pipeline, not any one retrieval algorithm.

```text
Query
  ↓
Apply tenant and permission filters
  ↓
Sparse and dense candidate retrieval
  ↓
Rank fusion
  ↓
Optional reranking
  ↓
Top passages for the user or LLM
```

Keep the same authorization constraints across every stage. A high-scoring passage is not useful if the user must not see it.

## Where It Fits

Hybrid retrieval is the selection layer between the keyword index, the vector index from [Embeddings and Semantic Search](embeddings-and-semantic-search.md), and a consumer such as a search page or an LLM.

```text
Source content
     ↓
Keyword index + vector index
     ↓
Hybrid retrieval pipeline
     ↓
Ranked evidence
     ↓
Search UI or RAG answer generation
```

The next topic, Data Ingestion, explains how source content becomes the chunks and metadata that these indexes need.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Sparse only | Exact-term matching, speed, simplicity | Misses paraphrases and semantic matches |
| Dense only | Meaning-based matching | Misses crucial exact terms and identifiers |
| Hybrid fusion | Candidate recall across both signals | Two indexes, merging logic, and more evaluation |
| Reranking | Top-result precision | Extra latency and model cost |
| Larger candidate pool | Chance of finding evidence | More downstream work and noise |

## What Can Go Wrong

**Exact-term blind spot.** Dense retrieval misses the relevant SKU, code, name, or version.

**Semantic near miss.** Dense retrieval finds a related topic, not the required policy, entity, or answer.

**Unsafe filters.** One retrieval branch uses weaker permission filters than the other.

**Score misuse.** Raw BM25 and vector scores are combined without calibration or a rank-based method.

**Reranking too little.** The relevant item never enters the candidate pool.

**Reranking too much.** A large candidate pool turns the precise stage into the latency bottleneck.

## Example

An internal engineering search receives: "why did checkout fail with `PAYMENT_DECLINED_42`?"

Sparse retrieval finds the runbook containing the exact code. Dense retrieval also finds an incident guide titled "handling card authorization failures." Fusion preserves both candidates. A reranker puts the exact-code runbook first because it directly explains the reported failure, while the incident guide remains useful background.

## Interview Takeaways

- Sparse retrieval is lexical and excels at exact terms; dense retrieval is semantic and excels at paraphrase.
- Hybrid retrieval exists because the two methods fail differently.
- Rank fusion is often safer than adding incompatible raw scores.
- Retrieval should prioritize candidate recall; reranking should improve the order of those candidates.
- Measure the full pipeline with realistic queries, filters, and relevance judgments.

## Next

Next: [Data Ingestion](../ingestion/data-ingestion.md). It explains how source data is extracted, chunked, enriched, embedded, and indexed for retrieval.

## References

- [Robertson and Zaragoza, *The Probabilistic Relevance Framework: BM25 and Beyond*](https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf)
- [Karpukhin et al., *Dense Passage Retrieval for Open-Domain Question Answering*](https://arxiv.org/abs/2004.04906)
- [Cormack, Clarke, and Buettcher, *Reciprocal Rank Fusion Outperforms Condorcet and Individual Rank Learning Methods*](https://cormack.uwaterloo.ca/cormack/cormacksigir09-rrf.pdf)
- [Nogueira and Cho, *Passage Re-ranking with BERT*](https://arxiv.org/abs/1901.04085)
