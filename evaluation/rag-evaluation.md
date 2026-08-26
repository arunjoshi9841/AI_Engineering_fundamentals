# Evaluating RAG Systems

[Retrieval-Augmented Generation](../rag/retrieval-augmented-generation.md) combines retrieval and generation. An end-to-end answer score alone cannot tell you whether a failure came from missing evidence, poor context construction, or unsupported generation.

## The Problem

A RAG assistant can sound confident while missing the document that answers the question. It can also retrieve the correct paragraph and still misread it, omit an exception, or cite an unrelated source.

If every failure is called a hallucination, teams change prompts when they should fix ingestion or retrieval. RAG evaluation must preserve the boundary between evidence selection and answer generation.

## Mental Model

Evaluate the pipeline at two checkpoints and then as a whole.

```text
Question
   ↓
Retrieved candidates ──> Did the needed evidence arrive?
   ↓
Final context ─────────> Is it focused, current, and permitted?
   ↓
Answer and citations ──> Is the answer correct and supported?
```

The needed passage is the bridge between these checks. If it is absent, the generator never had a fair chance to answer from evidence.

## How It Works

1. **Create grounded questions.** For each test case, record the question, source snapshot, necessary passage or document, expected claims, and any permission or freshness conditions.
2. **Run retrieval separately.** Inspect the candidate list before generation. Measure whether it contains the necessary evidence at the chosen retrieval depth.
3. **Inspect final context.** Check whether reranking and context construction retained the useful evidence without filling the prompt with distracting material.
4. **Grade the answer.** Check correctness, completeness, and whether individual claims follow from the supplied context.
5. **Check citations and constraints.** Validate that citations identify the passages that actually support the claims, and that no unauthorized or stale content was used.
6. **Diagnose by stage.** Change the component responsible for the failure, then rerun the same case as a regression test.

## Important Concepts

### Evidence Recall

For a question with known necessary evidence, ask: "Did the retrieved candidates include a passage that could support the answer?" The share of cases where this happens is often called retrieval recall or hit rate at a selected depth, such as the top 10 candidates.

This measure is not enough by itself. A large candidate list can increase the chance of finding the passage while making the final context noisier and more expensive. Evaluate the final selected context as well as the initial candidates.

### Context Relevance and Grounding

**Context relevance** asks whether the passages given to the LLM are useful for the question. **Grounding** or **faithfulness** asks whether the answer's claims can be supported by those passages.

These are separate. A focused context can still be misread. A correct answer can appear next to irrelevant context by coincidence. Grade important claims against the actual supplied text, not against the model's confidence or a citation that merely looks plausible.

### Answer Correctness and Completeness

Correctness asks whether the answer's claims are true for the user and source version. Completeness asks whether it included the important qualification, exception, or next step that the question requires.

Sometimes the correct answer is abstention: "The available sources do not establish this." Include such cases in the suite. A system that refuses when evidence is missing can be safer than one that guesses fluently.

### Citations, Permissions, and Freshness

A citation should map to a source ID that the application can verify. Grade whether the cited text supports the nearby claim, not only whether a citation exists.

Also test non-quality constraints. A response with the right answer fails if it exposes a restricted document, relies on a superseded policy, or returns a source the user cannot open.

## Where It Fits

RAG evaluation uses the cases and graders from [Evaluation Design and Datasets](evaluation-design-and-datasets.md). Its results point back to retrieval, ingestion, or prompt construction rather than treating RAG as one opaque model call.

```text
Source and expected evidence
             ↓
Ingestion and retrieval evaluation
             ↓
Context and answer evaluation
             ↓
Fix the failing pipeline stage
```

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Labeled necessary passages | Clear retrieval diagnosis | Annotation effort and changing-source maintenance |
| End-to-end answer grading | Measures user-visible result | Hides the failing component |
| Claim-level grounding checks | Detects unsupported synthesis | More detailed grading work |
| More retrieved candidates | Better evidence recall | More noise, token cost, and latency |
| Reference-free judge | Faster evaluation when answers vary | Must be calibrated with human review |

## Failure Modes

| Symptom | Likely cause | First place to investigate |
| --- | --- | --- |
| Necessary passage absent | Ingestion, filters, query handling, or retrieval recall | Source coverage and candidate retrieval |
| Passage retrieved but absent from prompt | Reranking or context-budget decision | Final-context selection |
| Passage present but answer is unsupported | Generation or answer instruction | Claim-level grounding and prompt behavior |
| Correct answer but wrong citation | Source metadata or citation mapping | Citation construction and validation |
| Good answer leaks data | Permission filter not enforced before context | Retrieval authorization path |

## Example

An employee asks whether unused vacation carries into the next year. The eval case records the employee's region, employment type, and the required policy section. Retrieval must find that section. The final context must retain its carryover limit and exception. The answer grader checks the limit, exception, and citation support.

If retrieval finds only a global policy while the case requires a regional exception, this is a retrieval failure. If the regional section is in context but the answer leaves out the exception, it is a generation failure. The different diagnosis leads to a different fix.

## Interview Takeaways

- Evaluate RAG retrieval and generation separately, then measure the end-to-end result.
- Necessary-evidence recall shows whether the model had a chance to answer from the source.
- Context relevance, grounding, correctness, completeness, and citation support are different checks.
- Test abstention, permissions, and freshness alongside answer quality.
- Diagnose the stage that failed before changing embeddings, prompts, or rerankers.

## Next

Next: [Evaluating Agents](agent-evaluation.md). Agents add changing environments, tool side effects, and multi-step traces to the evaluation problem.

## References

- [Es et al., *RAGAS: Automated Evaluation of Retrieval Augmented Generation*](https://arxiv.org/abs/2309.15217)
- [Yu et al., *Evaluation of Retrieval-Augmented Generation: A Survey*](https://arxiv.org/abs/2405.07437)
