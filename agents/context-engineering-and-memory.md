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

### Calculating a Context Budget

Persistent memory does not consume context until the application selects it for a request. Calculate the budget for that selected working set, not for everything stored in the database.

Use a model's documented context window and reserve output space first. For example, GPT-4.1 currently has a **1,047,576-token context window** and supports up to **32,768 output tokens**. Suppose an incident-response agent configures `max_output_tokens` to 4,000 because it needs a concise investigation update.

```text
context window                    1,047,576 tokens
- configured maximum output           4,000 tokens
-----------------------------------------------
maximum possible input             1,043,576 tokens
```

That is a hard limit, not a target. Sending one million input tokens on every call would be expensive and would usually distract the agent. Set a smaller operating budget based on the task. For this agent, use a 128,000-token input budget:

| Input allocation | Budget | Why it exists |
| --- | ---: | --- |
| Fixed instructions and tool schemas | 12,000 | Stable rules and allowed capabilities |
| Current task and latest user message | 4,000 | The immediate objective |
| Recent history and current plan | 16,000 | Local continuity |
| Selected long-term memory | 12,000 | Relevant prior facts or preferences |
| Retrieved knowledge | 32,000 | Source-backed evidence |
| Useful tool results | 16,000 | Observations needed for the next decision |
| Safety reserve | 36,000 | Unexpectedly long results, formatting, and future room |
| **Total input budget** | **128,000** | |

Only 92,000 tokens in that table are planned payload. The remaining 36,000 tokens are deliberately unused. If a tool returns a 20,000-token log instead of the expected 2,000 tokens, the reserve prevents a failed request or the accidental removal of important instructions.

The reusable calculation is:

```text
maximum_input = context_window - configured_max_output
operating_budget = min(maximum_input, chosen_cost_and_quality_budget)
payload_budget = operating_budget - safety_reserve
```

For the example:

```text
maximum_input = 1,047,576 - 4,000 = 1,043,576
operating_budget = 128,000
payload_budget = 128,000 - 36,000 = 92,000
```

Before each request, count every input component: instructions, message-role formatting, tool definitions, conversation turns, retrieved chunks, and tool results. Use the tokenizer and token-usage fields for the exact provider and model in production. If the planned payload exceeds 92,000 tokens, select less context, compress older state, summarize a tool result, or retrieve fewer and better passages. Do not silently drop the newest user instruction or an authorization constraint.

Token budgets are also cost budgets. GPT-4.1's listed price is $2 per million input tokens and $8 per million output tokens. At the example's maximum 128,000 input tokens and 4,000 output tokens, the approximate worst-case token charge is:

```text
input:  128,000 / 1,000,000 × $2 = $0.256
output:   4,000 / 1,000,000 × $8 = $0.032
total:                               $0.288 per call
```

Actual output is often smaller, and cached input may use different pricing. Track actual token usage instead of assuming every request reaches the configured maximum.

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

Next: Event-Driven AI Systems. It explains how AI work can begin or continue reliably when source changes, approvals, or background jobs produce events.

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
- [OpenAI, GPT-4.1 model documentation](https://developers.openai.com/api/docs/models/gpt-4.1)
