# Streaming AI Responses

[Model Routing](model-routing.md) chooses an appropriate model before work begins. Streaming changes what happens while that model is working: the application shows useful progress before the whole response is ready. The next concern is [Security](../security/security-for-ai-systems.md), which defines what the application may expose and act on.

## The Problem

Generating a useful answer can take several seconds. If the server waits for the complete answer, a user may see an empty screen and assume the application is stuck, even when the model is making progress.

Sending partial output improves *time to first useful feedback*, but it changes the contract. The client can disconnect after seeing half an answer. A network can fail after the model has produced more text. A safety check may reject content that has already been displayed. An agent may be working on a tool call, where showing half-formed JSON as prose is confusing or unsafe.

Streaming is therefore not just "send tokens sooner." It is a protocol and state-management decision for an incomplete result.

## Mental Model

Treat a streamed response as a short-lived sequence of typed events that reaches a terminal state.

```text
request accepted
      ↓
status: searching
      ↓
text updates: "Your plan..."
      ↓
completed  OR  incomplete  OR  failed  OR  cancelled
```

Text updates are provisional until the terminal event. The server remains the source of truth for the final status and saved conversation.

## How It Works

1. **Identify the task.** Create a response ID, validate input, and begin a trace before opening the stream.
2. **Run the pipeline.** Convert provider chunks from retrieval, generation, tools, and safety checks into stable application events.
3. **Send meaningful events.** Emit `started`, `text_delta`, `tool_status`, `completed`, `incomplete`, and `error`. Do not make the browser infer meaning from raw provider events.
4. **Render and accumulate separately.** Show text as it arrives, but retain its sequence and final status. Text alone does not mean the answer completed.
5. **Finish explicitly.** Send one terminal event with the response ID and finish reason. Persist the authoritative completed result before claiming success.
6. **Handle interruption honestly.** Propagate cancellation when practical. On connection loss, replay retained events or show an interrupted answer. Never stitch together two independent attempts as if they were one answer.

```text
Model provider stream
          ↓
Application: validate, moderate, normalize, record
          ↓
Client stream: status + displayable deltas + terminal event
```

## Important Concepts

### Chunks Are Transport Units, Not Words

A chunk may contain text, metadata, a tool-call fragment, or no user-visible text. Its boundary is controlled by the provider and network, not by grammar. Append only fields your application defines as displayable text.

Buffer enough text to avoid a distracting character-by-character UI, but do not wait for the full answer. Measure time to first displayed text and total completion time.

### A Small Application Event Contract

Server-Sent Events (SSE) are a common fit for one-way server-to-browser updates. WebSockets help when the session needs frequent two-way messages. The transport matters less than a clear application contract.

Keep that contract provider-neutral and typed. For example:

```text
started       response_id
text_delta    sequence, text
tool_status   tool name, safe progress label
completed     final response ID, usage, citations
incomplete    reason, resumable?
error         safe error code, retry guidance
```

Include increasing sequence numbers if the client may reconnect. SSE's `Last-Event-ID` only helps when *your application* retains events or a final result to replay. A provider connection cannot usually resume arbitrary generation after a browser reconnects.

### Partial State, Cancellation, and Reconnection

Make `completed`, `incomplete`, `failed`, and `cancelled` distinct. Incomplete means the answer stopped early or a recoverable step did not finish. Failed means the system could not produce a usable result.

If the user presses Stop, abort upstream work when supported and mark the response cancelled. Cancellation may arrive late, so metering or audit data can still show generated work.

For short answers, show the saved final answer if it completed; otherwise show an interrupted state and let the user retry. Replay is worthwhile when restarting wastes meaningful time.

### Tools and Structured Output

Tool calls and JSON are frequently streamed in fragments. Accumulate them out of view, validate the complete value against the [tool schema](../tools/structured-outputs-and-tool-calling.md), then decide whether to invoke the tool. Never execute a tool from a partial argument string, and never present internal tool arguments as user-facing progress.

