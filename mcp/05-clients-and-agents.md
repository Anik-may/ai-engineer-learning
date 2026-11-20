# MCP 05: Clients and Agents

## The SDK client

```python
import asyncio
from mcp import Client

async def main() -> None:
    async with Client("http://localhost:8000/mcp") as client:
        listed = await client.list_tools()
        for t in listed.tools:
            print(t.name, t.description, t.input_schema)

        result = await client.call_tool("get_order", {"order_id": "A100"})
        print(result.structured_content, result.is_error)

asyncio.run(main())
```

What you pass to `Client(...)` decides the transport:

| You pass | Transport |
| --- | --- |
| A URL string | Streamable HTTP |
| `StdioServerParameters(command="uv", args=["run", "server.py"], env={...})` | Starts a local server over stdio |
| An `MCPServer` object | In memory. Best for tests |
| `streamable_http_client(url, http_client=...)` | HTTP with your own headers and timeouts |

Good to know:

- `call_tool` returns `content`, `structured_content` and `is_error`.
- A failed tool is **not** an exception. Check `is_error`.
- `MCPError` is raised only for protocol errors.
- List methods take `cursor=`. Loop until `next_cursor` is `None`.
- For auth headers, build an `httpx2.AsyncClient(headers=..., timeout=...)`
  and pass it as `http_client`. Set the timeout: the plain default is 5
  seconds, too short for slow tools.
- To answer elicitation, pass `elicitation_callback=` to `Client`. The client
  then does the MRTR retry for you.
- `mode="auto"` (default) works with both old and new servers.

## The agent loop by hand

Interviews test this loop. Learn it.

```python
async with Client(url) as mcp_client:
    listed = await mcp_client.list_tools()
    llm_tools = [
        {"name": t.name, "description": t.description, "input_schema": t.input_schema}
        for t in listed.tools
    ]

    messages = [{"role": "user", "content": question}]
    while True:
        reply = await llm.chat(messages, tools=llm_tools)   # your LLM provider
        if not reply.tool_calls:
            break                                            # final answer
        messages.append(reply.as_message())
        for call in reply.tool_calls:
            if needs_approval(call) and not await ask_user(call):
                output = "User declined this action."
            else:
                result = await mcp_client.call_tool(call.name, call.arguments)
                output = result.content
            messages.append({"role": "tool", "tool_call_id": call.id, "content": output})
```

In words: list tools → give the schemas to the model → model picks a tool →
host checks rules and asks the user → client calls the server → result goes
back to the model → repeat. Add a step limit so it cannot loop forever.

## LangChain

MCP support is now inside LangChain (`langchain[mcp]>=1.4.0`, still beta).

```bash
pip install "langchain[mcp]"
```

```python
from pathlib import Path
from langchain.agents import create_agent
from langchain.mcp import MCPAdapter

async with MCPAdapter("https://example.com/mcp") as adapter:   # remote server
    tools = await adapter.list_tools()
    agent = create_agent("<provider:model>", tools)
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "Status of order A100?"}]}
    )

stdio = MCPAdapter(Path("server.py"))    # local server over stdio
```

For several servers, pass one config dict: `{"mcpServers": {...}}`
(**Not checked**: see the LangChain "MCP connections" page for the exact
shape.)

Old tutorials use `langchain-mcp-adapters` and `MultiServerMCPClient`. That
is the way before v1.4.0. Know the name, do not use it in new code.

## LangGraph: pause for human approval

MCP tools become normal LangChain tools, so they work in a graph.
(**Not checked** against current LangGraph docs. Confirm the names.)

```python
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt, Command

RISKY = {"create_ticket", "cancel_order"}

def approval(state: MessagesState):
    risky = [c for c in state["messages"][-1].tool_calls if c["name"] in RISKY]
    if risky and interrupt({"approve": risky}) != "approve":   # pauses here
        return Command(goto=END)
    return Command(goto="tools")

builder = StateGraph(MessagesState)
builder.add_node("agent", call_model)
builder.add_node("approval", approval)
builder.add_node("tools", ToolNode(tools))
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition, {"tools": "approval", END: END})
builder.add_edge("tools", "agent")
graph = builder.compile(checkpointer=InMemorySaver())

cfg = {"configurable": {"thread_id": "conv-1"}}
graph.invoke({"messages": [("user", "Cancel order A100")]}, cfg)   # stops and waits
graph.invoke(Command(resume="approve"), cfg)                       # continues
```

The checkpointer saves the paused run, so it can continue later.

## What a good host does

| Job | Why |
| --- | --- |
| Show which tools are available | The user must know what the model can do |
| Ask before risky calls, and show the arguments | Stops silent data leaks |
| Treat tool results as untrusted text | Results can hold prompt injection |
| Check `structuredContent` against `outputSchema` | Catches broken servers |
| Put a timeout on every tool call | A stuck server must not freeze the agent |
| Prefix tool names per server | Two servers may both have `search` |
| Cache `tools/list` using `ttlMs` | Fewer calls, better prompt caching |
| Log every tool call | Audit trail |

## Too many tools

Loading every tool from many servers fills the context and confuses the
model. Fixes:

1. Connect only the servers the task needs.
2. Filter tools by name before giving them to the model.
3. Use a router: first pick a category, then load only its tools.
4. Keep descriptions short and clearly different from each other.

## Exercise

1. Call your server from `04` with `Client`.
2. Write the agent loop by hand with your LLM provider.
3. Load the same tools into a LangGraph agent.
4. Add the approval pause for `create_ticket`.
