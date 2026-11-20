# Langfuse 03: LangChain and LangGraph Integration

Unlike LangSmith, Langfuse does not trace LangChain through environment
variables alone. You pass a callback handler.

## The callback handler (Verified)

```python
from langfuse.langchain import CallbackHandler

langfuse_handler = CallbackHandler()

agent.invoke(
    {"messages": [{"role": "user", "content": "what is 42 + 58?"}]},
    config={"callbacks": [langfuse_handler]},
)
```

The same pattern works for chains, LangGraph graphs and agents. Every node,
model call and tool call becomes an observation.

Requirements: Python SDK v5 and `langchain-core>=1`.

## User, session and tags (Verified)

Option A, through LangChain metadata:

```python
config = {
    "callbacks": [langfuse_handler],
    "metadata": {
        "langfuse_user_id": "user_123",
        "langfuse_session_id": "conv_abc",
        "langfuse_tags": ["whatsapp", "prod"],
    },
}
graph.invoke(inputs, config=config)
```

Option B, through the SDK:

```python
from langfuse import propagate_attributes

with propagate_attributes(
    trace_name="support-agent",
    session_id="conv_abc",
    user_id="user_123",
):
    graph.invoke(inputs, config={"callbacks": [langfuse_handler]})
```

## Streaming

Pass the same config to `stream` or `astream`. The trace is completed when
the stream ends.

```python
async for chunk in graph.astream(inputs, config={"callbacks": [langfuse_handler]}):
    ...
```

## FastAPI pattern

Create one handler at startup and reuse it; set per-request attributes
around the call.

```python
from fastapi import FastAPI
from langfuse import get_client, propagate_attributes
from langfuse.langchain import CallbackHandler

app = FastAPI()
langfuse = get_client()
handler = CallbackHandler()

@app.post("/chat")
async def chat(req: ChatRequest):
    with propagate_attributes(
        user_id=req.user_id,
        session_id=req.conversation_id,
        metadata={"tenant_id": req.tenant_id, "channel": req.channel},
    ):
        result = await graph.ainvoke(
            {"messages": req.messages},
            config={"callbacks": [handler]},
        )
    return result
```

On shutdown, call `langfuse.flush()` (From memory: or `langfuse.shutdown()`).

## Mixing decorators and the handler

If part of your code is plain Python and part is LangChain, decorate the
outer function with `@observe` and pass the handler to the inner LangChain
call. The LangChain observations nest under the decorated function's span.

## Using a Langfuse prompt in LangChain (From memory)

```python
from langchain_core.prompts import ChatPromptTemplate

lf_prompt = langfuse.get_prompt("support-system", type="chat")
prompt = ChatPromptTemplate.from_messages(
    lf_prompt.get_langchain_prompt(),
    # attaching the prompt to metadata links generations to this version
)
prompt.metadata = {"langfuse_prompt": lf_prompt}
```

Confirm the linking step in the docs page "Link prompts to traces".

## Running LangSmith and Langfuse together

They do not conflict. LangSmith traces through environment variables;
Langfuse through the callback. Useful while comparing the two. Turn one off
in production to avoid paying the overhead twice.

## Exercise

1. Add `CallbackHandler` to your LangGraph agent.
2. Pass user id, session id and tenant id.
3. Trigger a tool error on purpose and find it in the trace.
4. Compare the same request in LangSmith and Langfuse. Note what each shows
   that the other does not.
