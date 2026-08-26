# Calculating a Context Budget

This subpage expands [Context Engineering and Memory](context-engineering-and-memory.md) with a practical token-budget calculation. Persistent memory does not consume context until the application selects it for a request. Calculate the budget for that selected working set, not for everything stored in the database.

## The Calculation

Start with a model's documented context window and reserve output space first. For example, GPT-4.1 currently has a **1,047,576-token context window** and supports up to **32,768 output tokens**. Suppose an incident-response agent configures `max_output_tokens` to 4,000 because it needs a concise investigation update.

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

## A Reusable Formula

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

Before each request, count every input component: instructions, message-role formatting, tool definitions, conversation turns, retrieved chunks, and tool results. Use the tokenizer and token-usage fields for the exact provider and model in production.

If the planned payload exceeds 92,000 tokens, select less context, compress older state, summarize a tool result, or retrieve fewer and better passages. Do not silently drop the newest user instruction or an authorization constraint.

## Calculate the Cost Too

Token budgets are also cost budgets. GPT-4.1's listed price is $2 per million input tokens and $8 per million output tokens. At the example's maximum 128,000 input tokens and 4,000 output tokens, the approximate worst-case token charge is:

```text
input:  128,000 / 1,000,000 × $2 = $0.256
output:   4,000 / 1,000,000 × $8 = $0.032
total:                               $0.288 per call
```

Actual output is often smaller, and cached input may use different pricing. Track actual token usage instead of assuming every request reaches the configured maximum.

## References

- [OpenAI, GPT-4.1 model documentation](https://developers.openai.com/api/docs/models/gpt-4.1)
