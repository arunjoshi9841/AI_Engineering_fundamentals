# Evaluating Agents

[Agents](../agents/agents.md) choose actions over several steps. [Long-Running and Durable AI Workflows](../workflows/long-running-and-durable-ai-workflows.md) preserve those steps across waits and restarts. Agent evaluation tests whether the whole loop reaches the right outcome while obeying its limits.

## The Problem

An agent can produce a convincing final message while doing nothing useful. It can say a refund was issued when no refund exists, call the wrong tool, loop until its budget is exhausted, or take an unsafe action on the way to a correct answer.

The answer text is therefore only one observation. An agent eval needs a task, controlled tools and starting state, a trace of what happened, and a way to inspect the final environment.

## Mental Model

An agent attempt creates both a trajectory and an outcome.

```text
Task + controlled environment
            ↓
      Agent and tool loop
            ↓
Trace: decisions, tool calls, results, cost
            ↓
Outcome: final records and user-visible result
            ↓
Grade outcome first, then important constraints
```

The **trajectory** explains how the agent acted. The **outcome** is what is true afterward. For a booking agent, "your booking is confirmed" is text in the trajectory; a reservation in the test database is the outcome.

## How It Works

1. **Define a controlled task.** Provide the user request, starting data, tools, permissions, and success criteria. Reset the environment between trials.
2. **Specify outcome checks.** Prefer executable checks of the final database, file, API state, or test result over judging the final prose alone.
3. **Specify hard constraints.** Define actions that must never occur, such as reading another tenant's record, spending above a limit, or skipping required approval.
4. **Run multiple trials.** Model choices and tool timing can vary. Record task success, constraint failures, latency, cost, and number of steps for each trial.
5. **Inspect traces for failed or risky cases.** Use the trace to distinguish a bad tool description, missing context, loop, authorization denial, or grader problem.
6. **Improve the system and suite together.** Add a reviewed failure as a regression case, but do not require one exact tool sequence when different safe paths can reach the same outcome.

## Important Concepts

### The Harness Is Part of What You Evaluate

An agent is not only the model. Its harness supplies instructions, tool definitions, permissions, context selection, stopping conditions, and retries. Changing any of those can change performance.

Record these dependencies with each eval run. A comparison is misleading if one run has a better tool description, different retrieval data, or a larger step budget without saying so.

### Outcome Grading Before Trajectory Grading

When possible, grade what changed in the environment. A support agent succeeds when the correct refund record exists, not when it claims to have created one.

Do not require a single exact sequence of valid tool calls. Agents can find a safe path the eval author did not predict. Grade the trajectory only where the path itself matters: forbidden tools, missing approval, prohibited data access, destructive retries, or unacceptable cost and latency.

### Scenarios and Simulated Environments

An agent needs an environment that can respond to its actions. For safe testing, use a sandbox, fake service, or resettable copy of the relevant state. Include ordinary work and adversarial conditions:

- a tool returns a temporary error
- a needed record is missing or ambiguous
- a user lacks permission
- one of two similar records is the correct target
- an approval is delayed or denied

The simulation should be realistic enough that passing requires solving the actual task, not exploiting an easy test shortcut.

### Traces, Limits, and Partial Success

Keep a trace of messages, tool calls, results, retries, state transitions, and final outcome. It is the main debugging record for agent behavior.

Measure limits alongside success: step count, tool-call count, latency, token use, and cost. A task can be partly successful, for example identifying the right account but stopping before the refund. Partial credit helps diagnose progress, but hard safety constraints should still fail the trial.

## Where It Fits

Agent evaluation builds on [Evals](evals.md). It also validates the boundaries described in [Structured Outputs and Tool Calling](../tools/structured-outputs-and-tool-calling.md): the model proposes, while the application authorizes and executes.

```text
Versioned task and sandbox
             ↓
Agent harness + model + tools
             ↓
Trace and final environment state
             ↓
Outcome checks, safety checks, and failure review
```

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Final-state checks | Objective evidence of task success | Requires a controllable environment |
| Trace review | Diagnosis of loops and unsafe choices | Expensive at high volume |
| Simulated tools | Safe repeatable trials | Can differ from production behavior |
| Multiple trials | Measures reliability under variation | More model and tool cost |
| Exact trajectory matching | Easy automation | Brittle and rejects valid solutions |
| Hard safety gates | Prevents unacceptable behavior | Can lower headline success rate |

## What Can Go Wrong

**Text-only grading.** The agent says it completed work but the final system state proves otherwise.

**Leaky sandbox.** The agent can pass by reading test-only hints or using capabilities unavailable in production.

**Unreset environment.** One trial changes state for the next, making results non-reproducible.

**Path overfitting.** A grader rejects a safe novel approach because it expected one tool order.

**Safety blind spot.** The suite tracks task success but never checks authorization, approvals, or destructive actions.

**Trace ignored.** Repeated loops and high-cost failures remain invisible behind an aggregate score.

## Example

A support agent receives: "Refund my delayed order." The sandbox contains several orders, the customer's permissions, refund policy, and tools for lookup and refund creation. The success grader checks that the eligible order has exactly one refund record for the allowed amount.

The agent may look up the customer before the order or the order before the policy. Both paths can pass. It fails if it refunds a different customer's order, refunds twice after a timeout, or creates a refund without the required approval. The trace shows whether a failure came from a wrong lookup, a confused tool description, or an unsafe retry.

## Interview Takeaways

- Agent evals require tasks, tools, a controlled environment, traces, and checks of final state.
- Grade the outcome whenever possible; final prose is not proof that a side effect happened.
- The agent harness, tools, permissions, budgets, and model together form the system under test.
- Do not demand one exact valid trajectory, but enforce hard constraints on unsafe paths.
- Run multiple trials and inspect traces to diagnose loops, cost, tool misuse, and grader mistakes.

## Next

Next: [Reliability Patterns for AI Systems](../production/reliability-patterns.md). It turns known failure modes into timeouts, retries, fallbacks, and operational safeguards.

## References

- [Anthropic, *Demystifying evals for AI agents*](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [Liu et al., *AgentBench: Evaluating LLMs as Agents*](https://arxiv.org/abs/2308.03688)
- [OpenAI, *Inside our in-house data agent*](https://openai.com/index/inside-our-in-house-data-agent/)
