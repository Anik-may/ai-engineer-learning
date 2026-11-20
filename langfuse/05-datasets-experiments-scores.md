# Langfuse 05: Datasets, Experiments and Scores

## Datasets (Verified)

```python
from langfuse import get_client
langfuse = get_client()

langfuse.create_dataset(
    name="support-qa-v1",
    description="Support questions with reference answers",
    metadata={"author": "Anik"},
)

langfuse.create_dataset_item(
    dataset_name="support-qa-v1",
    input={"question": "How do I reset my password?"},
    expected_output={"answer": "Go to Settings > Security > Reset password."},
    metadata={"category": "account"},
)

# Turn a production trace into a test case:
langfuse.create_dataset_item(
    dataset_name="support-qa-v1",
    input={"question": "..."},
    expected_output={"answer": "..."},
    source_trace_id="<trace_id>",
)

dataset = langfuse.get_dataset("support-qa-v1")
```

## Experiments (Verified)

An experiment runs a task over data and scores each result.

```python
from langfuse import Evaluation

def my_task(*, item, **kwargs):
    question = item.input["question"]          # DatasetItem from Langfuse
    return my_agent(question)

def contains_answer(*, input, output, expected_output, metadata, **kwargs):
    ok = expected_output["answer"].lower() in output.lower()
    return Evaluation(name="contains_answer", value=1.0 if ok else 0.0,
                      comment="" if ok else "Reference text not found")

def average_score(*, item_results, **kwargs):
    values = [e.value for r in item_results for e in r.evaluations
              if e.name == "contains_answer" and e.value is not None]
    return Evaluation(name="avg_contains_answer",
                      value=sum(values) / len(values) if values else None)

dataset = langfuse.get_dataset("support-qa-v1")
result = dataset.run_experiment(
    name="baseline",
    task=my_task,
    evaluators=[contains_answer],
    run_evaluators=[average_score],
    max_concurrency=4,
)
print(result.format())
```

Notes:

- Task signature: `def task(*, item, **kwargs)`.
- Item evaluator signature:
  `def ev(*, input, output, expected_output, metadata, **kwargs)`.
- Run-level evaluator signature: `def ev(*, item_results, **kwargs)`.
- Evaluators return `Evaluation(name=..., value=..., comment=...)`.
- Other options: `run_name`, `description`, `metadata`.
- The exact attributes on `item_results` entries are From memory; print one to
  confirm.

You can also run over local data without a hosted dataset:

```python
result = langfuse.run_experiment(
    name="quick-check",
    data=[{"input": "What is the capital of France?", "expected_output": "Paris"}],
    task=lambda *, item, **kw: my_agent(item["input"]),
)
```

Each task execution creates a trace. Item scores attach to those traces; run
scores attach to the dataset run. Runs can be compared side by side in the UI.

## Scores (Verified)

A score is how any quality signal is stored: user feedback, code checks,
judge results, human review.

Three ways to create one:

```python
# 1. By id, from anywhere (for example a feedback endpoint)
langfuse.create_score(
    name="user_thumb",
    value=1,
    trace_id=trace_id,
    data_type="BOOLEAN",
    comment="Solved my problem",
)

# 2. On the observation you are inside
with langfuse.start_as_current_observation(as_type="span", name="answer") as span:
    span.score(name="relevance", value=0.9)        # this observation
    span.score_trace(name="helpful", value=1.0)    # the whole trace

# 3. From context
langfuse.score_current_span(name="relevance", value=0.9)
langfuse.score_current_trace(name="helpful", value=1.0)
```

Data types:

| Type | Value |
| --- | --- |
| `NUMERIC` | float |
| `CATEGORICAL` | string |
| `BOOLEAN` | float `1` or `0`; pass `data_type="BOOLEAN"` explicitly |
| `TEXT` | string, 1 to 500 characters |

Optional: `observation_id` (score one step), `comment`, `config_id` (validate
against a score config), `score_id` (idempotency key, lets you update).

## User feedback end to end

1. When you answer, return the trace id to the frontend.
   From memory: `langfuse.get_current_trace_id()` inside the traced code.
2. The thumbs button posts `{trace_id, value}` to your backend.
3. The backend calls `create_score`.

## LLM-as-judge in the UI (From memory)

Langfuse can run managed judge evaluators on new traces or on dataset runs:
choose a template (hallucination, relevance, toxicity and so on) or write your
own prompt, map trace fields to prompt variables, set a sampling rate and a
filter. Results appear as scores. This is the equivalent of LangSmith's
online evaluators.

## Annotation queues (From memory)

Human review inbox with score configs as the rubric. Use it to label a sample
of traces and to check agreement between your judge and humans.

## Exercise

1. Create `support-qa-v1` with 50 items.
2. Write `my_task` and two evaluators (one code, one judge).
3. Run `baseline`, change the prompt, run `prompt-v2`, compare in the UI.
4. Add a thumbs endpoint that calls `create_score`.
