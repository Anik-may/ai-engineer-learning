# LangSmith 06: Pytest and CI

Evaluations are most useful when they run automatically on every change.

## Pytest integration (Verified)

```bash
pip install -U "langsmith[pytest]"
```

```python
import pytest
from langsmith import testing as t

@pytest.mark.langsmith
def test_refund_policy():
    question = "Can I get a refund after 30 days?"
    expected = "No. Refunds are available within 14 days."

    t.log_inputs({"question": question})
    t.log_reference_outputs({"answer": expected})

    answer = my_agent(question)
    t.log_outputs({"answer": answer})

    t.log_feedback(key="mentions_14_days", score="14 days" in answer)
    assert "14 days" in answer
```

What happens:

- Each decorated test is synced to a dataset example.
- Each run is traced and added to an experiment.
- Pass or fail is recorded under the `pass` feedback key.

Wrap judge calls so they are traced separately from the app:

```python
with t.trace_feedback():
    grade = judge(question, answer)
    t.log_feedback(key="correctness", score=grade)
```

## Running (Verified)

```bash
pytest tests/                                          # basic
LANGSMITH_TEST_SUITE='support agent tests' pytest tests/   # name the suite
pytest --langsmith-output tests                        # rich terminal table
LANGSMITH_TEST_TRACKING=false pytest tests/            # dry run, nothing uploaded
```

## Parametrised tests over many cases

```python
CASES = [
    ("How do I reset my password?", "Settings"),
    ("Where is my order?", "order number"),
]

@pytest.mark.langsmith
@pytest.mark.parametrize("question,must_contain", CASES)
def test_answers(question, must_contain):
    t.log_inputs({"question": question})
    answer = my_agent(question)
    t.log_outputs({"answer": answer})
    assert must_contain.lower() in answer.lower()
```

## `evaluate()` or pytest?

| Use | When |
| --- | --- |
| `client.evaluate()` | Large datasets, score distributions, comparing experiments |
| pytest | Each case has its own assertion; you want a red or green build |

Most teams use both: pytest for a small must-pass set, `evaluate()` for the
full quality report.

## GitHub Actions

`.github/workflows/evals.yml`:

```yaml
name: evals
on:
  pull_request:
    paths: ["app/**", "prompts/**", "tests/**"]

jobs:
  evals:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt "langsmith[pytest]"
      - run: pytest tests/evals --langsmith-output
        env:
          LANGSMITH_API_KEY: ${{ secrets.LANGSMITH_API_KEY }}
          LANGSMITH_TRACING: "true"
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

## Gating a release on scores

For `evaluate()` runs, read the results and fail the job below a threshold:

```python
import sys

results = client.evaluate(target, data="support-qa-v1", evaluators=[hit_rate_at_5])
df = results.to_pandas()                      # From memory: confirm method name
score = df["feedback.hit_rate_at_5"].mean()
print(f"hit rate: {score:.2f}")
sys.exit(0 if score >= 0.85 else 1)
```

## Keeping CI cheap and stable

1. Run the small `smoke` split on every pull request; the full set nightly.
2. Cache model responses for unchanged inputs where your tests allow it.
3. Set temperature 0 for the app under test.
4. Use thresholds with a margin, not exact scores; LLM output varies.
5. Limit concurrency so you do not hit provider rate limits.

## Exercise

1. Write 10 pytest cases for your agent's must-not-break behaviour.
2. Add the workflow above to your project repository.
3. Break the prompt on purpose and confirm the build fails.
