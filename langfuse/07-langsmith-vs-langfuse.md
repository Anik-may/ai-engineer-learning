# Langfuse 07: LangSmith vs Langfuse, and a Cheat Sheet

## Side by side

| Topic | LangSmith | Langfuse |
| --- | --- | --- |
| Licence | Proprietary | Open source |
| Hosting | Cloud; self-hosting is an enterprise offering (From memory) | Cloud or free self-hosting |
| LangChain / LangGraph tracing | Environment variables only | Callback handler |
| Plain Python tracing | `@traceable` | `@observe` |
| Built on | LangChain's run tree | OpenTelemetry |
| Unit of a step | Run | Observation (span, generation, ...) |
| Conversation grouping | Thread | Session |
| Quality signal | Feedback | Score |
| Test collection | Dataset / example | Dataset / item |
| Test execution | Experiment via `client.evaluate` | Experiment via `run_experiment` |
| Judge helpers | `openevals` | Managed evaluators in UI, or your own |
| Prompt store | Prompt commits and tags | Prompt versions and labels |
| Pytest plugin | Yes | Not built in; call the SDK from tests |

## Which to pick

- **Already on LangGraph, small team, data location is not a constraint**:
  LangSmith. Least setup.
- **Customer data must stay on your servers, or you want no vendor lock-in**:
  Langfuse self-hosted.
- **Not using LangChain**: either works; Langfuse's OpenTelemetry base is
  convenient.
- **For interviews**: know one deeply and be able to explain the other. The
  concepts transfer completely.

## The concepts that transfer to any tool

1. Trace every request with user, session, tenant and version attributes.
2. Keep a versioned dataset built from real failures.
3. Score with code first, judges second, humans to calibrate.
4. Compare experiments before shipping a change.
5. Sample and score production traffic.
6. Feed failures back into the dataset.

## Langfuse cheat sheet

```python
from langfuse import get_client, observe, propagate_attributes, Evaluation
from langfuse.langchain import CallbackHandler

langfuse = get_client()

# trace plain Python
@observe(as_type="generation")
def call_model(...): ...

# trace LangChain / LangGraph
handler = CallbackHandler()
graph.invoke(inputs, config={"callbacks": [handler]})

# attributes
with propagate_attributes(user_id="u1", session_id="s1", tags=["prod"]):
    ...

# prompts
langfuse.create_prompt(name="p", type="text", prompt="Hi {{name}}", labels=["production"])
langfuse.get_prompt("p").compile(name="Anik")

# datasets and experiments
langfuse.create_dataset(name="ds")
langfuse.create_dataset_item(dataset_name="ds", input={...}, expected_output={...})
langfuse.get_dataset("ds").run_experiment(name="v1", task=task, evaluators=[ev])

# scores
langfuse.create_score(name="user_thumb", value=1, trace_id=tid, data_type="BOOLEAN")

# always, in short-lived processes
langfuse.flush()
```

Environment:

```bash
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_BASE_URL=https://cloud.langfuse.com
```

## Glossary

- **Annotation queue**: human review inbox.
- **Dataset run**: one experiment over a dataset.
- **Generation**: observation for a model call, with tokens and cost.
- **Label**: named pointer to a prompt version (`production`, `staging`).
- **Observation**: any step in a trace.
- **Score**: numeric, categorical, boolean or text quality signal.
- **Score config**: schema that validates a score's allowed values.
- **Session**: traces grouped into a conversation.
- **Span**: observation for a generic unit of work.
- **Trace**: one request end to end.

## Interview questions to practise

1. Why might a company choose a self-hosted observability tool?
2. What is the difference between a span and a generation?
3. How would you roll back a bad prompt without deploying?
4. How do you connect a user's thumbs-down to the exact trace?
5. What happens to your app if the observability backend is down?
6. What does OpenTelemetry give you in an LLM system?
