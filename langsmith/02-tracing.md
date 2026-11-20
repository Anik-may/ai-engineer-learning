# LangSmith 02: Tracing

Tracing answers "what did my app actually do for this request?"

## Three ways to trace

### 1. LangChain and LangGraph: no code change (Verified)

Set `LANGSMITH_TRACING=true` and `LANGSMITH_API_KEY`. Every chain, graph node,
model call, retriever call and tool call is traced automatically.

### 2. Wrap the provider SDK (Verified)

```python
from openai import OpenAI
from langsmith.wrappers import wrap_openai

client = wrap_openai(OpenAI())   # every call through this client is logged
```

There are matching wrappers `wrap_anthropic` and `wrap_gemini`.

### 3. The `@traceable` decorator (Verified)

```python
from langsmith import traceable

@traceable(run_type="tool")          # shows as a tool span
def search_orders(customer_id: str) -> list[dict]:
    ...

@traceable(run_type="retriever")
def retrieve(query: str) -> list[str]:
    ...

@traceable                           # the whole pipeline is one trace
def assistant(question: str) -> str:
    docs = retrieve(question)
    ...
```

Decorated functions called inside other decorated functions become child runs,
so the trace tree mirrors your call tree.

## Run types

`llm`, `chain`, `tool`, `retriever`, `embedding`, `prompt`, `parser`.
Choosing the right type changes how the UI renders the run (for example
retriever runs show documents, llm runs show tokens).

## Metadata, tags and names (Verified for LangChain)

Pass them in the config at call time. Child runs inherit them.

```python
graph.invoke(
    {"messages": [("user", "Where is my order?")]},
    config={
        "run_name": "support-agent",
        "tags": ["whatsapp", "prod"],
        "metadata": {"tenant_id": "t_123", "app_version": "1.4.2"},
    },
)
```

Use metadata for anything you will later filter by: tenant, channel, model,
prompt version, app version.

With `@traceable` (From memory):

```python
@traceable(name="support-agent", tags=["prod"], metadata={"app_version": "1.4.2"})
def assistant(question: str) -> str: ...
```

## Trace only some calls (Verified)

```python
import langsmith as ls

with ls.tracing_context(enabled=True):
    chain.invoke({"question": "..."})      # traced even if the env var is unset

with ls.tracing_context(enabled=False):
    chain.invoke({"question": "..."})      # never traced
```

Use this to keep sensitive requests out of LangSmith.

## Threads: grouping a conversation (From memory)

Put the same identifier in metadata under `session_id`, `thread_id` or
`conversation_id` on every trace of a conversation. LangSmith groups them in
the Threads view.

## What to look at in a trace

1. **Inputs and outputs** of each step: where did the wrong text first appear?
2. **Retrieved documents**: was the answer in them at all?
3. **Tool calls**: right tool, right arguments?
4. **Latency per step**: which step is slow?
5. **Tokens and cost per step**: which step is expensive?

## Privacy

Traces contain user messages. Before sending production traffic:

- Mask personal data. LangSmith supports hiding inputs and outputs and custom
  masking functions (From memory: `LANGSMITH_HIDE_INPUTS`,
  `LANGSMITH_HIDE_OUTPUTS`, and an anonymizer on the `Client`). Confirm in the
  docs under "mask inputs and outputs".
- Check where data is stored (US or EU region) against your customers' needs.

## Sampling

At high volume, trace a fraction of requests.
From memory: `LANGSMITH_TRACING_SAMPLING_RATE=0.1` traces 10%. Confirm.

## Exercise

1. Turn on tracing for your LangGraph agent in a dev project.
2. Add `tenant_id`, `channel` and `app_version` metadata.
3. Send 20 test conversations.
4. Find the slowest step and the most expensive step. Write both down.
