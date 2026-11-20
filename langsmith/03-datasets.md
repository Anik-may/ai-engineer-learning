# LangSmith 03: Datasets

A dataset is your test suite. Evaluation is only as good as the dataset.

## What an example looks like

```python
{
    "inputs": {"question": "How do I reset my password?"},
    "outputs": {"answer": "Go to Settings > Security > Reset password."},  # reference
    "metadata": {"category": "account", "language": "en", "source": "prod-trace"},
}
```

- `inputs`: what your app receives.
- `outputs`: the reference ("gold") answer. Optional but strongly recommended.
- `metadata`: labels for slicing results later.

## Create a dataset in code (Verified)

```python
from langsmith import Client

client = Client()

dataset = client.create_dataset(
    dataset_name="support-qa-v1",
    description="Customer support questions with reference answers.",
)

examples = [
    {
        "inputs": {"question": "How do I reset my password?"},
        "outputs": {"answer": "Go to Settings > Security > Reset password."},
        "metadata": {"category": "account"},
    },
    {
        "inputs": {"question": "আমার অর্ডার কোথায়?"},
        "outputs": {"answer": "অর্ডার নম্বর দিন, আমি ট্র্যাকিং দেখে জানাচ্ছি।"},
        "metadata": {"category": "orders", "language": "bn"},
    },
]

client.create_examples(dataset_id=dataset.id, examples=examples)
```

## Read, filter, update (Verified)

```python
examples = client.list_examples(dataset_name="support-qa-v1")
account_only = client.list_examples(
    dataset_name="support-qa-v1", metadata={"category": "account"}
)

client.update_example(
    example_id=example.id,
    outputs={"answer": "Updated reference answer."},
    split="test",
)
```

## Splits (Verified)

Tag examples with a split such as `train`, `test` or `regression`, then run an
experiment on one split. A common setup:

- `smoke`: 10 examples, runs on every commit.
- `regression`: bugs you fixed; must never fail again.
- `full`: everything, runs before a release.

## Where examples come from

| Source | How | Good for |
| --- | --- | --- |
| Hand-written | You write question and answer | Core behaviour, edge cases |
| Production traces | "Add to dataset" in the UI, or query runs in code | Realistic inputs |
| Failures | Every bug report becomes an example | Regression protection |
| Synthetic | An LLM generates questions from your documents | Coverage, cheaply |
| Adversarial | Prompt injections, off-topic, abusive input | Security testing |

Building from traces in code (Verified in outline; check exact call):

```python
runs = client.list_runs(project_name="omnichan-prod", is_root=True, limit=50)
examples = [{"inputs": r.inputs, "outputs": r.outputs} for r in runs]
client.create_examples(dataset_id=dataset.id, examples=examples)
```

Review outputs before using them as references. A production answer is not
automatically a correct answer.

## Rules for a good dataset

1. **Start small.** 30 good examples beat 500 careless ones.
2. **Cover categories evenly** and record the category in metadata.
3. **Include hard cases**: ambiguous questions, missing information,
   questions the bot must refuse, mixed Bangla and English.
4. **Include negatives**: questions whose answer is not in the documents. The
   correct output is "I don't know".
5. **Version it.** Name datasets `-v1`, `-v2`. LangSmith also versions
   datasets on every change, so past experiments stay reproducible.
6. **Keep growing it.** Add every production failure.

## Dataset types (From memory)

- Key-value: free-form dict inputs and outputs (the default).
- Chat: lists of messages.
- LLM: single prompt and completion.

Key-value is right for agents and RAG.

## Exercise

Build `support-qa-v1` with 50 examples: 30 normal, 10 hard, 5 unanswerable,
5 adversarial. Put `category` and `language` in metadata.
