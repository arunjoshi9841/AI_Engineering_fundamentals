# Structured Outputs and Tool Calling

[Retrieval-Augmented Generation](../rag/retrieval-augmented-generation.md) gives an LLM evidence to read. Many applications also need the LLM to produce machine-readable data or request a real operation. Structured outputs and tool calling create that boundary.

## The Problem

Free-form text is useful for a reply, but fragile for software. A response such as "This is probably urgent and should go to billing" is difficult to route reliably. The same problem becomes dangerous when the requested operation has a side effect, such as sending an email, creating a payment, or deleting a document.

The application needs a contract for what the model may return, and it must keep control of what actually happens.

## Mental Model

Structured output is a form the model fills in. Tool calling is a request form the model fills in for the application.

```text
Structured output
LLM ──> { category: "billing", urgency: "high" }
                         ↓
                    Application uses data

Tool calling
LLM ──> { tool: "lookup_invoice", invoice_id: "inv_123" }
                         ↓
                 Application validates and executes
                         ↓
                    Tool result returns to LLM
```

The key rule is:

> The LLM proposes actions. The application owns execution.

## How It Works

Tool calling normally follows this sequence:

1. Define each allowed tool with a name, clear description, input schema, expected result, and side-effect behavior.
2. Give the tool definitions and the current task to the LLM.
3. The LLM returns either a normal response or a proposed tool name and arguments.
4. The application validates the arguments and applies authorization and business rules.
5. The application executes the tool, records the result, and returns a structured observation to the LLM if another step is needed.
6. The LLM uses the result to respond or propose its next action.

```text
LLM proposes tool + arguments
            ↓
Validate schema and business rules
            ↓
Authorize or request human confirmation
            ↓
Execute application-owned tool
            ↓
Return a structured success or error result
```

## Important Concepts

### Schemas and Structured Outputs

A schema defines the expected shape of data: its fields, types, required values, and allowed alternatives. JSON Schema is a common way to express such a contract.

```json
{
  "category": "billing",
  "priority": "high",
  "needs_human_review": false
}
```

Use structured output when the LLM is classifying, extracting, planning, or choosing a route and your application needs dependable parsing. A schema reduces formatting mistakes, but it cannot make the values true. `priority: "high"` can match the schema and still be a bad judgment.

Validate again in application code. Check semantic constraints that a general schema cannot express, such as whether a customer ID exists or whether a requested amount is within a permitted limit.

### Tool Definitions

A tool definition is an API contract written for both the model and the application. It should state:

- what the tool does and when to use it
- argument names, types, and constraints
- expected result fields
- whether it reads data or changes state
- retry safety, timeouts, and known errors

Prefer narrow tools. `get_invoice(invoice_id)` is easier to authorize, observe, and test than a generic `run_database_query(query)`.

Tool descriptions help the model choose correctly. They do not grant permission. The server must enforce tenant boundaries, ownership, rate limits, and every business rule independently.

### Validation and Authorization

Validation asks whether the request is well formed. Authorization asks whether this actor may perform it. They are separate checks.

```text
Model arguments
    ↓
Schema validation: is invoice_id present and correctly shaped?
    ↓
Authorization: may this user access that invoice?
    ↓
Business rule: is the invoice eligible for the requested operation?
```

For destructive or high-impact actions, require explicit user confirmation after the application has shown the concrete action. Never infer confirmation from an ambiguous chat message.

### Results, Errors, and Retries

Tools should return machine-readable outcomes, not only prose. For example:

```json
{ "ok": false, "code": "NOT_FOUND", "retryable": false }
```

This lets the application and model distinguish a missing record from a temporary timeout. Limit retries and make their conditions explicit.

Retries are especially dangerous for side effects. Retrying `send_email()` or `create_payment()` after a network timeout may perform the action twice. Use an idempotency key, check the recorded operation state, or require a human to resolve an uncertain outcome. A retryable read is not the same as a retryable write.

### Parallel Calls

Independent, read-only tools can run in parallel, such as looking up an order and retrieving its shipping status. Dependent calls must wait for the result they need.

Do not parallelize actions that can conflict, spend money, or mutate the same resource unless the underlying system makes their ordering and idempotency safe.

## Where It Fits

Structured outputs turn language into data. Tool calling lets that data participate in a controlled application loop.

```text
User request
     ↓
LLM proposes structured result or tool call
     ↓
Application validates, authorizes, and executes
     ↓
Tool result or final response
```

This separation is the prerequisite for [Agents](../agents/agents.md), where the model can choose several actions over multiple steps.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Free-form output | Flexibility and fast prototyping | Fragile parsing and ambiguous integration |
| Structured output | Reliable shape for software | Schema design and still requires semantic validation |
| Narrow tools | Safety, testing, and authorization | More interfaces to maintain |
| Automatic retry | Resilience to transient reads | Duplicate side effects without idempotency |
| Human confirmation | Control over destructive actions | Extra user interaction and latency |
| Parallel reads | Lower latency | Coordination complexity |

## What Can Go Wrong

**Valid but wrong arguments.** The output matches the schema but identifies the wrong customer or amount.

**Authorization bypass.** The application trusts the model's choice without checking the caller's permissions.

**Duplicate side effect.** A timeout triggers an unsafe retry of a write operation.

**Ambiguous tool result.** The tool returns prose that neither code nor the model can handle reliably.

**Overly broad tool.** A generic action interface gives the model far more authority than the task requires.

**Unconfirmed destructive action.** A chat instruction is treated as permission to delete, pay, or send.

## Example

An LLM helps support agents process refund requests. It first returns structured output that classifies the request and identifies the claimed order. The application validates the order ID and checks that the support agent may access it.

If a refund is eligible, the LLM may propose `create_refund(order_id, amount)`. The application displays the amount to the support agent, requires confirmation, sends an idempotency key with the request, and returns the recorded refund ID to the model. The model can then draft the customer reply.

## Interview Takeaways

- Structured output constrains the shape of model output; it does not validate truth, permissions, or business rules.
- Tool calling lets a model propose an operation, while the application validates, authorizes, and executes it.
- Tool schemas should describe inputs, results, side effects, and retry behavior.
- Treat every side-effecting tool call as an idempotency and confirmation problem.
- Parallelize only independent operations that are safe to run concurrently.

## Next

Next: [Agents](../agents/agents.md). They use structured outputs and tool calls in a bounded loop to pursue multi-step tasks.

## References

- [JSON Schema specification](https://json-schema.org/specification)
- [JSON Schema validation vocabulary](https://json-schema.org/draft/2020-12/json-schema-validation)
- [OpenAI Structured Outputs guide](https://developers.openai.com/api/docs/guides/structured-outputs)
- [OpenAI Function Calling guide](https://developers.openai.com/api/docs/guides/function-calling)
