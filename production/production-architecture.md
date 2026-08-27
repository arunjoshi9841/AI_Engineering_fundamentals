# Production Architecture for AI Systems

[LLM Inference and Serving](../inference/llm-inference-and-serving.md) explains model-serving capacity. This page combines it with retrieval, tools, workflows, security, and operations.

## The Problem

An AI demo can accept a question, call a model, and return text. A production system must also apply permissions before reading private knowledge, keep indexes current, prevent unauthorized actions, continue long work safely, and diagnose regressions.

The answer is not maximum separation. It is clear ownership: a component owns a security decision, external side effect, derived index, or durable task record. Those boundaries can begin inside one deployable application.

## Mental Model

Think of a production AI system as three connected paths:

```text
Live request path                 Data update path
User → application → answer      Source change → ingestion → index
                 ↓                         ↑
        retrieval, model, tools   authorized retrieval uses current index

Release and operations path
versions, evals, policies, traces, alerts, rollback
```

The **live request path** serves one task. The **data update path** keeps derived data current. The **release and operations path** controls changes and measures whether the system meets its promises.

## How It Works

1. **Accept and classify.** Authenticate the user, apply rate and budget limits, start a trace, and choose a short response or durable workflow.
2. **Build permitted context.** Apply tenant and document permissions before retrieval, then enforce a context budget. The index is not the source of truth.
3. **Run the model step.** Select an eligible model, send a bounded prompt, and validate the result against the feature's requirements.
4. **Mediate actions.** Validate and authorize every proposed tool call. Require approval for consequential actions.
5. **Return or continue.** Stream a short task with a terminal state. Persist a task that waits for approval, a callback, or background work.
6. **Change and operate deliberately.** Version, evaluate, observe, and roll back changes to prompts, models, indexes, tools, and policies.

## Important Concepts

### Boundaries Are Ownership, Not Necessarily Network Calls

The application or **orchestrator** owns the task lifecycle. Other boundaries have narrow jobs:

- the knowledge boundary owns extraction, indexing, and permission-aware retrieval
- the model boundary owns model eligibility, serving, and usage limits
- the tool boundary owns argument validation, authorization, and external side effects
- the workflow boundary owns durable state, waits, retries, and repair

A boundary can be a module, library, process, or service. Split it when independent scaling, isolation, ownership, or failure behavior justifies the added operational cost.

Each boundary needs a clear contract. Retrieval returns authorized evidence and its version; tools return recorded outcomes; workflow steps consume and record durable state. These contracts make tests, access control, retries, and diagnosis possible.

### The Source of Truth Is Separate From the Serving Index

A retrieval index is derived data optimized for search. The document store, business database, or external system remains authoritative for facts and permissions.

The request path reads the index quickly while the update path keeps it current. Avoid synchronous chunking and embedding after every edit unless the product requires it. Instead, set a freshness expectation and run reconciliation. [Index Maintenance and Synchronization](../ingestion/index-maintenance-and-synchronization.md) covers the update mechanics.

### Synchronous Work and Durable Work Need Different Homes

An interactive question has a short deadline and bounded retrieval, model, tool, and streaming budgets. A research task, approval flow, bulk job, or callback belongs in a durable workflow that can wait and resume safely.

A queue is not a reliability layer by itself. A job still needs an owner, task ID, deadline, retry rules, idempotent side effects, and a result or repair state. An HTTP connection is not workflow state.

### The Control Loop Is Part of the Architecture

Prompts, models, indexes, tool schemas, policies, and serving configuration are versioned inputs to behavior. Before widening a release, compare it with a baseline using [evals](../evaluation/evals.md). After release, use [observability](observability-for-ai-systems.md) to connect outcomes to versions and request slices.

Targets should describe user-relevant behavior: grounded answers, completion before a deadline, safe tool outcomes, or index freshness. Define the signal, owner, and response in advance.

## Where It Fits

