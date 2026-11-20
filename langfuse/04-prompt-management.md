# Langfuse 04: Prompt Management

Store prompts in Langfuse, not in code, so that:

- a prompt can change without a deploy,
- every trace records which prompt version produced it,
- you can compare quality, cost and latency per prompt version.

## Create a prompt (Verified)

Text prompt:

```python
from langfuse import get_client
langfuse = get_client()

langfuse.create_prompt(
    name="movie-critic",
    type="text",
    prompt="As a {{criticlevel}} movie critic, do you like {{movie}}?",
    labels=["production"],      # optional: promote immediately
)
```

Chat prompt:

```python
langfuse.create_prompt(
    name="support-system",
    type="chat",
    prompt=[
        {"role": "system", "content": "You are a support agent for {{company}}."},
        {"role": "user", "content": "{{question}}"},
    ],
    labels=["production"],
)
```

Variables use double curly braces. Creating a prompt with an existing name
adds a new version.

## Fetch and compile (Verified)

```python
prompt = langfuse.get_prompt("movie-critic")            # production version
text = prompt.compile(criticlevel="expert", movie="Dune 2")

chat_prompt = langfuse.get_prompt("support-system", type="chat")
messages = chat_prompt.compile(company="Omnichan", question="Where is my order?")
```

## Versions and labels

- Every save is a numbered **version**: 1, 2, 3.
- A **label** is a named pointer to a version. `production` is what
  `get_prompt` returns by default; `latest` points at the newest.
- Add your own labels such as `staging` or `experiment-a`.

Fetching a specific one (From memory; the REST API takes `?label=` and
`?version=`, and the Python SDK takes matching keyword arguments):

```python
langfuse.get_prompt("support-system", label="staging")
langfuse.get_prompt("support-system", version=3)
```

Promoting a version is a label move in the UI. Rolling back is moving the
label back. No deploy.

## Caching (Verified)

The SDK caches prompts in memory. After the first fetch they are served with
no added latency, and if Langfuse is unreachable your app keeps using the
cached prompt.

From memory: the cache has a time-to-live you can set (`cache_ttl_seconds`),
and you can pass a `fallback` prompt for the case where the first fetch fails.

## Link a prompt to its traces (From memory)

Pass the prompt object when you create the generation so the trace records
the version:

```python
with langfuse.start_as_current_observation(
    as_type="generation", name="answer", model="<model>", prompt=prompt
) as gen:
    ...
```

Then the prompt page shows metrics per version: latency, cost, scores.
Confirm in the docs page "Link prompts to traces".

## Config on a prompt (From memory)

A prompt can carry a `config` object: model name, temperature, tool
definitions. Reading these from the prompt lets you change the model without a
deploy too.

## A safe release workflow

1. Edit the prompt in the UI. It becomes version N+1 with no label.
2. Run your dataset experiment against version N+1 (label it `staging`).
3. Compare scores with the `production` version.
4. If better, move the `production` label to N+1.
5. Watch online scores for an hour. If worse, move the label back.

## Playground

The UI playground runs a prompt against a model with variables filled in, and
can open a generation from a real trace so you can replay a failure and edit
the prompt until it passes.

## Exercise

1. Move your agent's system prompt into Langfuse as a chat prompt.
2. Fetch it at runtime with `get_prompt` and compile variables.
3. Create a second version, label it `staging`, and run both over your dataset.
4. Promote the better one and roll back once to practise.
