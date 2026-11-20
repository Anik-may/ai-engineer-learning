# Langfuse 01: Overview and Setup

## What Langfuse is

Langfuse is an open-source LLM engineering platform. It covers the same
ground as LangSmith:

1. **Observability**: traces of every model call, tool call and retrieval.
2. **Prompt management**: versioned prompts fetched at runtime.
3. **Evaluation**: datasets, experiments, scores, LLM-as-judge, human review.

The differences that matter:

- It is open source and can be **self-hosted** (see `06-self-hosting.md`).
- It is framework-neutral and built on **OpenTelemetry**.
- There is also a hosted cloud version.

## Data model

| Term | Meaning | LangSmith equivalent |
| --- | --- | --- |
| Trace | One request end to end | Trace |
| Observation | One step inside a trace | Run |
| Span | Observation for a generic step | Chain run |
| Generation | Observation for a model call, with model, tokens, cost | LLM run |
| Session | Traces grouped into a conversation | Thread |
| User | The end user a trace belongs to | Metadata |
| Score | A number, category or boolean on a trace or observation | Feedback |
| Dataset / item | Test collection / one test case | Dataset / example |
| Dataset run / experiment | One execution over a dataset | Experiment |
| Prompt | Versioned prompt with labels | Prompt commit and tag |

## Versions (Verified)

- Python SDK: **v5**. It requires `langchain-core>=1` for the LangChain
  integration.
- Self-hosted server: **v4** (the v5 SDK needs server v4.59.0 or later).

Tutorials written for SDK v2 use a different API (`langfuse.trace(...)`,
`langfuse.decorators`). Do not copy those.

## Setup (Verified)

```bash
pip install langfuse
```

Create a project in the Langfuse UI, copy the keys, then put them in `.env`:

```bash
LANGFUSE_SECRET_KEY="sk-lf-..."
LANGFUSE_PUBLIC_KEY="pk-lf-..."
LANGFUSE_BASE_URL="https://cloud.langfuse.com"     # or your self-hosted URL
```

For self-hosting, `LANGFUSE_BASE_URL` is your own address, for example
`http://localhost:3000`.

## First trace (Verified)

```python
from langfuse import get_client, observe

langfuse = get_client()

@observe
def answer(question: str) -> str:
    return "42"

answer("What is the meaning of life?")
langfuse.flush()        # send buffered data before the script exits
```

`flush()` matters in short scripts, tests and serverless functions. A
long-running server sends data in the background.

## What to learn next

1. `02-tracing.md` and `03-langchain-langgraph.md` to instrument your agent.
2. `04-prompt-management.md`.
3. `05-datasets-experiments-scores.md` to evaluate.
4. `06-self-hosting.md` if data must stay on your servers.
