# MCP 01: What MCP Is

Checked against the official docs on 21 November 2026 (spec `2026-07-28`,
Python SDK v2). Parts I could not check are marked **Not checked**.

## The problem and the idea

AI apps need tools and data: search a database, read a file, create a ticket.
Before MCP, every app built every connection in its own way. With M apps and
N systems, that is M × N pieces of work.

**MCP (Model Context Protocol)** is an open standard that fixes this. A system
shares its features once, as an **MCP server**. Any app that speaks MCP can
use it. M × N becomes M + N.

## Three roles

| Role | What it is | Example |
| --- | --- | --- |
| **Host** | The AI app the user talks to | A chat app, an IDE assistant, your agent service |
| **Client** | A connector inside the host. One per server | The `Client` object your agent creates |
| **Server** | A program that offers tools, resources and prompts | A CRM server, a GitHub server, your own |

```
┌───────────────── Host (AI app) ─────────────────┐
│                                                 │
│  ┌─────┐    ┌────────────┐    ┌──────────┐      │  stdio   ┌───────────────┐
│  │ LLM │◄──►│            │◄──►│ Client A │──────┼─────────►│   Server A    │
│  └─────┘    │            │    └──────────┘      │          │ (local files) │
│             │ Agent loop │                      │          └───────────────┘
│             │            │    ┌──────────┐      │  HTTP    ┌───────────────┐
│             │            │◄──►│ Client B │──────┼─────────►│   Server B    │
│             └────────────┘    └──────────┘      │          │ (remote CRM)  │
│                                                 │          └───────────────┘
└─────────────────────────────────────────────────┘
```

The model never talks to a server directly. The host chooses which tools the
model can see, asks the user for approval, and makes the call.

## What a server offers

| Feature | Who decides to use it | What it is |
| --- | --- | --- |
| **Tools** | The model | Functions to call: `search_orders`, `create_ticket` |
| **Resources** | The app | Read-only data with a URI: a file, a record |
| **Prompts** | The user | Ready-made prompt templates, often slash commands |

Most servers are mostly tools.

The client offers one thing back: **elicitation**. The server can ask the
user for more input in the middle of a task (a missing field, a yes/no, a
login).

## When to use MCP

Use it when:

- many apps or agents need the same integration,
- users should bring their own tools into your app,
- another team owns the system,
- you want to plug into existing hosts (desktop assistants, IDEs).

Skip it when only one agent uses the tool and the tool lives in the same
code. A plain function tool is simpler.

## Versions

Each MCP version is named by a date.

| Version | Main change |
| --- | --- |
| `2024-11-05` | First release. HTTP+SSE transport |
| `2025-03-26` | Streamable HTTP, OAuth authorization |
| `2025-06-18` | Structured tool output, elicitation |
| `2025-11-25` | Experimental tasks, URL-mode elicitation |
| `2026-07-28` | **Stateless.** No handshake, no sessions. MRTR. Tasks moved to an extension |

`2026-07-28` is the current one. Most tutorials online still teach the old
design. If a tutorial uses `initialize`, `FastMCP` or `ClientSession`, it is
old.

## Words to know

- **JSON-RPC**: the message format MCP uses.
- **Transport**: how messages travel. Either stdio or Streamable HTTP.
- **Capability**: a feature a client or server says it supports.
- **MRTR**: Multi Round-Trip Requests. How a server asks the client for input.
- **Extension**: an optional add-on outside the core (Tasks, MCP Apps).
- **SEP**: Specification Enhancement Proposal. How the spec changes.

## Check yourself

1. In your own agent project, what is the host, the client and the server?
2. Why does the model control tools but the app controls resources?
3. Give one case where a plain function tool is better than an MCP server.
