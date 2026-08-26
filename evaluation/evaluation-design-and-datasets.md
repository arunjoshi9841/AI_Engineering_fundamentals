# Evaluation Design and Datasets

[Evals](evals.md) provide the feedback loop. This page focuses on the part that determines whether the loop is trustworthy: choosing cases and graders that reflect the work the system must actually do.

## The Problem

It is easy to create an eval that reports an impressive number and does not predict user experience. The cases may be too easy, the expected answer may be ambiguous, or the grader may reward a superficial pattern rather than real success.

Bad evals are dangerous because they create false confidence. A team can spend weeks optimizing a prompt for the score while moving no closer to a useful product.

## Mental Model

Treat an evaluation dataset like a focused test plan for an uncertain system.

```text
Real job to be done
       ↓
Representative scenarios and edge cases
       ↓
Expected outcome or scoring rubric
       ↓
Grader validated against expert judgment
```

The goal is not a giant benchmark. It is a small suite where every case corresponds to an important user need, risk, or known failure.

## How It Works

1. **Write the success criterion first.** Describe the outcome, constraints, and what counts as failure before changing the prompt or model.
2. **Map the task space.** Identify common requests, high-value cases, risky boundaries, different user groups, and realistic bad inputs.
3. **Create cases from evidence.** Prefer anonymized production examples and expert-authored cases. Add historical incidents and edge cases deliberately.
4. **Specify what is observable.** Store the input, necessary context or environment, expected outcome, and relevant metadata such as task type and risk level.
5. **Choose and test a grader.** Use a deterministic check when possible. For language judgment, write a narrow rubric and compare grades with domain experts.
6. **Version and improve the suite.** Keep a development set for iteration and a held-out set for final comparison. Add reviewed production failures without silently changing old expectations.

## Important Concepts

### A Case Needs More Than a Prompt and Answer

For an AI system, expected behavior may include retrieved sources, allowed tools, an initial database state, or a policy version. A useful case can look like this:

```text
input: "Can I carry vacation into next year?"
context: employee is in France; policy version is 12
expected: answer applies the France carryover rule and cites section 4.2
must not: use another region's policy or invent a rule
slice: leave-policy / regional / high-visibility
```

This form makes the expected outcome testable without forcing one exact sentence.

### Representative Cases and Slices

Start from user value and failure risk, not from whatever data is easiest to collect. Include ordinary cases, but deliberately cover boundaries:

- ambiguous or incomplete requests
- uncommon but high-impact tasks
- permission and policy boundaries
- stale, missing, or conflicting data
- languages, regions, or customer segments that matter to the product

A **slice** groups cases that share one characteristic, such as "French leave policy" or "requests requiring escalation." Slices show whether a broad average hides a weak group.

### Grading Without Exact String Matching

Exact matching works for a classification label, a calculation, or a database state. It is usually wrong for open-ended language. Two useful answers can use different words, and a polished answer can still be unsupported or incomplete.

For flexible outputs, separate the dimensions in a rubric. For example: "Does the answer state the correct carryover limit? Does it name the correct region? Does each claim follow from the provided policy?" Small, observable checks are easier for a human or LLM judge to apply consistently than one vague instruction to grade quality.

Use an LLM judge only after sampling its decisions beside expert decisions. Inspect disagreements and revise the rubric, evidence, or judge prompt. A judge that cannot determine the answer should be allowed to return "unknown," not forced to guess.

### Development, Holdout, and Regression Sets

The **development set** is the set you read while improving the system. The **holdout set** is kept separate for an honest comparison after changes. If the team repeatedly tunes to every holdout failure, it becomes a development set too.

A **regression set** preserves cases that once failed in production or during testing. It protects a repair from being undone later. Keep source, review date, and expected behavior with each case so the suite remains understandable as policies and products change.

## Where It Fits

Evaluation design comes before specialized measurement. It supplies the cases and grading rules that [RAG evaluation](rag-evaluation.md) and [agent evaluation](agent-evaluation.md) need.

```text
User work and production failures
              ↓
     Versioned cases and rubrics
              ↓
  RAG, agent, or response evaluation
              ↓
   Failure review and better cases
```

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Small expert-curated set | Clear, high-value signal | Misses long-tail behavior |
| Production-derived data | Realism | Privacy, labeling, and staleness work |
| Synthetic cases | Fast coverage of rare situations | Can encode unrealistic assumptions |
| Exact expected answer | Cheap deterministic grading | Rejects valid variations |
| Rubric-based grading | Fits nuanced language | More design and calibration work |
| Large development set | Faster iteration | Easy to overfit without a holdout set |

## Failure Modes

- **Ambiguous success criterion:** Reviewers or judges disagree because the task never defined what good means.
- **Data leakage:** The same or near-duplicate cases appear in both development and holdout sets.
- **Stale truth:** A policy changes but its expected answer does not.
- **Grader drift:** A judge prompt or model changes and scores are compared as though they were identical.
- **Missing risk slice:** A rare but costly category has no cases, so the score says nothing about it.
- **Metric gaming:** The system learns to satisfy the grader's wording without solving the user's problem.

## Example

A travel-support team evaluates an assistant that answers baggage-policy questions. It samples resolved conversations from the last quarter, removes personal data, and groups them by airline, route type, loyalty status, and disruption scenario. Experts write expected rules and mark cases where the assistant must ask a clarifying question rather than answer.

The team keeps recent cases in the development set and a separate reviewed holdout set for release decisions. A model judge grades explanation quality against a rubric, while a deterministic checker verifies the airline and policy version. A human auditor reviews a sample of judge disagreements each week.

## Interview Takeaways

- Start an eval with a user outcome and a failure definition, not a favorite metric.
- A good case includes the context and constraints that make the expected behavior meaningful.
- Use task slices to expose weak groups that an average score hides.
- Exact matching is appropriate only when the output has one valid form; use narrow rubrics for flexible language.
- Separate development and holdout sets, and preserve production incidents as regression cases.

## Next

Next: [Evaluating RAG Systems](rag-evaluation.md). It separates evidence retrieval from answer generation so failures can be fixed at the right layer.

## References

- [Anthropic, *Demystifying evals for AI agents*](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [OpenAI, *Inside our in-house data agent*](https://openai.com/index/inside-our-in-house-data-agent/)
