# LangSmith 08: Cheat Sheet and Glossary

## Environment variables

| Variable | Purpose |
| --- | --- |
| `LANGSMITH_TRACING=true` | Turn tracing on |
| `LANGSMITH_API_KEY` | Your key |
| `LANGSMITH_PROJECT` | Project that receives traces |
| `LANGSMITH_ENDPOINT` | Regional API URL (EU: `https://eu.api.smith.langchain.com`) |
| `LANGSMITH_WORKSPACE_ID` | Needed only if your key spans several workspaces |
| `LANGSMITH_TEST_SUITE` | Name of the pytest suite |
| `LANGSMITH_TEST_TRACKING=false` | Run pytest without uploading |

## Code you will type most

```python
from langsmith import Client, traceable
from langsmith.wrappers import wrap_openai
import langsmith as ls

client = Client()

# trace
@traceable(run_type="tool")
def my_tool(x): ...

# dataset
ds = client.create_dataset(dataset_name="name")
client.create_examples(dataset_id=ds.id, examples=[{"inputs": {...}, "outputs": {...}}])

# evaluate
def my_eval(outputs: dict, reference_outputs: dict) -> bool: ...
client.evaluate(target, data="name", evaluators=[my_eval], experiment_prefix="v1")

# feedback
client.create_feedback(run_id=run_id, key="user_thumb", score=1)

# selective tracing
with ls.tracing_context(enabled=False):
    ...
```

## A to Z glossary

- **Annotation queue**: inbox where humans review and score traces.
- **Automation rule**: action applied to traces matching a filter.
- **Child run**: a run nested inside another run.
- **Dataset**: collection of examples for testing.
- **Evaluator**: function that scores an output.
- **Example**: one test case with inputs and optional reference outputs.
- **Experiment**: one execution of a target over a dataset, with scores.
- **Feedback**: a score attached to a run.
- **Few-shot**: examples placed in the prompt; datasets can supply them.
- **Groundedness / faithfulness**: the answer is supported by the context.
- **Heuristic evaluator**: scoring by code rules, no model.
- **Judge (LLM-as-judge)**: a model that scores another model's output.
- **Key-value dataset**: free-form dict inputs and outputs.
- **Latency p95**: 95% of requests are faster than this.
- **Metadata**: key-value labels on a run, used for filtering and grouping.
- **Offline evaluation**: on a dataset, before release.
- **Online evaluation**: on live traces, after release.
- **Pairwise evaluation**: judging which of two outputs is better.
- **Playground**: UI for trying prompts and models.
- **Project**: bucket of traces.
- **Prompt commit / tag**: a saved prompt version and a named pointer to it.
- **Reference output**: the expected answer for an example.
- **Regression**: an example that passed before and fails now.
- **Run**: one traced unit of work.
- **Run type**: llm, chain, tool, retriever, embedding, prompt, parser.
- **Sampling**: tracing or evaluating only a fraction of traffic.
- **Split**: named subset of a dataset.
- **Summary evaluator**: scores a whole experiment at once.
- **Tag**: string label on a run.
- **Target**: the function being evaluated.
- **Thread**: traces grouped into one conversation.
- **Trace**: tree of runs for one request.
- **Trajectory**: the sequence of steps and tool calls an agent took.
- **TTFT**: time to first token.

## Interview questions to practise

1. How do you know a prompt change made your agent better and not worse?
2. What is the difference between offline and online evaluation?
3. Your RAG bot gave a wrong answer. How do you find out whether retrieval or
   generation failed?
4. When would you not trust an LLM judge? How do you validate one?
5. How do you turn a production failure into a regression test?
6. Which metrics would you put on a dashboard for a support agent?
7. How do you keep personal data out of your traces?
8. How do you evaluate an agent's tool use, not only its final answer?
