# Long-Running and Durable AI Workflows

[Context Engineering and Memory](../agents/context-engineering-and-memory.md) explains how an agent selects useful state for its next model call. A durable workflow keeps the larger task alive when it outlasts that call, its worker process, or an immediate response to the user.

## The Problem

An AI task often starts in one request but cannot safely finish there. An agent may gather evidence, wait two days for an approval, trigger a background job, and then report the result. A server restart, timeout, deployment, or retry can interrupt any point in that sequence.

A plain background job can retry work, but it does not by itself answer important questions:

- Did the approval belong to this task?
- Was the provisioning request already sent before the worker crashed?
- Which evidence and model decision led to the next step?
- Can the task resume without repeating a charge, email, or deletion?

Long-running workflows make those questions part of the application design.

## Mental Model

Treat a workflow as a durable case file, not a function that stays in memory.

```text
Request starts workflow WF-42
          ↓
Persist task state and run a step
          ↓
Record its result before the next step
          ↓
Wait, retry, branch, or ask for approval
          ↓
Resume WF-42 on any available worker
          ↓
Record final outcome
```

The workflow can run for minutes, days, or longer because its state is stored outside the worker process. A worker may disappear while the workflow is waiting; another worker can continue from the last recorded checkpoint.

Durability does **not** mean an external action happens exactly once by magic. It means the application retains enough state to resume and reason about an interrupted action. Side effects still need idempotency, reconciliation, or human review.

## How It Works

1. **Create one execution record.** Give the task a stable workflow ID, input, owner, status, and time limits. Record the user and authorization context that began it.
2. **Break work into meaningful steps.** A step might call an LLM, retrieve evidence, start an external job, or write a record. Store its completed result durably before allowing dependent work to continue.
3. **Make side effects safe to repeat.** Associate every external write with a stable idempotency key or an application record that can detect an earlier attempt.
4. **Persist waits.** Store that the workflow is waiting for an approval, timer, callback, or job result. The waiting worker should be free to stop or serve other work.
5. **Resume by correlation.** When an approval or callback arrives, authenticate it, associate it with the correct workflow and step, and check that it has not expired or already been handled.
6. **Finish or repair deliberately.** Record success, failure, cancellation, or an operator-needed state. Monitor workflows that exceed their expected duration and provide a safe repair path.

```text
Durable workflow state
        ↓
Worker runs one eligible step
        ↓
LLM, tool, or external system
        ↓
Record result, retry policy, or waiting state
        └─────────────── resume later from stored state
```

## Important Concepts

### Checkpoints and Step Boundaries

A checkpoint is a stored fact about workflow progress: "retrieval completed," "approval requested," or "provisioning succeeded with request ID `p_82`." It is the boundary that lets a new worker continue without guessing what happened before it started.

Choose step boundaries around externally visible work. A database read and an LLM call may be separate steps. Sending a message, charging a card, or changing a permission must be an explicit step with an outcome record.

Do not keep the only copy of progress in process memory, an in-memory agent transcript, or a browser connection. Those are transports, not workflow state.

### Retries and Idempotent Side Effects

Workers can fail after an external service accepts a request but before the worker records success. On retry, the outcome is uncertain: the request may have happened already.

For a retryable write, use a stable idempotency key, such as one derived from the workflow ID and step ID. The receiving service or your own database should treat repeated requests with that key as the same operation. If the target cannot deduplicate and a duplicate would be harmful, do not blindly retry. Reconcile the outcome first or send the task to an operator.

```text
WF-42 / provision-access
        ↓
idempotency key: WF-42:provision-access
        ↓
External service returns the same recorded result on a retry
```

Reads and idempotent upserts are usually easier to retry than irreversible writes. The retry policy should name the timeout, maximum attempts, backoff, and errors that are safe to retry.

### Waiting for People and External Work

A durable wait is persisted state, not a long `sleep()` call. The workflow records what it expects, then releases its worker. A later event wakes the workflow and supplies the answer.

An approval should carry more than "approved." Store the workflow ID, requested action, approver identity, decision time, expiry, and the approval record ID. Before acting, recheck that the target still exists and that the approver and requester still have the necessary permissions.

The same pattern works for a batch job, a vendor callback, or a timer. Every wait needs a timeout and an owner for resolving an expired or missing response.

### LLM State Is Workflow State

An LLM response is not a durable plan merely because it appeared in a chat. After validating a structured model decision, store the accepted decision, its inputs or evidence references, and any tool results needed to continue. On resume, rebuild a bounded context from that state as described in [Context Engineering and Memory](../agents/context-engineering-and-memory.md).

Model calls can be nondeterministic, can time out after reaching the provider, and may produce a different answer if repeated. Do not assume a retry is equivalent to resuming. Record the result you accepted, and keep model-generated text separate from authoritative workflow facts, approvals, and permissions.

