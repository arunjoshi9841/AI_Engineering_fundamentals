# Context Engineering and Memory

[Agents](agents.md) choose actions over several LLM calls. Context engineering decides what belongs in a call's finite context window. Memory keeps useful state outside it for later selection.

## The Problem

An incident-response agent may run for an hour. It reads a runbook, log queries, deployment history, and a human approval. Sending the full transcript on every turn eventually exceeds the context window. Even before that, old tool output can distract the model from the current failure.

Truncating everything old loses decisions and completed work. Keeping everything is expensive and often makes the next decision worse. The system needs a deliberate way to retain, retrieve, compress, and discard information.

## Mental Model

Treat model context as a small working desk, not a filing cabinet.

```text
instructions
+ current task and user identity
+ relevant conversation state
+ selected knowledge and memory
+ useful recent tool results
-------------------------------
          model context
```

The application stores the full record elsewhere. Before each LLM call, it builds a working set from that record. The goal is not maximum context. It is the smallest set of information that lets the model make the next correct decision.

> More context is not automatically better context.

## How It Works

1. **Define a token budget.** Reserve space for instructions, the current task, selected context, and the model's output.
2. **Persist state outside the prompt.** Store messages, tool results, decisions, approvals, and task status with IDs and timestamps.
3. **Select the next working set.** Include the current task, recent relevant turns, required instructions, and only the evidence or memories needed for the next step.
4. **Compress what must survive.** Replace older detail with a structured summary or a small durable note that preserves decisions, facts, open questions, and source references.
5. **Retrieve long-term memory selectively.** Search or look up scoped records when they are relevant, then pass the selected result into the prompt.
6. **Write and retire memory carefully.** Validate persistent writes, record provenance and time, and expire, supersede, or delete records when they should no longer influence behavior.

```text
Durable conversation, task state, and memory
                    ↓
       select, retrieve, and compress
                    ↓
             bounded model context
                    ↓
          response or proposed action
                    ↓
       persist the new useful state
```

## Important Concepts

### Context Budget and Selection

The context window is a hard token limit, but a context budget is the application's allocation inside that limit. A practical budget might reserve space for fixed instructions and output before deciding how much history or retrieved material may enter.

Start with the information directly needed for the current step. For an agent, that is usually the task, the latest meaningful observation, the current plan or status, and a few relevant tool results. Raw logs, full search responses, and old unsuccessful attempts rarely deserve permanent space.

Long-context models do not remove this design problem. More tokens cost more and can make relevant information harder to use among unrelated material.

For a worked example with a real model limit, safety reserve, and cost calculation, see [Calculating a Context Budget](calculating-context-budgets.md).

### History, Summaries, and Tool Results

**Short-term memory** is state within one conversation or task, such as recent messages, the current plan, and the last tool result. It should be durable enough to resume after a worker restart, even though it is scoped to that task.

As a task grows, keep recent detail and compact older detail. A good summary records verified facts, decisions, unfinished work, constraints, and pointers to original records:

```text
Incident INC-482
- Verified: checkout errors affect eu-west only since deploy 2026.08.25.3.
- Completed: retrieved runbook and compared baseline metrics.
- Open: confirm rollback approval from on-call engineer.
- Evidence: metric query q_91; deployment record d_204.
```

Keep the parsed conclusion and a reference to the full result, instead of repeatedly sending a 5,000-line log response. Retain raw evidence outside the prompt for audit and debugging.

### Long-Term Memory

**Long-term memory** survives beyond a single conversation or task. It is application data, not a change to the model's weights:

| Type | Example | Typical retrieval |
| --- | --- | --- |
| Semantic memory | "Acme invoices use fiscal year starting in February" | Search by meaning or entity |
| Episodic memory | "On May 4, the user approved a one-time data export" | Look up by user, task, and time |
| User memory | Preferred report format or accessibility setting | Read by user ID and policy |

These labels guide storage and retrieval. An approval should be an immutable workflow record, not a model-written note. Current balances, permissions, and order status must still come from authoritative systems at execution time.

