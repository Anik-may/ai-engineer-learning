# LangSmith 04: Evaluation

Evaluation = run your app over a dataset + score every output + compare runs.

## The three parts

1. **Target**: the function under test. Takes `inputs`, returns `outputs`.
2. **Dataset**: the examples (see `03-datasets.md`).
3. **Evaluators**: functions that return a score.

## Minimal working evaluation (Verified)

```python
from langsmith import Client

client = Client()

def target(inputs: dict) -> dict:
    answer = my_agent(inputs["question"])       # your app
    return {"answer": answer}

def exact_match(outputs: dict, reference_outputs: dict) -> bool:
    return outputs["answer"].strip() == reference_outputs["answer"].strip()

results = client.evaluate(
    target,
    data="support-qa-v1",
    evaluators=[exact_match],
    experiment_prefix="baseline",
    max_concurrency=4,
)
```

The call prints a link to the experiment in the UI.

## Writing evaluators (Verified)

An evaluator may take any subset of these arguments, by name:

- `inputs: dict` the example inputs
- `outputs: dict` what your target returned
- `reference_outputs: dict` the example's reference outputs
- `run` the full run object (to inspect intermediate steps)
- `example` the full example object

Allowed return values:

| Return | Treated as |
| --- | --- |
| `bool`, `int`, `float` | A numeric metric named after the function |
| `str` | A categorical metric |
| `dict` with `key` and `score` (or `value`) | A named metric; add `comment` to explain |
| `list[dict]` | Several metrics from one function |

Examples:

```python
def is_concise(outputs: dict) -> bool:
    return len(outputs["answer"]) < 600

def cites_source(outputs: dict) -> dict:
    ok = "[source:" in outputs["answer"]
    return {"key": "cites_source", "score": int(ok),
            "comment": "" if ok else "No citation marker found"}

def retrieval_hit(outputs: dict, reference_outputs: dict) -> bool:
    # your target must return the retrieved doc ids
    return reference_outputs["doc_id"] in outputs["retrieved_ids"]
```

Async evaluators and `aevaluate()` exist for async apps.

## Kinds of evaluator

| Kind | Example | Cost | Trust |
| --- | --- | --- | --- |
| Code / heuristic | exact match, regex, JSON valid, length | Free | High |
| Reference-based | answer matches the gold answer | Low | High |
| LLM-as-judge | faithfulness, tone, relevance | Model calls | Medium; needs checking |
| Human | annotation queue | Time | Highest |
| Pairwise | which of two answers is better | Model calls | Good for small differences |

Use code evaluators wherever you can. Use a judge only for what code cannot
measure. See `05-llm-as-judge-and-rag-metrics.md`.

## Comparing experiments

Run the same dataset twice with different `experiment_prefix` values (for
example `baseline` and `new-prompt`). In the UI, select both and open the
comparison view. It shows per-example differences and which examples regressed.

The workflow that matters:

1. Run `baseline`. Record scores.
2. Change one thing (prompt, model, chunk size).
3. Run again. Compare.
4. Keep the change only if scores go up and nothing important regresses.

## Summary evaluators (From memory)

A summary evaluator scores the whole experiment, not one example, for metrics
like precision, recall or F1. Pass it with `summary_evaluators=[...]`. It
receives lists of outputs and reference outputs. Confirm the signature in the
docs page "summary evaluator".

## Evaluating agents

Three levels, from cheapest to most detailed:

1. **Final answer**: is the last message correct?
2. **Trajectory**: did it call the right tools in a sensible order? Return
   the list of tool names from your target and compare with the expected list.
3. **Single step**: given this state, does it choose the right next tool?

```python
def right_tools(outputs: dict, reference_outputs: dict) -> bool:
    return outputs["tools_called"] == reference_outputs["expected_tools"]
```

## Repetitions and noise

LLM outputs vary. For important comparisons run each example several times
(From memory: `num_repetitions=3`) and compare averages, not single runs.

## Exercise

1. Write `target` for your agent.
2. Add three code evaluators: `retrieval_hit`, `is_concise`, `right_tools`.
3. Run a `baseline` experiment.
4. Change the system prompt, run `prompt-v2`, and compare in the UI.
5. Write down which examples regressed and why.
