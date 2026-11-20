# Langfuse 02: Tracing

## Three ways to create observations

### 1. The `@observe` decorator (Verified)

```python
from langfuse import observe

@observe(name="retrieve", as_type="retriever")
def retrieve(query: str) -> list[str]:
    ...

@observe(name="llm-call", as_type="generation")
def call_model(messages: list[dict]) -> str:
    ...

@observe
def assistant(question: str) -> str:
    docs = retrieve(question)
    return call_model(build_messages(question, docs))
```

A decorated function called inside another decorated function becomes a child
observation. Function arguments are recorded as input and the return value as
output.

### 2. Context managers (Verified)

Use these when a step is not a whole function.

```python
from langfuse import get_client
langfuse = get_client()

with langfuse.start_as_current_observation(as_type="span", name="process-request") as span:
    with langfuse.start_as_current_observation(
        as_type="generation", name="llm-response", model="<model-name>"
    ) as generation:
        text = call_provider(...)
        generation.update(
            output=text,
            usage_details={"input_tokens": 120, "output_tokens": 45},
        )
    span.update(output={"answer": text})
```

### 3. Manual observations (Verified)

When start and end happen in different places:

```python
child = parent.start_observation(name="manual-step", as_type="generation")
...
child.update(output="...")
child.end()          # From memory: manual observations must be ended explicitly
```

## Observation types

Set with `as_type`. From memory the list includes `span`, `generation`,
`tool`, `agent`, `chain`, `retriever`, `embedding`, `evaluator` and
`guardrail`. Check the "observation types" page. The type controls how the UI
shows the step; `generation` is the one with model, token and cost fields.

## User, session, tags, metadata (Verified)

```python
from langfuse import propagate_attributes

with propagate_attributes(
    user_id="user_123",
    session_id="conv_abc",
    tags=["whatsapp", "prod"],
    metadata={"tenant_id": "t_42", "app_version": "1.4.2"},
):
    assistant("Where is my order?")
```

Everything created inside the block carries these attributes. `session_id`
groups traces into one conversation in the Sessions view.

## Updating the current observation (Verified)

```python
langfuse.update_current_span(metadata={"step1_complete": True})
langfuse.update_current_generation(...)      # named in the docs; check arguments
```

The trace's own input and output come from its root observation, so set them
by calling `.update(input=..., output=...)` on the root.

## Tokens and cost

- If you pass `model` and `usage_details` on a generation, Langfuse computes
  cost from its model price table.
- For models it does not know, add a custom model price in project settings
  or pass cost details yourself (From memory: `cost_details={...}`).
- Provider wrappers and framework integrations fill these in automatically.

## Provider integrations (From memory)

There is a drop-in OpenAI wrapper:

```python
from langfuse.openai import openai     # use this instead of `import openai`
```

Because the SDK is built on OpenTelemetry, any OpenTelemetry instrumentation
for LLM libraries can also send spans to Langfuse.

## Flushing and shutdown (Verified)

```python
langfuse.flush()        # block until queued events are sent
```

Call it at the end of scripts, tests, Celery tasks and serverless handlers.

## Masking personal data (From memory)

The client accepts a `mask` function applied to inputs and outputs before
they leave your process. Use it to redact phone numbers, emails and tokens.
Confirm the parameter name in the SDK reference.

## Reading a trace

Same questions as in LangSmith:

1. Where did the wrong text first appear?
2. Was the answer in the retrieved documents?
3. Were the right tools called with the right arguments?
4. Which step is slowest?
5. Which step costs most?

## Exercise

1. Instrument a plain Python RAG function with `@observe`.
2. Mark the model call as a `generation` and record token usage.
3. Wrap calls in `propagate_attributes` with a user id and session id.
4. Open the Sessions view and replay one conversation.