For an agent, stream honest status such as "Searching account records" or "Waiting for approval". Do not stream a predicted final answer while the system is still gathering evidence. The answer could change after the tool result arrives.

### Safety Before and During Display

Input controls still run before streaming begins. For output, there is a real choice:

- **Buffer then check:** Screen a larger unit before display. This offers stronger control but delays the first visible text.
- **Check incrementally:** Display sooner while checking chunks or short windows. This improves responsiveness but can expose unsafe text before a later intervention.

High-risk domains should favor buffering, safe templates, or constrained output. If an output check intervenes, stop the stream and show a clear safe message. A later warning cannot undo content already copied or acted upon.

## Where It Fits

Streaming sits at the response boundary and must reflect the state of the AI system.

```text
Browser or app
       ↕ cancellation / reconnect
Application response service
       ↓               ↑
retrieve → model → tools → validate and moderate
       ↓
typed progress and terminal events
```

[Reliability Patterns for AI Systems](reliability-patterns.md) supplies deadlines, cancellation, retries, and honest degraded behavior. [Observability for AI Systems](observability-for-ai-systems.md) records time to first event, disconnects, terminal reasons, and the full task trace. Streaming should expose progress, not bypass either system.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Stream text immediately | Perceived responsiveness | Partial, unverified content and more client state |
| Buffer before displaying | Output-safety control and simpler final rendering | Worse time to first text |
| Typed application events | Stable client behavior across providers | Event-design and compatibility work |
| Event replay on reconnect | Better recovery for long work | Durable event storage, ordering, and retention concerns |
| Propagated cancellation | Lower wasted work and cost | More lifecycle coordination; cancellation may arrive late |
| Stream tool status only | Honest progress without leaking internals | Less detailed UI |

## What Can Go Wrong

**Treating the last text chunk as success.** A stream can end because of a network error, safety stop, or output limit.

**Rendering provider events directly.** A provider upgrade or a tool-call fragment breaks the user interface or leaks internal data.

**Duplicate text after reconnect.** The client appends replayed events without sequence checks or reset logic.

**Retrying a broken connection as a new conversation turn.** The user sees two different partial answers merged together.

**Ignoring client disconnects.** The backend continues expensive generation for abandoned requests.

**Executing partial structured output.** An incomplete tool argument becomes a malformed or unintended action.

**Choosing fast display where safety needs buffering.** Harmful or sensitive text is shown before the check can intervene.

**No terminal event or saved final state.** The UI cannot distinguish a slow answer from one that was lost.

## Example

A travel assistant searches policy documents, then streams an answer about changing a ticket. It emits `started`, then `status: searching policies`, then ordered `text_delta` events. The booking tool is not shown as raw JSON; the UI says "Checking your reservation."

The user closes the tab during generation. The server stops the model request and records cancellation. On return, the conversation shows an interrupted state, not a confirmed partial answer. A new request starts a new response ID.

## Interview Takeaways

- Streaming reduces time to first useful feedback, but it turns one response into an ordered lifecycle with partial state.
- Normalize provider chunks into typed application events and require an explicit terminal event.
- A reconnect can replay only state the application retained; it does not automatically resume arbitrary generation.
- Validate complete structured output before using it, and stream safe tool progress rather than raw arguments.
- Output safety and fastest display are a deliberate tradeoff. Choose buffering or incremental checking according to the harm of early exposure.

## Next

Next: [Security for AI Systems](../security/security-for-ai-systems.md). It defines the trust boundaries, permissions, and data protections that streaming must respect.

## References

- [WHATWG HTML Standard: Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [MDN: Using Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [OpenAI API Reference: Streaming Events](https://platform.openai.com/docs/api-reference/responses-streaming)
- [Amazon Bedrock: Configure Streaming Response Behavior to Filter Content](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-streaming.html)