### Workflow Definition and Execution

The **definition** is the code or state machine that describes allowed steps. The **execution** is one durable instance, such as `WF-42`, with its own input, history, and status.

This distinction makes operations practical. You can inspect one stuck execution, cancel it, or repair it without changing every future workflow. It also makes deployment changes safer: a running execution may have started under an older definition and must remain interpretable after code changes.

## Where It Fits

A durable workflow provides the execution layer around an [agent](../agents/agents.md), tools, approvals, and background jobs. The agent can choose a next action, but the workflow owns the durable record of which action was approved and completed.

```text
User request
     ↓
Durable workflow execution
     ├── agent or fixed decision step
     ├── retrieve evidence
     ├── call controlled tool
     ├── wait for human or external result
     └── record outcome and notify user
```

Use a fixed workflow when the path is known, such as a review followed by approval and provisioning. Use an agent inside a bounded step when evidence determines the next safe action. The durable layer should not give the agent more authority than the original request allows.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Ordinary background job | Simple short-lived retry work | Weak support for long waits, inspection, and multi-step recovery |
| Durable workflow | Recovery, visibility, and long-lived coordination | State modeling and a workflow runtime or equivalent infrastructure |
| Fine-grained checkpoints | Precise resume and audit trail | More stored history and versioning discipline |
| Automatic retries | Resilience to transient failure | Duplicate external effects without idempotency |
| Human approval waits | Control over consequential actions | Expiries, stuck work, and a slower user experience |
| Persisted LLM decisions | Reliable continuation and auditability | Storage of sensitive task data and careful context reconstruction |

## Failure Modes

- **State only in memory:** A restart loses the current step or causes the workflow to repeat it blindly.
- **Duplicate side effect:** A timeout after an external write leads to a second email, payment, or permission change.
- **Wrong resume:** An approval or callback is correlated to the wrong workflow, stale step, or expired request.
- **Stuck wait:** No timeout, alert, or owner exists for a workflow waiting on a person or external system.
- **Stale authority:** The workflow uses the permissions captured at start even though access was revoked while it waited.
- **Unrecorded model decision:** A resumed agent recreates a different plan because the accepted prior result was never stored.
- **Unsafe definition change:** New code cannot interpret the state of an execution that began under an older version.

## Example

An IT assistant handles an employee's request for access to a production dashboard. It first retrieves the employee's role and the dashboard's policy. The agent can summarize why access is requested, but it cannot grant access itself.

The workflow records a request ID and sends an approval request to the dashboard owner. It moves to `waiting_for_approval` and releases its worker. Two days later, an authenticated approval event names the workflow and approval record. The workflow verifies that the request is still open and that the approver still owns the dashboard.

It then calls the provisioning service with the idempotency key `WF-42:grant-dashboard-access`. If the worker crashes after sending that call, a retry asks the provisioning service for the result associated with the same key instead of granting access twice. The workflow records the final permission change and sends the employee a status update.

## Interview Takeaways

- A long-running workflow is durable when its execution state survives workers, restarts, and long waits.
- Checkpoint completed steps and waits outside process memory so a different worker can resume safely.
- A retry cannot guarantee an external write was not performed. Use idempotency keys, outcome reconciliation, or human handling.
- Model output is workflow input, not the source of truth. Persist accepted decisions and revalidate authority before consequential work.
- Waiting for approval or a callback requires correlation, authentication, expiry, and monitoring, not a thread that sleeps.

## Next

Next: [Evals: Measuring AI Systems](../evaluation/evals.md). It asks whether the workflow, agent decisions, and final outcomes are actually correct and reliable.

## Go Deeper

### Replay-Based Workflow Engines

Some durable workflow engines rebuild an execution by replaying its recorded history. In those systems, orchestration code must be deterministic: it cannot directly depend on a fresh clock reading, random value, network call, or LLM result. The engine records those observations as step results and reuses them during replay.

This is an implementation model, not a requirement for every durable workflow system. The general rule is the same: persist the observations on which future steps depend, and isolate side effects behind explicit, retry-safe boundaries.

### Versioning Running Workflows

Treat a workflow definition as a long-lived contract. When changing its states or input shape, keep a compatible path for executions already in progress, migrate them deliberately, or allow the previous version to finish. Deleting a state that thousands of live executions still expect turns a normal deployment into a recovery incident.

## References

- [Microsoft: Durable Functions overview](https://learn.microsoft.com/en-us/azure/durable-task/durable-functions/durable-functions-overview)
- [Microsoft: Handle external events in Durable Task](https://learn.microsoft.com/en-us/azure/durable-task/common/durable-task-external-events)
- [Temporal: Workflow Execution overview](https://docs.temporal.io/workflow-execution)
- [AWS: Idempotency and retries for durable execution](https://docs.aws.amazon.com/durable-execution/patterns/best-practices/idempotency/)
