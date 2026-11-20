# MCP 10: Testing and Debugging

## The MCP Inspector

The official tool for trying a server by hand: list tools, call them, read
resources, see raw messages.

```bash
uv run mcp dev server.py      # needs npx on your PATH
```

For every tool, check:

1. Are the name and description clear?
2. Is the input schema what you meant (types, required fields)?
3. Does a good call return text and structured content?
4. Does a bad call return a helpful error?

Docs: https://modelcontextprotocol.io/docs/2026-07-28/tools/inspector

## Unit tests: the in-memory client

Pass the server object to `Client`. No subprocess, no network, but the full
protocol still runs.

```bash
uv add --dev pytest
```

```python
import pytest
from mcp import Client
from server import mcp            # your MCPServer


@pytest.fixture
def anyio_backend():
    return "asyncio"


@pytest.fixture
async def client():
    async with Client(mcp, raise_exceptions=True) as c:
        yield c


@pytest.mark.anyio
async def test_get_order_ok(client):
    result = await client.call_tool("get_order", {"order_id": "A100"})
    assert result.structured_content["status"] == "shipped"


@pytest.mark.anyio
async def test_unknown_order_is_tool_error(client):
    result = await client.call_tool("get_order", {"order_id": "nope"})
    assert result.is_error
    assert "No order" in result.content[0].text


@pytest.mark.anyio
async def test_create_ticket_is_idempotent(client):
    args = {"subject": "s", "body": "b", "idempotency_key": "k1"}
    first = await client.call_tool("create_ticket", args)
    second = await client.call_tool("create_ticket", args)
    assert first.structured_content == second.structured_content
```

`raise_exceptions=True` shows real errors from outside the tool body instead
of "Internal server error". Use it in tests only. Auth is skipped in memory,
so test auth over HTTP.

## What to test

| Layer | Test |
| --- | --- |
| Tool logic | Plain unit tests on your functions |
| Tool contract | In-memory client: schema, structured output, error text |
| Authorization | No token, wrong audience, missing scope, another tenant's id |
| Idempotency | Same key twice gives one result and one side effect |
| Limits | Too-big input, huge result, slow upstream |
| **With a model** | Dataset: does the model pick the right tool with the right arguments? |

The last row matters most and people skip it. A tool can work when you call
it and still fail because the model misreads its description. Measure tool
selection accuracy with your LangSmith or Langfuse dataset.

## Debugging

| Problem | Check |
| --- | --- |
| Host does not see the server | Full path in the config? Does the command run by hand? Full path to `uv`? Restarted the host? Read the host's MCP logs |
| stdio server dies or messages are garbage | Something wrote to **stdout**. Search for `print(`. Libraries that print banners break it too. Log to stderr. Set `PYTHONUNBUFFERED=1` |
| Tools are listed but calls fail | JSON-RPC error (protocol) or `isError` (your tool)? Compare the model's arguments with the schema. Repeat the call in the Inspector |
| HTTP 400 | Missing `MCP-Protocol-Version`, `Mcp-Method` or `Mcp-Name`. Header does not match body (`-32020`). Unsupported version (`-32022`). Old client with a new server, or the other way round |
| HTTP 421 | `Host` is not in the allow-list. Set `transport_security` (see `06`) |
| HTTP 401 | No token, expired, or wrong audience. Read `WWW-Authenticate` |
| HTTP 403 | Missing scope (the header names it), or a bad `Origin` |
| "Task group is not initialized" | The app is mounted without the lifespan step (see `04`) |
| "Invalid or expired requestState" | Server copies do not share a `RequestStateSecurity` key |
| Stream stops behind a proxy | The proxy buffers. Turn buffering off. Send keep-alive comments |
| Works in the Inspector, but the model never calls it | Description is unclear or too close to another tool. Too many tools loaded. Fix the description first: it is the only manual the model has |

## Observability

- Log one line per tool call: tool, caller, time taken, success, error type.
- Pass the OpenTelemetry trace context in `_meta` (`traceparent`), so one
  trace covers the agent and the server.
- In your agent, MCP tool calls already show up in LangSmith or Langfuse as
  tool runs.

## Load testing

Before going live, measure latency and error rate at the load you expect,
and check that rate limits work. The Inspector docs have a load-testing
guide.

## Exercise

1. Write the tests above for your server.
2. Add `print()` to a stdio server, watch it break, then fix it.
3. Build a 20-example dataset of user requests with the expected tool.
   Measure how often your agent picks the right one.
