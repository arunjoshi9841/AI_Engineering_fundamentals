# Agents

[Structured Outputs and Tool Calling](../tools/structured-outputs-and-tool-calling.md) established that an LLM can propose actions while the application retains control of execution. An agent adds a loop: it decides what to do next after seeing the result of its previous action.

## The Problem

Some tasks cannot be expressed as one LLM call or a fixed sequence of API calls. Consider investigating a production incident: the next useful step depends on what the first log query reveals. The system may need to inspect an alert, retrieve a runbook, query metrics, and stop to ask a human for approval.

A fixed workflow can encode known paths. An agent is useful when the system must choose a path at runtime from a bounded set of tools and rules.

## Mental Model

An agent is a controlled observe-decide-act loop.

```text
Task + state + allowed tools
            ↓
        Choose next step
            ↓
       Execute through application
            ↓
      Observe tool result
            ↓
      Stop or continue safely
```

The loop gives the model flexibility. The application provides the boundaries: state, tool access, budgets, stopping conditions, and approvals.

## How It Works

1. Start with a task, user identity, relevant context, allowed tools, and execution limits.
2. Ask the LLM for the next action or a final answer using structured output or a tool call.
3. Validate and execute the proposed action through the application.
4. Add the tool result and updated task state to the next turn.
5. Continue until the task succeeds, the agent needs human input, a limit is reached, or no safe next step exists.

```text
Observe → decide → act → observe
                         ↓
                  final answer or stop
```

Persist observable state such as tool calls, results, approvals, and task status. Do not depend on an unbounded chat transcript as the only record of what happened.

## Important Concepts

### LLM Call, Workflow, Agent, and Multi-Agent System

These terms describe different control structures.

| Structure | Who chooses the next step? | Use it when |
| --- | --- | --- |
| LLM call | Application code | One response is enough |
| Workflow | Application code, using a fixed path | The steps are known and repeatable |
| Agent | Model within application limits | The next step depends on intermediate results |
| Multi-agent system | An orchestrator and several agents | Work can be meaningfully delegated or parallelized |

Do not call a fixed sequence of prompts an agent merely because it uses an LLM. A workflow is usually easier to test, faster, cheaper, and more predictable.

### State, Planning, and Execution

Agent state is the durable information needed to continue safely: task status, selected evidence, completed tool calls, tool outputs, approvals, and remaining budget.

The agent may make a short plan, but it should update that plan after real observations. A plan based only on the initial request can become wrong after the first tool result.

Execution remains application-owned. The agent chooses from predefined actions, but the application validates each action and records its outcome, as described in [Structured Outputs and Tool Calling](../tools/structured-outputs-and-tool-calling.md).

### Stopping Conditions

Every agent needs explicit stopping conditions:

- the task's success criteria are met
- a required fact or tool result is unavailable
- human approval or clarification is required
- the maximum number of steps, time, cost, or tool calls is reached
- an unsafe or repeated action is proposed

Without these limits, an agent can loop, accumulate cost, or keep retrying a problem it cannot solve.

### Tool Selection and Handoffs

Tool selection is the decision to use one approved capability rather than another. Make that decision easier with narrow tools and clear descriptions.

A handoff transfers a task to a specialist with a smaller toolset or different instructions, such as routing a billing question to a billing agent. The handoff must pass only the necessary state and preserve the user identity, permissions, and audit trail.

### When Multiple Agents Help

Multiple agents can help when work is genuinely independent, needs separate specialized contexts, or benefits from parallel review. A common pattern is an orchestrator that decomposes work and workers that return structured results.

They add coordination cost, more failure paths, and more context transfer. Start with one agent or a workflow. Add a second agent only when evaluation shows a specific bottleneck that specialization or parallelism solves.

## Where It Fits

An agent sits above tools, retrieval, and other application services. It is not a replacement for their safety checks.

```text
User task
     ↓
Agent loop
 ├── retrieve evidence
 ├── call approved tools
 ├── maintain task state
 └── request approval when needed
     ↓
Final answer or completed workflow
```

The next topic, Context Engineering and Memory, explains how to keep the agent's finite context useful across longer tasks.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Fixed workflow | Predictability and testability | Cannot adapt to unexpected intermediate results |
| Single agent | Flexibility with limited coordination | Can still loop or choose poor tools |
| Multi-agent system | Specialization or parallel work | Higher latency, cost, and coordination complexity |
| Larger step budget | More room for complex tasks | More latency, cost, and failure surface |
| Human approval | Safety for high-impact actions | Slower completion |

## What Can Go Wrong

**Looping.** The agent repeats an unsuccessful action without a meaningful new observation.

**Wrong tool choice.** It selects a plausible tool that cannot answer the task or has an unsafe scope.

**State loss.** A restart forgets which actions completed and repeats side effects.

**Premature success.** The agent stops after finding a plausible answer without checking the task's success criteria.

**Unbounded delegation.** Subagents create more work and context than they resolve.

**Authority escalation.** A handoff or broad tool gives a subtask more access than the original user had.

## Example

An incident-response agent receives an alert that checkout errors increased. It retrieves the relevant runbook, queries error metrics, and checks the latest deployment. The metric result shows only one region is affected, so it requests that region's logs rather than running a global investigation.

It summarizes the evidence and proposes a rollback. The application requires an on-call engineer to approve the rollback. After approval, a separate controlled workflow performs the deployment action and returns its result for the agent to communicate.

## Interview Takeaways

- An agent is a bounded loop that chooses the next action from observations; it is not simply an LLM with tools.
- Workflows are better when the path is predictable. Agents are useful when the path depends on intermediate results.
- State, stopping conditions, and tool limits are required for reliable agent execution.
- The application still owns validation, authorization, idempotency, and side effects.
- Multi-agent designs should justify their coordination cost with a concrete need for specialization or parallelism.

## Next

Next: [Context Engineering and Memory](context-engineering-and-memory.md). It explains how to manage finite context and durable state across longer agent tasks.

## References

- [Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models*](https://arxiv.org/abs/2210.03629)
- [Anthropic, *Building Effective AI Agents*](https://www.anthropic.com/engineering/building-effective-agents)
- [Liu et al., *AgentBench: Evaluating LLMs as Agents*](https://arxiv.org/abs/2308.03688)
