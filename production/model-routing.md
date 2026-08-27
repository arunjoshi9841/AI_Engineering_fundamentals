# Model Routing

Caching removes work that does not need a new model call. Model routing makes the calls that remain proportionate to the request. It chooses from an approved set of models so a routine task does not always pay the latency and cost of the most capable option.

## The Problem

One product can handle very different requests. Extracting an invoice number, summarizing a note, answering a policy question, and planning a tool workflow do not need the same capability. Sending every request to one strong model is simple, but expensive and sometimes slow. Sending everything to a small model is cheaper, but the hard cases become user-visible failures.

Model routing makes that tradeoff deliberately for each request or workflow step. It aims to meet the application's quality, latency, cost, and policy requirements for a known class of work.

## Mental Model

Think of a router as a dispatcher with a small, approved menu. It removes models that cannot legally or technically handle the request, then chooses a remaining option likely to meet the service target.

```text
Request
  ↓
Eligibility: region, modality, context, tools, policy
  ↓
Routing policy: task class, risk, budget, current health
  ↓
Selected model and version
  ↓
Validate outcome → return, escalate, or repair
```

Judge the router by the whole path. A low-cost route that damages answer quality is a regression.

## How It Works

1. **Define a baseline.** Choose a model that meets the product requirement, then measure its quality, cost, and latency with representative [evals](../evaluation/evals.md).
2. **Create an approved pool.** Filter candidates before any cost decision: data location, input and context support, output and tool contracts, safety policy, and availability.
3. **Classify the request or step.** Use stable signals such as endpoint, workflow step, input size, language, risk class, and task type. Do not call a request “simple” just because it is short.
4. **Apply a policy.** Start with understandable rules, such as a fast model for extraction and a stronger model for ambiguous analysis. A learned router can later estimate the likely outcome, inside the same pool and budget.
5. **Validate and escalate.** A schema, evidence, business rule, or human-review requirement can reject an inadequate result. Send it to a stronger model or stop safely, within a separate escalation budget.
6. **Record and compare.** Trace the policy version, eligible pool, selected model, outcome, tokens, latency, cost, and escalation reason. Compare the routed system with the baseline by important traffic slices before rollout.

```text
Support request
  ├── policy or capability mismatch → reject before model call
  ├── account lookup                → fast compatible model
  ├── routine explanation           → standard model
  └── ambiguous, high-impact case   → stronger model or human path
```

## Important Concepts

### Hard Constraints Come First

Some decisions are not tradeoffs. A model that cannot accept the input, fit the context, produce the required schema, keep data in an allowed location, or call the needed tools is ineligible. It must not become an option merely because a router scored the prompt as easy.

For consequential work, route on the *workflow step*, not only the user's message. A short request such as “approve this” may invoke a high-impact action and need stricter validation or a human path. Routing does not replace those controls.

### Static Rules, Learned Routers, and Cascades

**Static routing** uses explicit rules. For example, classification uses a smaller model and contract analysis a stronger one. It is easy to test and roll back.

A **learned router** estimates which eligible model will satisfy a prompt. This can help with mixed workloads, but its prediction can drift or have blind spots. It needs its own evaluation, versioning, and fallback policy.

A **cascade** starts with a lower-cost model and escalates only when there is evidence that the result is insufficient. Good signals are externally checked: invalid structured output, missing required citations, a policy threshold, or reviewer decision. A model's own confidence is not proof that the answer is correct.

Start with static rules unless the workload clearly justifies a learned router.

### Routing Is Not Failover

Routing is a planned choice before a request runs. **Failover** responds when the chosen provider or model is unavailable.

Do not fail over to a model that violates hard constraints or the response contract. A substitute that cannot use the needed tools or format can turn a provider outage into an application failure. Treat unknown external side effects using [Reliability Patterns for AI Systems](reliability-patterns.md), not by silently repeating an agent task.

### Evaluation and Feedback

Measure quality, cost, and latency together against the baseline. Break results down by task type, language, input length, tenant, risk level, and model selected. A good global average can hide a route that fails one valuable group.

Production feedback has a blind spot: for a request sent only to the small model, you do not observe how the larger model would have performed. Use representative offline comparisons, sampled shadow runs where policy allows, and reviewed failures to update the policy. A lower escalation rate is not automatically better quality.

## Where It Fits

Model routing sits between application policy and model providers. Observability and evals determine whether its choices remain justified.

```text
Application or agent step
          ↓
Policy and eligibility filter
          ↓
Model router ───→ compatible model pool
     ↓                    ↓
Route record  ←──── outcome, cost, latency
     ↓
Evals, dashboards, and controlled policy updates
```

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| One fixed model | Simplicity and reproducibility | Pays for capacity that routine requests may not need |
| Static rules | Predictable behavior and easy debugging | Rules become stale as traffic and models change |
| Learned router | Can adapt to diverse requests | Requires training data, evaluation, and drift monitoring |
| Cascade with validation | Low average cost while preserving a stronger path | Extra latency and cost on escalated requests |
| Broad model pool | More options for price and availability | More compatibility testing and governance |

## Failure Modes

- **Routing before eligibility:** A low-cost model receives a request it cannot safely process.
- **“Simple” defined by prompt length:** A short legal, financial, or action-triggering request is routed below its required quality or control level.
- **Unvalidated cascade:** The inexpensive model gives an incorrect answer that looks well formed, so escalation never happens.
- **Optimization for the average:** Cost falls overall while one language, task, or customer tier has a sharp quality regression.
- **Incompatible substitutions:** Model-specific prompts, schemas, or tool behavior change the application after a route change.
- **Unbounded escalation:** A difficult request calls several models, misses the user deadline, and costs more than the original baseline.
- **Incomplete feedback:** A router learns only from routes it chose and reinforces a bad preference for a cheaper model.

## Example

A support product has three approved models. It handles many routine “how do I?” questions, some long account histories, and a few disputed account changes.

Its first policy is deliberately small: a fast model extracts intent and fields, a standard model writes routine explanations, and a stronger model handles long histories or ambiguous policy analysis. Account changes still pass authorization and explicit confirmation. They are not made safe by choosing a stronger model.

The team evaluates this policy against its former single-model baseline. Overall cost drops, but non-English account histories have lower citation accuracy. The team sends that slice to the stronger model, then re-evaluates. The goal is a service target, not the highest percentage of cheap requests.

## Interview Takeaways

- Model routing selects an eligible model for a request or workflow step to balance quality, cost, latency, and policy requirements.
- Filter hard constraints before optimizing. Compliance, context capacity, tool support, and response contracts are not soft preferences.
- Start with explicit rules; use learned routing only when its extra complexity is justified and evaluated.
- A cascade needs independent evidence to escalate. A valid-looking answer or self-reported confidence is not enough.
- Compare routed behavior with a fixed baseline by important traffic slices, and record every routing decision for diagnosis and rollback.

## Next

Next: [Streaming AI Responses](streaming-ai-responses.md). Streaming improves perceived responsiveness, but makes response delivery, cancellation, and partial results part of the system design.

## References

- [FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance](https://arxiv.org/abs/2305.05176)
- [RouteLLM: Learning to Route LLMs with Preference Data](https://arxiv.org/abs/2406.18665)
- [Microsoft Learn: Choose the Right AI Model for Your Workload](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/choose-ai-model)
- [Microsoft Learn: Evaluate Model Router for Your Workload](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/evaluate-model-router)