### Memory Selection and Lifecycle

Every memory needs a scope, provenance, creation time, and, where appropriate, expiry or a source version.

Retrieval-based memory follows [RAG](../rag/retrieval-augmented-generation.md): search candidates, apply permission and scope filters, then add only the best results. Retrieval is not permission enforcement.

Persistent writes need a higher bar than reads. Validate a proposed memory's schema, scope, source, and confidence. Require user confirmation or a trusted system event for sensitive or consequential information.

## Where It Fits

Context engineering sits inside each LLM call. Memory and task state surround the call so an agent can continue without replaying its entire past.

```text
User task
     ↓
Agent state + selected memory + retrieved knowledge
     ↓
Context construction
     ↓
LLM call and approved tools
     ↓
Updated task state and, when justified, memory
```

It builds on [RAG](../rag/retrieval-augmented-generation.md) and [Structured Outputs and Tool Calling](../tools/structured-outputs-and-tool-calling.md), which keeps persistent writes application-controlled.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Full transcript | Simple prototype continuity | Token growth, cost, and irrelevant context |
| Sliding recent window | Bounded context and low implementation effort | Important older facts can disappear |
| Summarized history | Longer-running coherence | Summaries can omit or distort details |
| Retrieved long-term memory | Cross-session continuity | Retrieval errors, latency, and stale records |
| Automatic memory writes | Less manual state handling | Incorrect or sensitive facts persist |
| Explicit memory lifecycle | Privacy and correctness | More data-model and operational work |

## Failure Modes

- **Context overflow:** Fixed instructions, history, and tool results leave no room for the output or force important content out.
- **Lost decision during compaction:** A summary omits an approval, constraint, or failed attempt, causing the agent to repeat unsafe work.
- **Context pollution:** Broad retrieval or full transcripts bury the evidence needed for the next action.
- **Stale or contradictory memory:** An old preference or policy remains after its source changed.
- **Memory leak:** A memory retrieved without tenant, user, or permission checks exposes data from another scope.
- **Poisoned memory:** Untrusted text or a model hallucination is stored as a durable fact and influences later tasks.

## Example

A customer-success agent prepares a renewal brief across several sessions. It keeps the account ID, task, and recent notes in short-term state. Before each call, it retrieves the current contract and usage data from authoritative systems, then optionally retrieves a user-approved preference: "show costs in EUR."

When an account manager confirms a renewal risk, the application records it as a dated workflow record, not a conversational summary. Older research becomes a status note with links to source documents, so the agent does not resend every tool response.

## Interview Takeaways

- Context engineering selects a bounded working set for each model call; it is more than writing a better prompt.
- Conversation history, task state, retrieved knowledge, and persistent memory have different lifecycles and should not be stored as one opaque transcript.
- Summaries and tool-result compression save tokens but must preserve decisions, constraints, and pointers to original evidence.
- Long-term memory is application-owned data, not model learning. Scope, provenance, freshness, permissions, and deletion are design requirements.
- More context can increase cost and distraction, so evaluate whether the right information is present and useful at the next decision.

## Next

Next: Long-Running and Durable AI Workflows. It explains how AI work persists safely across background work, restarts, and human approval.

## Go Deeper

### Structured Notes Over Free-Form Summaries

For long-running work, a schema such as `facts`, `decisions`, `open_questions`, `completed_steps`, and `evidence_ids` is easier to validate than prose. Keep notes a working aid, not a duplicate database.

### Evaluate Context and Memory Separately

Test whether the needed fact was selected, fresh, authorized, and used correctly. Selection, stale-memory, and generation failures need different fixes.

## References

- [Anthropic, *Effective Context Engineering for AI Agents*](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Liu et al., *Lost in the Middle: How Language Models Use Long Contexts*](https://arxiv.org/abs/2307.03172)
- [Packer et al., *MemGPT: Towards LLMs as Operating Systems*](https://arxiv.org/abs/2310.08560)
- [Park et al., *Generative Agents: Interactive Simulacra of Human Behavior*](https://arxiv.org/abs/2304.03442)
