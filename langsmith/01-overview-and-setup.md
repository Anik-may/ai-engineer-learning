# LangSmith 01: Overview and Setup

## What LangSmith is

LangSmith is a hosted platform from the LangChain team for three jobs:

1. **Observability**: record every step of an LLM app as a trace.
2. **Evaluation**: run your app over a dataset and score the results.
3. **Prompt engineering**: version, test and compare prompts.

It works with any code, not only LangChain. With LangChain or LangGraph you
get tracing by setting environment variables and changing no code.

## Core vocabulary

| Term | Meaning |
| --- | --- |
| Run | One unit of work: an LLM call, a tool call, a retriever call, a chain |
| Trace | A tree of runs for one request, from the first input to the final output |
| Project | A named bucket that traces are written to |
| Thread | Several traces grouped as one conversation |
| Dataset | A collection of examples used for testing |
| Example | One test case: inputs plus optional reference outputs |
| Experiment | One run of your app over a dataset, with scores |
| Evaluator | A function that scores an output |
| Feedback | A score attached to a run (from a human, code, or an LLM judge) |

## Setup (Verified)

```bash
pip install -U langsmith openevals
```

Create an API key in the LangSmith UI under Settings, then set:

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY="<your-langsmith-api-key>"
export LANGSMITH_PROJECT="omnichan-dev"        # which project receives traces
# Only if your account is outside the US region:
export LANGSMITH_ENDPOINT="https://eu.api.smith.langchain.com"
```

Notes:

- The current docs also describe `LANGSMITH_AGENT_ID` and
  `LANGSMITH_AGENT_ENVIRONMENT` (`local`, `development`, `staging`,
  `production`) for workspaces organised around agents. If you use those, the
  docs say to remove `LANGSMITH_PROJECT`. Check which style your workspace uses.
- Older tutorials use `LANGCHAIN_TRACING_V2` and `LANGCHAIN_API_KEY`. Prefer
  the `LANGSMITH_*` names.
- Never commit the API key. Keep it in `.env` and add `.env` to `.gitignore`.

## First trace in five lines (Verified)

```python
from langsmith import traceable

@traceable
def answer(question: str) -> str:
    return "42"

answer("What is the meaning of life?")
```

Open the project in the UI and you should see one trace.

## Pricing

There is a free tier with a monthly trace limit and paid plans above it.
I did not verify the current numbers; check https://www.langchain.com/pricing.

## What to learn next

1. `02-tracing.md` to see inside your agent.
2. `03-datasets.md` and `04-evaluation.md` to measure quality.
3. `07-production-monitoring.md` to watch a live system.
