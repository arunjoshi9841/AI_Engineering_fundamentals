# Streaming AI Responses

[LLM Inference and Serving](../inference/llm-inference-and-serving.md) explains why generation is incremental: after prefill, the model decodes output token by token. This page explains how an application exposes that work safely to a user. It leads into [Production Architecture](production-architecture.md), where streaming becomes one part of the request path.

## The Problem

An answer may take seconds to generate. Waiting for the final result makes a working application appear stuck. Streaming improves time to first useful feedback, but it means the user can see an incomplete answer, disconnect midway, or cancel while the model is still running.

The important design decision is not how to send tokens sooner. It is how to represent an incomplete response honestly.

## Mental Model

A stream is a short-lived, ordered lifecycle with an explicit final state.

```text
accepted → searching → text updates → completed
                                  ↘ incomplete / failed / cancelled
```

Displayed text is provisional until a terminal event. The server, not the browser, owns the saved final result and response status.

## A Small Response Contract

Normalize provider-specific chunks into a few typed application events:

```text
started       response ID
status        safe progress label
text_delta    sequence number, displayable text
completed     final response ID, citations, usage
incomplete    reason, resumable?
failed        safe error code, retry guidance
cancelled     response ID
```

The transport can be Server-Sent Events for one-way browser updates or WebSockets when the client needs frequent two-way messages. The application contract matters more than the transport. Chunks can contain text, tool-call fragments, or metadata, so never render raw provider events directly.

Use increasing sequence numbers if the client can reconnect. Reconnection can replay only events or final state that the application retained. It does not resume an arbitrary model-generation connection.

## Handling Partial Work

1. Create a response ID and trace before opening the stream.
2. Translate retrieval, model, tool, and safety activity into stable events.
3. Persist the final result before emitting `completed`.
4. On client cancellation or disconnect, abort upstream work when practical and record the terminal state.
5. If a result did not complete, present it as interrupted and let the user start a new response. Do not merge two attempts into one answer.

Tool calls and structured output need a stricter rule: accumulate fragments privately, validate the complete value against its schema, then invoke the tool. Stream a safe status such as "Checking your reservation," not raw arguments or a predicted answer before evidence arrives.

Safety also changes the display choice. Buffering output before display gives stronger control but delays feedback. Incremental checks improve responsiveness but can expose content before a later intervention. Choose based on the harm of early exposure, and use constrained output or safe templates for high-risk features.

## Where It Fits

```text
Browser or app
       ↕ cancellation and reconnect
Response service
       ↓
retrieve → model decode → tools → validation
       ↓
typed progress and terminal events
```

[Reliability Patterns](reliability-patterns.md) supplies deadlines and honest recovery. [Observability](observability-for-ai-systems.md) records time to first event, disconnects, terminal reasons, and the task trace. Streaming must reflect those systems, not bypass them.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Display text immediately | Faster perceived response | Partial, unverified content and more client state |
| Buffer before display | Stronger output control | Slower first visible feedback |
| Typed events | Stable client behavior across providers | A contract to design and maintain |
| Retain events for replay | Better reconnect experience | Storage, ordering, and retention work |

## Failure Modes

- Treating the last text chunk as success when the stream actually failed or hit a limit.
- Rendering provider events directly, which leaks tool fragments or breaks after a provider change.
- Duplicating text after reconnect because replay lacks sequence checks.
- Continuing costly generation after the user leaves.
- Executing incomplete JSON or tool arguments.
- Claiming an interrupted answer completed.

## Example

A travel assistant emits `started`, then `status: searching policies`, followed by ordered text updates. When it needs booking data, the UI shows "Checking your reservation" while the complete tool request is validated privately. If the user closes the tab, the server cancels the upstream request and records the response as cancelled. Returning to the conversation shows an interrupted answer, not a confirmed one.

## Interview Takeaways

- Streaming turns one answer into an ordered lifecycle with partial and terminal states.
- Use provider-neutral, typed events and preserve authoritative final state on the server.
- Reconnection requires retained application state; it does not automatically resume generation.
- Validate complete structured output before action, and choose buffering or incremental checks according to safety risk.

## Next

Next: [Production Architecture for AI Systems](production-architecture.md). It places streaming alongside retrieval, tools, workflows, security, and operations.

## References

- [WHATWG HTML Standard: Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [OpenAI API Reference: Streaming Events](https://platform.openai.com/docs/api-reference/responses-streaming)
