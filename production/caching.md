# Caching for AI Systems

[Observability for AI Systems](observability-for-ai-systems.md) tells you where time and money go in a request. Caching can avoid repeated work, but only if returning an older result is still correct for this user and this moment.

## The Problem

AI work is expensive when the same work happens repeatedly. Many users may ask for the same policy, and long instructions may be sent to a model again and again.

The tempting fix is to save a result and reuse it. But an AI result may depend on the user, permissions, prompt version, model, documents, tool data, and time. Different conditions can leak data, give stale advice, or skip a required action.

Caching is a controlled promise: for a defined scope and time, an earlier computation is safe enough to reuse.

## Mental Model

A cache is a short-lived copy placed in front of expensive work.

```text
Incoming request
       ↓
Build the cache key from everything that can change the result
       ↓
Cache hit? ── yes ──→ verify it is still allowed and fresh → reuse
       │
       no
       ↓
Do the original work → store a bounded result → return it
```

The hard part is defining **equivalence**: when two requests may receive the same result.

For a public, versioned FAQ answer, it might be the question, language, answer version, and policy version. For "What is my remaining leave balance?", the user and current balance are part of the result. A shared answer cache is unsafe.

## How It Works

1. **Choose the work to reuse.** Cache a stable retrieval result, tool read, final answer, or repeated prompt prefix.
2. **Define a complete key and scope.** Include inputs that affect the result: tenant, authorization, language, versions, tool parameters, and relevant time. Use stable identifiers or hashes, not raw secrets.
3. **Set a freshness rule.** Give the entry a time-to-live (TTL), invalidate it when its source changes, or require validation before reuse. A TTL is an upper bound on acceptable staleness, not proof that data stayed correct.
4. **Read, then fill on a miss.** In *cache-aside*, the application checks the cache, calls the dependency on a miss, and stores the result. Coalesce identical misses so one request refreshes it.
5. **Measure the result.** Record hits, misses, stale responses, and avoided model tokens or tool calls. A high hit rate is not success if answers are outdated.

## Important Concepts

### Exact Result Caches

An exact cache reuses a result only when the key matches exactly. It is the safest response cache because its equivalence rule is explicit. Use it for public, read-only requests, such as translating a fixed release note.

Include model, prompt, document, or retrieval versions when they can change the answer. The source can still change after the entry is created.

Avoid caching results for side effects. A cache must never turn "book this meeting" into a reused success response. Write actions need the idempotency and reconciliation rules in [Reliability Patterns for AI Systems](reliability-patterns.md), not a response cache.

### Data, Retrieval, and Tool-Read Caches

Often the best cache is below the final answer: a document parse, embedding, search result, or slow tool lookup. The model can still reason over the current input.

Include source version, filters, tenant boundary, and permission scope. Apply authorization before a shared result is stored or returned. Filtering afterward can already reveal document identifiers.

Invalidate dependent keys when source data changes. TTL-only invalidation deliberately permits stale data until expiry.

### Prompt Caching

Model providers can reuse computation for a repeated prompt prefix. Put stable instructions, tool definitions, and reference material first. Put the changing request and tool results later.

This reduces input-token cost and usually latency, but it is not an answer cache: the model generates a new response. Provider rules differ, so inspect current documentation and cache-use metrics.

### Semantic Caches

A semantic cache uses an embedding to treat closely worded requests as candidates for reuse. For example, two public password-reset questions may safely share a help answer.

Similarity is not equivalence. A nearby question can differ in user, locale, date, or implied action. Use semantic caches only in narrow domains, with strict boundaries and review of false hits. Avoid them for personalized, sensitive, transactional, or changing questions.

See [Embeddings and Semantic Search](../retrieval/embeddings-and-semantic-search.md) for similarity. It does not make the correctness decision.

### Freshness, Invalidation, and Stampedes

Every value needs an owner and freshness contract: what invalidates it, who may read it, and what happens on cache failure. Read paths should fall back to the source within the request's [deadline](reliability-patterns.md#deadlines-timeouts-and-cancellation), subject to load limits.

If many entries expire together, requests can overload the dependency. This is a cache stampede. Coalesce per-key requests, vary TTLs slightly, prewarm predictable data, and cap refresh work. Serve an older value only when the contract permits it.

## Where It Fits

Caching should sit next to the work it avoids, with the same authorization and version information as the original path.

```text
Authorize → construct scoped key → cache
                                  ├── hit  → permitted, fresh value
                                  └── miss → retrieval / tool read / model call → store
```

Caching retrieval or tool reads may preserve quality better because the model still sees the current question. Choose the lowest layer that removes repeated work.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Exact response cache | Lowest latency and model cost | Stale answers if its key, version, or TTL is incomplete |
| Retrieval or tool-read cache | Reuses dependency work while preserving a fresh answer | Invalidation and authorization remain complex |
| Prompt caching | Lowers repeated input work | Provider-specific behavior and a stable layout |
| Semantic cache | More hits than exact matching | False hits for the wrong situation |
| Long TTL | More hits and lower cost | Larger staleness window |
| Short TTL or frequent invalidation | Fresher values | More misses, source load, and implementation effort |

## Failure Modes

- **Key omits an input:** A response generated for one tenant, permission level, locale, or prompt version is served to another.
- **Cached authorization:** A user keeps access after a role change because the result was not scoped or invalidated correctly.
- **TTL mistaken for consistency:** A policy changes just after its answer is cached, and the application serves it until expiry.
- **Semantic false hit:** A similar-looking request gets an answer that misses an important constraint.
- **Cache stampede:** A popular key expires and many requests call the model or source at once.
- **Cache outage overloads the source:** The fallback sends unlimited traffic to the dependency.
- **Hit-rate tunnel vision:** The dashboard looks good while users receive stale answers or while per-user requests are never appropriate to cache.

## Example

A company has an internal assistant that explains travel policy, which changes by country and policy release.

It caches each retrieval result using country, policy group, query normalization version, index version, and document version. It does not share results between permission groups. A policy change invalidates affected keys. The final answer is not broadly cached because employees add trip-specific details.

It also orders stable instructions and tool schema before conversation state, so supported providers can reuse the prompt prefix. Traces record cache status and cached input tokens. If user corrections rise after a release, engineers inspect document versions and stale-result counts.

## Interview Takeaways

- Define exactly when two requests may share a result before optimizing performance.
- Scope keys to tenant, authorization, versions, source state, and time.
- Prefer stable lower-level caches when final answers are personalized or change often.
- A TTL limits acceptable staleness; invalidation is faster when it is reliable.
- Protect misses from stampedes and measure freshness and quality with hit rate and cost.

## Next

Next: [Model Routing](model-routing.md). Routing chooses the model and execution path for each request; caching changes the cost and latency information that routing should consider.

## References

- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [Microsoft Learn: Cache-Aside Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [AWS: Performance at Scale with Amazon ElastiCache](https://docs.aws.amazon.com/whitepapers/latest/scale-performance-elasticache/scale-performance-elasticache.pdf)
- [OpenAI: Prompt Caching](https://developers.openai.com/api/docs/guides/prompt-caching)
- [Anthropic: Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
