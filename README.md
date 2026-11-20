# AI Engineer Learning Notes

Study material for the 26-week AI engineer roadmap. Written 20 November 2026.

## How the folders map to the roadmap

| Roadmap week | Topic | Where to study |
| --- | --- | --- |
| 3 | Evaluation basics | `langsmith/03`, `langsmith/04`, `langsmith/05` |
| 4 | Build the evaluated RAG assistant | `langsmith/04`, `langsmith/06` |
| 5 | Tracing, cost, latency | `langsmith/02`, `langsmith/07`, all of `langfuse/` |
| 6 | Guardrails and AI security | `mcp/08` plus `lessons/week-08` references |
| 7 | MCP and agent reliability | all of `mcp/` |
| 8 to 26 | One lesson per week | `lessons/week-08.md` to `lessons/week-26.md` |

## Folders

- `langsmith/` LangSmith from setup to production monitoring (8 files).
- `langfuse/` Langfuse, the open-source alternative (7 files).
- `mcp/` Model Context Protocol: protocol, servers, clients, auth, security (11 files).
- `lessons/` Weekly lessons for weeks 8 to 26 (19 files).

## How accurate is this

These tools change fast. On 20 November 2026 I checked the official docs for
the parts marked **Verified**. Parts marked **From memory** were not checked
and may be out of date; confirm them in the docs before relying on them.

Three things changed recently that older tutorials get wrong:

1. **MCP is now stateless** (spec revision `2026-07-28`). The `initialize`
   handshake and sessions are gone.
2. **The MCP Python SDK is v2.** `FastMCP` is now `MCPServer`, and
   `ClientSession` is replaced by `Client`.
3. **LangChain's MCP support moved** from the `langchain-mcp-adapters`
   package into `langchain[mcp]`.

## Official docs

- LangSmith: https://docs.langchain.com/langsmith
- Langfuse: https://langfuse.com/docs
- MCP: https://modelcontextprotocol.io
- MCP Python SDK: https://py.sdk.modelcontextprotocol.io
