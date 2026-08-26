# Evals: Measuring AI Systems

[Long-Running and Durable AI Workflows](../workflows/long-running-and-durable-ai-workflows.md) make AI work recoverable. Evals tell you whether that work is useful, safe, and still behaving as intended after a prompt, model, retrieval, or workflow change.

## The Problem

An AI demo can look excellent on a few hand-picked prompts and still fail for ordinary users. A change that improves one answer can quietly damage another. Model output also varies between runs, so an engineer's intuition after trying a feature once is weak evidence.

Traditional tests are still necessary, but they do not fully answer questions such as:

- Did the answer address the customer's actual question?
- Did a RAG system use evidence that supports its claims?
- Did an agent complete the task without violating a safety rule?
- Did a model upgrade improve quality without breaking latency or cost targets?

An eval turns these questions into repeatable tests.

## Mental Model

An eval is an acceptance test for an AI system. It gives the system a realistic case, captures what it does, and applies grading logic to decide whether it succeeded.

```text
Representative task
       ↓
AI system configuration
       ↓
Output, tool trace, and system state
       ↓
Grader or human review
       ↓
Scores, failure examples, and a ship decision
```

The unit under test is the whole application configuration: model, prompt, retrieval, tools, and execution limits. A model is not "good" or "bad" in isolation from the system around it.

## How It Works

1. **Define success.** State the user outcome and constraints that matter, such as correct routing, supported claims, no unauthorized tool use, or a response under a latency target.
2. **Build representative cases.** Collect real tasks, expert-written edge cases, and known production failures. Give each case the inputs and environment the system needs.
3. **Run a fixed version of the system.** Record the model, prompt, retrieval configuration, tool versions, and relevant data snapshot so a result can be compared later.
4. **Grade the result.** Use a deterministic check where possible, a reference answer or rubric where needed, and human review for judgments that cannot be trusted to automation alone.
5. **Slice and inspect failures.** Compare performance by task type, language, customer segment, source, or risk level. Read the actual failures instead of relying only on an average score.
6. **Use the result to make a decision.** Keep a baseline, reject regressions, and add important new failures to the suite after fixing them.

```text
Production examples + expert cases
             ↓
          Eval suite
             ↓
Candidate change ──> run and grade ──> compare with baseline
             ↑                              ↓
             └──── add real failures ───────┘
```

## Important Concepts

### Test Case, Trial, and Evaluation Suite

A **test case** is one task with an input, any required context or environment, and a success criterion. One attempt to run it is a **trial**. An **evaluation suite** is a versioned collection of cases that measures a capability or behavior.

Multiple trials matter when output is variable. A single successful answer can hide a task that works only half the time. Run enough trials to see whether a change is reliably better, especially before using a system for consequential work.

### Graders

A grader scores one dimension of a result. Use the strongest available evidence:

| Grader | Good fit | Limitation |
| --- | --- | --- |
| Deterministic check | A valid JSON field, calculation, database state, or policy rule | Cannot judge nuanced language quality |
| Reference comparison | A bounded answer or known expected result | Can reject a correct answer phrased differently |
| Rubric or LLM judge | Helpfulness, completeness, or evidence support | Needs a precise rubric and calibration with people |
| Human review | High-stakes, subjective, or domain-specific judgment | Slower and more expensive |

An LLM judge is another model with its own error modes. Give it clear criteria and enough evidence, allow an "insufficient evidence" result, and regularly compare its grades with expert review.

### Quality Is More Than One Score

An average quality score can hide the failure that matters most. Track dimensions separately:

- task success or answer correctness
- safety and permission compliance
- latency, cost, and tool-call count
- grounding, citations, or other domain-specific requirements

Decide whether a dimension is a hard gate or a tradeoff. A customer-support assistant may allow a small drop in helpfulness for much lower latency. It should not allow any unauthorized refund.

### Offline Evals and Production Feedback

**Offline evals** run controlled cases before release. They are fast to repeat and essential for regression testing. **Production feedback** shows what real users and real data do after release: user corrections, escalations, abandoned tasks, and sampled traces.

Neither is enough alone. Offline cases may miss new behavior. Production signals are noisy and arrive after users feel the impact. Turn important production failures into reviewed offline cases, then use the suite to prevent recurrence.

## Where It Fits

Evals form a feedback loop around every AI component. They do not replace unit tests, authorization checks, or monitoring. They measure whether the assembled system produces the intended user outcome.

```text
Prompt, model, retrieval, tools, workflow
                 ↓
              AI system
                 ↓
      offline evals and production feedback
                 ↓
       improve, compare, and safely release
```

The next pages make the loop concrete for [RAG](rag-evaluation.md) and [agents](agent-evaluation.md).

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Small hand-written suite | Fast start and clear intent | Poor coverage and easy saturation |
| Real production cases | Relevance to actual users | Privacy review and annotation work |
| Automated grading | Fast iteration at scale | Grader errors or loopholes |
| Human review | Domain alignment and trust | Time, money, and limited throughput |
| One aggregate score | Easy comparison | Hides risk-specific regressions |
| Versioned suites and baselines | Reproducible decisions | Test-data and infrastructure discipline |

## Failure Modes

- **Demo-driven development:** The system is optimized for a few memorable prompts rather than representative work.
- **Brittle grader:** A valid answer fails because it differs from one exact expected string.
- **Weak proxy:** The suite measures style or lexical overlap instead of the user outcome that matters.
- **Unrepresentative data:** Easy, synthetic, or stale cases overstate quality.
- **Average hides a regression:** Overall score rises while a critical permission or safety slice fails.
- **Eval saturation:** Every case passes, so the suite can detect regressions but no longer distinguishes better designs.

## Example

A support assistant classifies incoming requests and proposes the next queue. The team creates cases from resolved tickets, including ambiguous billing questions, account-access requests, and messages that should be escalated. A deterministic grader checks the queue name. A reviewer checks whether the explanation cites the relevant policy when required.

When a prompt change improves the overall routing score but routes account-access cases to billing, the access slice fails. The team keeps the old prompt, investigates the failures, and adds those cases to the permanent regression suite.

## Interview Takeaways

- An eval is a repeatable test of an AI application, not a one-time model demo.
- Measure the full system configuration, including prompts, retrieval, tools, and execution environment.
- Prefer deterministic outcome checks; use rubric-based and LLM judging carefully and calibrate them with people.
- Track slices and hard constraints separately from average quality.
- Production failures should become reviewed regression cases in the offline suite.

## Next

Next: [Evaluation Design and Datasets](evaluation-design-and-datasets.md). It explains how to make test cases and graders representative enough to trust.

## References

- [Anthropic, *Demystifying evals for AI agents*](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [OpenAI, *Inside our in-house data agent*](https://openai.com/index/inside-our-in-house-data-agent/)
