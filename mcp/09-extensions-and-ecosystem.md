# MCP 09: Extensions and Ecosystem

This file is an overview. Read the linked pages before you build.

## Extensions

Optional add-ons outside the core protocol. Both sides must support one to
use it. Clients list their extensions on each request. Servers list theirs in
`server/discover` (field `extensions`).

### Tasks (`io.modelcontextprotocol/tasks`)

For long jobs: a report that takes ten minutes, a big import.

- The server returns a **task handle** instead of the final result.
- The client checks on it with `tasks/get`.
- The client can send input during the task with `tasks/update`.
- The handle lasts, so the client can disconnect and come back.

Use it when a tool may run longer than a normal request timeout. For shorter
work, progress notifications are enough.

Old tutorials show `tasks/result` and `tasks/list`. Both are gone.

Docs: https://modelcontextprotocol.io/extensions/tasks/overview

### MCP Apps

A server can send **interactive UI** (charts, forms, players). The host
shows it inside the chat.

Docs: https://modelcontextprotocol.io/extensions/apps/overview

### Others

- **Skills over MCP**: sending step-by-step instructions for agent workflows.
- **Auth extensions**: extra authorization methods, in the repo
  `modelcontextprotocol/ext-auth`.

The MCP site has an Extension Support Matrix that shows which hosts support
which extension.

## The MCP Registry

The official public list of MCP servers.

- Authors publish a description of their server and how to install or reach
  it.
- You must prove you own the namespace to publish.
- Hosts read the registry so users can find servers.

**A registry listing does not mean a server is safe.** Review it yourself.

Docs: https://modelcontextprotocol.io/registry/about

## SDKs

Official SDKs exist for several languages. Python and TypeScript are used
most.

- Python: package `mcp`, v2. https://py.sdk.modelcontextprotocol.io
- TypeScript: `@modelcontextprotocol/sdk` (**Not checked**).

## Hosts and reference servers

- Desktop assistants, IDE assistants and coding agents act as MCP hosts.
  Support differs. Check which spec version and features each one has.
- The MCP organisation on GitHub keeps reference servers (filesystem, git,
  fetch). Read them to learn good tool design.

## MCP next to other protocols

| Protocol | Connects | Purpose |
| --- | --- | --- |
| MCP | Agent ↔ tools and data | Give a model context and actions |
| A2A (Agent2Agent) | Agent ↔ agent | Agents hand tasks to each other |
| Function calling | Model ↔ functions in the same app | Tool use, tied to one provider |

They work together. An agent can use MCP for tools and A2A to talk to other
agents.

## How the spec changes

- Changes are proposed as **SEPs**, as pull requests.
- A feature is Active, Deprecated or Removed. A deprecated feature stays for
  at least 12 months.
- Each revision has a changelog and a list of deprecated features.

## Staying current

1. Read the changelog for each new revision:
   https://modelcontextprotocol.io/specification/2026-07-28/changelog
2. Read "What's new" in the Python SDK docs after each big release.
3. Check the date on every tutorial. Before mid-2026 means the old stateful
   protocol and the v1 SDK.

## Check yourself

1. When do you use Tasks instead of progress notifications?
2. Why is a registry listing not a safety promise?
3. MCP versus A2A, in one sentence.