This is a synthesis. It reuses earlier boundaries rather than re-implementing them:

```text
Identity and policy
        ↓
Application / orchestration
  ├── Context: memory + authorized retrieval
  ├── Decision: model routing + inference
  ├── Action: validated tools + approval
  └── Continuation: durable workflow
        ↓
Response, task status, or validated side effect

Across every boundary: reliability, observability, evaluation, and security
```

For a read-only assistant, the tool and workflow branches may not exist. For an agent that changes business data, they are essential. Start small, then separate components as scale or risk requires.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| One deployable application with internal boundaries | Fast iteration and simple deployment | Independent scaling and isolation are limited as the system grows |
| Separate knowledge, tool, or inference services | Clearer ownership, targeted scaling, and stronger isolation | Network failures, deployment coordination, and more operations |
| Asynchronous indexing | Fast source writes and scalable ingestion | Eventual consistency and repair work |
| Synchronous request path | Immediate response for bounded work | Tight latency budget; poor fit for long waits or expensive fan-out |
| Durable workflow | Safe waits, recovery, and inspectable multi-step work | State modeling, versioning, and workflow operations |
| Central policy and tool gateway | Consistent authorization and audit | A critical dependency that needs availability and careful change control |
| Strong release gates and version records | Safer changes and faster rollback | Evaluation effort and slower rollout for high-risk changes |

## What Can Go Wrong

**The model becomes the integration layer.** It receives broad credentials or direct database access, so an incorrect plan becomes an unauthorized action.

**The index is treated as truth.** Deleted, stale, or reclassified content remains retrievable because no source-change, deletion, or repair path exists.

**Authorization happens after context assembly.** Another tenant's evidence reaches the model, even if the final answer tries to hide it.

**A long task runs in the request handler.** A timeout or restart loses progress, while retries repeat external actions.

**A queue has no workflow record.** Messages are retried, but no one can determine the task's final state or repair an uncertain side effect.

**Components change independently without a contract.** A tool schema, prompt, index, or model update silently breaks a downstream consumer.

**Everything is split too early.** A small product acquires distributed-system failure modes before it has a real scaling, isolation, or ownership need.

**Operations see only uptime.** The service is available while retrieval is empty, citations are wrong, or an expensive route harms a meaningful user slice.

## Example

An employee assistant answers travel-policy questions and can request access to an internal booking dashboard.

For a question, the application authenticates the employee, retrieves only permitted policy documents, selects a model, and streams a cited answer. When the policy team edits a document, ingestion updates the index separately. The source and its access rules remain authoritative.

For dashboard access, the model only proposes the action. A tool gateway checks the employee, dashboard, and policy. The request enters a durable approval workflow, which invokes provisioning with an idempotency key after approval.

Every path records safe identifiers for the prompt, model, index, tool, and workflow versions. The team evaluates a routing rule before release, then watches completion, answer quality, approval, and freshness by region. A regional regression can be rolled back without stopping unrelated workflows.

## Interview Takeaways

- Production architecture is about explicit ownership and contracts across the request path, data-update path, and operational control loop.
- Keep authoritative sources separate from derived retrieval indexes, and design both an update path and a repair path.
- Use a short, bounded request path for interactive work and a durable workflow for waits, callbacks, retries, and consequential multi-step work.
- The model proposes decisions; policy and tool boundaries authorize data access and external actions.
- Start with clear internal boundaries, then introduce separate services only when scaling, isolation, failure, or ownership needs justify them.

## Next

This closes the core handbook. Return to the [README](../README.md) to revisit a prerequisite or apply the ideas to a system of your own.

## References

- [NIST AI Risk Management Framework: Core and Profiles](https://airc.nist.gov/airmf-resources/airmf/5-sec-core/)
- [Google SRE Book: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [OpenTelemetry: Traces](https://opentelemetry.io/docs/concepts/signals/traces/)
- [Microsoft Learn: Design and Develop a RAG Solution](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide)
