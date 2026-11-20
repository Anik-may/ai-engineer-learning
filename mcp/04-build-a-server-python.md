# MCP 04: Build a Server in Python

Uses the MCP Python SDK **v2**. Docs: https://py.sdk.modelcontextprotocol.io

## v1 to v2: what changed

| v1 (old tutorials) | v2 |
| --- | --- |
| `from mcp.server.fastmcp import FastMCP` | `from mcp.server import MCPServer` |
| Transport options in the constructor | Transport options go to `run()` |
| `ClientSession` + `stdio_client` | `from mcp import Client` |
| `McpError` | `MCPError` |
| `ctx.info("msg")` | Standard `logging` |
| `httpx` | `httpx2` (comes with the SDK) |

## Setup

Needs Python **3.11 or higher**.

```bash
uv init support-mcp && cd support-mcp
uv add "mcp[cli]"          # pin it in pyproject: mcp>=2,<3
```

## A small complete server

```python
import logging
import uuid
from typing import Literal

from pydantic import BaseModel, Field
from mcp.server import MCPServer
from mcp.server.mcpserver.exceptions import ToolError

logger = logging.getLogger(__name__)   # logging writes to stderr

mcp = MCPServer("support")

ORDERS = {"A100": {"status": "shipped"}}   # fake data
TICKETS: dict[str, dict] = {}


class Order(BaseModel):
    order_id: str
    status: Literal["pending", "shipped", "delivered", "cancelled"]


@mcp.tool()
def get_order(order_id: str) -> Order:
    """Look up one order by its id.

    Use this when the user asks about the status of one order.

    Args:
        order_id: The order id, for example A100.
    """
    order = ORDERS.get(order_id)
    if order is None:
        raise ToolError(f"No order with id {order_id}. Ask the user to check it.")
    return Order(order_id=order_id, status=order["status"])


@mcp.tool()
def create_ticket(
    subject: str = Field(description="Short summary, under 80 characters"),
    body: str = Field(description="Full description of the problem"),
    idempotency_key: str = Field(description="Unique key. Reuse it when you retry"),
) -> dict:
    """Create a support ticket. Call only after the user confirms."""
    if idempotency_key in TICKETS:
        return TICKETS[idempotency_key]            # safe to retry
    ticket = {"ticket_id": f"T-{uuid.uuid4().hex[:8]}", "subject": subject}
    TICKETS[idempotency_key] = ticket
    logger.info("created ticket %s", ticket["ticket_id"])
    return ticket


@mcp.resource("orders://{order_id}")
def order_resource(order_id: str) -> str:
    """Raw order record as text."""
    return str(ORDERS.get(order_id, {}))


@mcp.prompt()
def summarise_order(order_id: str) -> str:
    """Ask the model to summarise an order for the customer."""
    return f"Summarise order {order_id} in two friendly sentences."


if __name__ == "__main__":
    mcp.run()          # stdio by default
```

What the SDK does for you:

- Type hints and the docstring become the `inputSchema` and description.
- The return value becomes both `content` and `structured_content`, and an
  `outputSchema` is made from the return type.

## Errors

| You raise | The client gets | Model sees it? |
| --- | --- | --- |
| `ToolError("...")` | Result with `is_error=True` and your message | Yes |
| `MCPError(code=..., message=...)` (from `mcp`) | JSON-RPC error | No |
| Any other exception | `is_error=True` with only "Error executing tool X" | Not the details |

Rule: if a smarter model could avoid the error, raise `ToolError`.

## Run it

```bash
uv run server.py                                      # stdio
uv run mcp dev server.py                              # opens the Inspector (needs npx)
uv run mcp run server.py --transport streamable-http  # http://127.0.0.1:8000/mcp
```

Or in code:

```python
mcp.run(transport="streamable-http", host="127.0.0.1", port=8000)
```

Defaults: host `127.0.0.1`, port `8000`, path `/mcp`, request body limit
4 MiB (`max_request_body_size=`).

**stdio rule: never write to stdout.** stdout carries the protocol. One
`print()` breaks it. Use `logging`.

## Progress

```python
from mcp.server.mcpserver import Context

@mcp.tool()
async def import_orders(file_url: str, ctx: Context) -> str:
    """Import orders from a CSV file."""
    for i in range(100):
        ...
        await ctx.report_progress(i + 1, 100)    # total so far, not a step
    return "Imported 100 orders"
```

## Ask the user (elicitation)

The old `ctx.elicit()` **fails on `2026-07-28` connections**. Use a
**resolver**: a function the SDK runs before your tool.

```python
from typing import Annotated
from mcp.server.mcpserver import Elicit, Resolve

class Confirm(BaseModel):
    ok: bool = Field(description="Cancel this order?")

async def ask_confirm(order_id: str) -> Confirm | Elicit[Confirm]:
    return Elicit(f"Cancel order {order_id}?", Confirm)

@mcp.tool()
async def cancel_order(
    order_id: str,
    confirm: Annotated[Confirm, Resolve(ask_confirm)],
) -> str:
    """Cancel an order after the user confirms."""
    if not confirm.ok:
        return "Not cancelled."
    return f"Order {order_id} cancelled."
```

- The model does not see `confirm` in the schema, so it cannot fake it.
- The answer model must be flat: `str`, `int`, `float`, `bool` or a `Literal`.
- If the user declines, the call fails. To handle that yourself, type the
  parameter as `ElicitationResult[Confirm]`.
- The SDK signs `requestState` with a key made at startup. With more than one
  server copy, share a key:
  `MCPServer("x", request_state_security=RequestStateSecurity(keys=[key]))`.

## Add it to an existing web app

```python
import contextlib
from starlette.applications import Starlette
from starlette.routing import Mount

mcp_app = mcp.streamable_http_app()

@contextlib.asynccontextmanager
async def lifespan(app):
    async with mcp.session_manager.run():    # required
        yield

app = Starlette(routes=[Mount("/", app=mcp_app)], lifespan=lifespan)
```

Without the lifespan step, the first request fails with
"Task group is not initialized". Put your own routes **before** `Mount("/")`.

## Connect to a desktop host

```json
{ "mcpServers": { "support": {
    "command": "uv",
    "args": ["--directory", "/FULL/PATH/support-mcp", "run", "server.py"] } } }
```

Use full paths. You may need the full path to `uv` (`which uv`).

## Server checklist

1. Check every input yourself, even with a schema.
2. Raise `ToolError` with advice the model can use.
3. Make writes safe to retry.
4. Limit and paginate results.
5. Put timeouts on calls to other services.
6. Keep no per-connection state. Use handles.
7. Log to stderr. Never put secrets in descriptions or results.

## Exercise

1. Build the server above and open it with `mcp dev`.
2. Add `search_orders(customer_id, status=None, limit=10)`.
3. Add a tool that fails on purpose. Read the error text the model gets.
4. Run it over HTTP and call it with the client from `05`.
