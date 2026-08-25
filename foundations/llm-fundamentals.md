# LLM Fundamentals

At inference time, an LLM does not search a database for an answer or execute a plan. It repeatedly predicts the next token that best fits its input. The useful behavior we see emerges from doing that prediction very well across many steps.

## The Problem

Traditional software needs its inputs in a known format. A support system can route a ticket only after someone defines the fields and rules.

Language is not like that. A user might say, "My payment failed," "Why was I charged twice?" or describe the same problem in three paragraphs. We want one interface that can interpret the request, follow instructions, and produce a useful response without a separate rule for every phrasing.

An LLM is that flexible language interface. Its output is still a prediction, though, not a fact checked result or an action taken in the real world.

## Mental Model

Think of an LLM as extremely capable autocomplete.

Give it a sequence of text and instructions. It estimates what token should come next, adds one token, then makes the same decision again with the longer sequence. A convincing answer is a long chain of these small decisions.

```text
Instructions + conversation + user request
                 ↓
             Tokenize
                 ↓
     Predict a next-token distribution
                 ↓
           Choose one token
                 ↓
        Repeat until the response ends
```

This is a deliberately simple model. You do not need to understand attention mechanisms yet to reason about how to use an LLM in a system.

## How It Works

At inference time, a typical text request follows this sequence:

1. The application assembles the input: its instructions, the relevant conversation messages, and the user's request.
2. A model-specific tokenizer converts that input into token IDs.
3. The request must fit inside the model's context window, while leaving room for output tokens.
4. The model produces probabilities for possible next tokens, based on every token currently in context.
5. A decoding strategy selects one token. Sampling settings can make the choice more or less variable.
6. The selected token is appended to the context and the process repeats until the model reaches a stop condition or output limit.

The model's learned parameters remain fixed during this request. It can appear to learn from a conversation because earlier messages are included again in the current input, not because the API call changed the model.

### Tokens and Tokenization

A token is a unit of text the model processes, not reliably a word or a character. Uncommon names, source code, punctuation, and non-English text may break into several tokens.

Tokenization converts text into those units before inference and converts generated token IDs back into text afterward. Each model family has its own tokenizer, so a token estimate from one model is not exact for another.

For engineering, tokens control three important constraints:

- context capacity
- API cost
- generation latency

Count tokens with the tokenizer for the model you plan to use. Do not derive production limits from word counts.

### Prompts and Messages

A prompt is the information that steers a model's response. In practice it often includes more than the user's visible request: application instructions, examples, retrieved documents, and previous turns can all become part of the prompt.

Many APIs accept structured messages with roles such as system, developer, user, and assistant. Roles clarify intent and preserve a conversation, then are encoded into the model-specific input format.

Messages improve organization. They are not a security boundary. The application must still decide which user-provided content may enter the prompt and must enforce permissions outside the model.

### Context Windows

The context window is the maximum number of tokens the model can consider in one request. It normally includes input tokens and generated output tokens.

This makes context a finite budget:

```text
context window
├── instructions
├── conversation history
├── user request
├── supplied documents
└── reserved output space
```

More context can give the model useful evidence, but it also raises cost and latency. A long conversation is not persistent model memory. Your application chooses what history or external information to send on each call.

### Output Generation and Sampling

The model assigns a probability to each possible next token. Decoding turns that probability distribution into an actual token.

With a low-temperature setting, decoding favors the highest-probability options. This is useful when you want stable extraction or classification. A higher temperature allows less likely options more often, which can produce more varied writing but also more variation and mistakes.

Some systems also offer `top_p`, or nucleus sampling. It limits choices to a set of likely tokens whose combined probability reaches a threshold. Treat it as an alternative way to control variation unless a provider gives a reason to combine it with temperature.

Lower randomness does not make an answer correct, and it does not guarantee identical output across providers or model versions.

### Structured Outputs

Free-form prose is inconvenient when software needs a response it can process. Structured output asks the model to return a shape such as a JSON object that follows a schema.

For example, a support application may require:

```json
{
  "category": "billing",
  "urgency": "high",
  "needs_human_review": true
}
```

Schema-constrained output greatly reduces parsing failures. It does not prove that `category` or `urgency` is correct. Treat the result as untrusted input: parse it, validate it, then apply normal business rules.

## Where It Fits

An LLM is one component in an application. The application owns the input, the policies, and what happens after generation.

```text
User request
     ↓
Application assembles relevant context
     ↓
LLM generates text or schema-shaped data
     ↓
Application validates, stores, displays, or routes the result
```

This boundary matters. An LLM can classify a ticket or draft a reply. It should not be treated as the source of truth for account data, authorization, or business rules.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| More prompt context | More available instructions and evidence | More tokens, latency, cost, and irrelevant material |
| Lower temperature | More stable, focused outputs | Less variety and no guarantee of correctness |
| Higher temperature | More varied language | Less predictable output |
| Structured output | Easier integration and parsing | Schema design and validation work |
| Larger output limit | Room for complete answers | More latency and cost; still may ramble |

## Failure Modes

- **Confident but wrong output:** The model optimizes for a plausible continuation, not verified truth. Give it authoritative context where needed and evaluate important workflows.
- **Context overflow or truncation:** Important instructions or history may not fit. Budget tokens before sending the request.
- **Token surprises:** Code, tables, and multilingual text can be expensive despite a low word count.
- **Format is valid, meaning is wrong:** A schema-matching JSON response can still contain an incorrect classification or invented value.
- **Prompt and model changes alter behavior:** Version prompts and test representative cases.

## Example

Suppose a support inbox receives the message: "I was charged twice for my subscription yesterday."

The application sends instructions for ticket triage plus the message, and asks for structured output. The model predicts tokens that form a result such as `category: billing` and `needs_human_review: false`.

The application then validates the schema and uses deterministic rules to route the ticket. It does not let the model issue a refund or look up payment data on its own. Those actions require later concepts: trusted data access, tool calling, and authorization.

## Interview Takeaways

- An autoregressive LLM generates by repeatedly selecting the next token from a probability distribution conditioned on its current context.
- Tokens, not words, determine context limits, cost, and much of a request's latency.
- Conversation state is normally application-managed context, not a permanent memory written into the model.
- Temperature controls variation, not factual accuracy or safety.
- Structured outputs make model results easier to consume, but application validation and business rules remain necessary.

## Next

Next: [Embeddings and Semantic Search](../retrieval/embeddings-and-semantic-search.md). It explains how AI systems represent meaning for retrieval instead of generating text directly.

## References

- [Radford et al., *Language Models are Unsupervised Multitask Learners*](https://cdn.openai.com/better-language-models/language_models_are_unsupervised_multitask_learners.pdf)
- [Vaswani et al., *Attention Is All You Need*](https://arxiv.org/abs/1706.03762)
- [Hugging Face Tokenizers documentation](https://huggingface.co/docs/tokenizers/main/en/api/tokenizer)
- [Hugging Face text generation documentation](https://huggingface.co/docs/transformers/main_classes/text_generation)
- [OpenAI Structured Outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
