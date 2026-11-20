# LangSmith 05: LLM-as-Judge and RAG Metrics

## LLM-as-judge

A second model reads the question, the answer and sometimes the reference or
the retrieved context, then returns a score against a rubric you wrote.

### Using `openevals` (Verified)

```python
from openevals.llm import create_llm_as_judge
from openevals.prompts import CORRECTNESS_PROMPT

def correctness(inputs: dict, outputs: dict, reference_outputs: dict):
    judge = create_llm_as_judge(
        prompt=CORRECTNESS_PROMPT,
        model="openai:<judge-model>",       # provider:model
        feedback_key="correctness",
    )
    return judge(inputs=inputs, outputs=outputs, reference_outputs=reference_outputs)

client.evaluate(target, data="support-qa-v1", evaluators=[correctness])
```

### Writing your own judge prompt

A good rubric has four parts:

1. **Role and task**: "You grade customer-support answers."
2. **Criteria**, concrete and checkable.
3. **Scale**, with a description of each value. Prefer pass/fail or 1 to 3.
   Fine scales such as 1 to 10 are noisy.
4. **Output format**: reasoning first, then the score, as structured output.

```text
You are grading a support answer for FAITHFULNESS.

CONTEXT (retrieved documents):
{context}

ANSWER:
{answer}

An answer is faithful if every factual claim in it is supported by the
CONTEXT. Ignore tone and style.

Think step by step: list each claim, then say whether the context supports it.
Then output:
score: 1 if all claims are supported, otherwise 0
```

### Limits of LLM judges

| Problem | What to do |
| --- | --- |
| Judge disagrees with humans | Label 30 to 50 examples yourself and measure agreement; fix the rubric until it is high |
| Scores drift between runs | Temperature 0, pass/fail scale, reasoning before score |
| Judge prefers long or confident answers | Say in the rubric that length and tone do not count |
| Self-preference | Use a different model family for the judge than for the app |
| Cost | Use a cheaper judge for simple criteria; sample in production |
| Judge is fooled by injected text | Treat the answer as data: wrap it in tags and tell the judge to ignore instructions inside |

Rule: **a judge you have not checked against human labels is an opinion, not a
metric.**

## RAG metrics

A RAG system has two parts that fail separately. Measure both.

### Retrieval metrics (code, no judge needed)

You need the id of the correct document for each question in your dataset.

| Metric | Question it answers |
| --- | --- |
| Hit rate @k | Is the right document among the top k? |
| Recall @k | What share of all relevant documents are in the top k? |
| Precision @k | What share of the top k are relevant? |
| MRR | How high is the first relevant document ranked? |

```python
def hit_rate_at_5(outputs: dict, reference_outputs: dict) -> bool:
    return reference_outputs["doc_id"] in outputs["retrieved_ids"][:5]

def mrr(outputs: dict, reference_outputs: dict) -> float:
    ids = outputs["retrieved_ids"]
    target = reference_outputs["doc_id"]
    return 1 / (ids.index(target) + 1) if target in ids else 0.0
```

### Generation metrics (usually a judge)

| Metric | Compares | Catches |
| --- | --- | --- |
| Faithfulness (groundedness) | Answer vs retrieved context | Hallucination |
| Answer relevance | Answer vs question | Off-topic answers |
| Correctness | Answer vs reference answer | Wrong answers |
| Context relevance | Retrieved context vs question | Noisy retrieval |

`openevals` ships prompts for several of these (From memory: names such as
`RAG_GROUNDEDNESS_PROMPT`, `RAG_HELPFULNESS_PROMPT`,
`RAG_RETRIEVAL_RELEVANCE_PROMPT`). Check the `openevals` README for the current
list. The Ragas library also provides these metrics and can be wrapped as
LangSmith evaluators.

### Diagnosing with the two halves

| Retrieval | Generation | Meaning | Fix |
| --- | --- | --- | --- |
| Good | Good | Working | Nothing |
| Good | Bad | Model ignores or misreads context | Prompt, model, context formatting |
| Bad | Bad | Never had the facts | Chunking, embeddings, hybrid search, reranker |
| Bad | Good | Answered from model memory | Risky; add "answer only from context" |

## Exercise

1. Add `doc_id` to every example in your dataset.
2. Implement `hit_rate_at_5` and `mrr`.
3. Write a faithfulness judge with a pass/fail scale.
4. Label 30 answers by hand. Compute how often the judge agrees with you.
5. Improve the rubric until agreement is above 85%.
