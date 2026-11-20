# MCP 11: Portfolio Project and Interview Questions

## Project: MCP agent with approvals

A support agent that finishes a multi-step task through your own MCP server,
with retries and a human approval step.

It shows: agents, tool calling, current MCP, reliability, human-in-the-loop,
and security awareness.

### The server (`support-mcp`, Python SDK v2, Streamable HTTP)

| Tool | Type | Notes |
| --- | --- | --- |
| `search_orders(customer_id, status?, limit=10)` | Read | Paginated, tenant-checked |
| `get_order(order_id)` | Read | Structured output |
| `create_ticket(subject, body, idempotency_key)` | Write | Safe to retry |
| `draft_email(to, subject, body)` | Write | Makes a draft only. Never sends |
| `send_email(draft_id)` | Write, destructive | Needs approval |

Plus one resource (`policy://refunds`) and one prompt (`summarise_ticket`).

### The agent (LangGraph)

1. Look up the customer's orders.
2. Work out the problem, using the refund policy resource.
3. Create a ticket and draft a reply.
4. **Pause for human approval.** Show the draft and the tool arguments.
5. If approved, send. If rejected, revise.
6. Update the ticket.

### Reliability to build in

- Retry with exponential backoff on temporary errors (max 3).
- Timeout on every tool call.
- Idempotency keys on writes, so a retry cannot create a duplicate.
- A fallback when a tool keeps failing: tell the user and stop.
- A step limit and a cost limit per run.
- Checkpointing, so an approved run continues after a restart.

### Make it measurable

- A 30-example dataset: request → expected tools in order → expected result.
- Metrics: task success rate, tool-selection accuracy, average steps, cost
  per task, p95 latency.
- Failure injection: make `create_ticket` fail 30% of the time. Show the
  success rate with and without retries.
- 10 injection attacks, with the attack success rate.

### README

1. One paragraph and a 60-second demo video or GIF.
2. Architecture diagram: host, client, server, data store.
3. How to run with Docker Compose.
4. Results table (the metrics above).
5. Security: what needs approval, what is blocked, test results.
6. "What I would do next".

### Stretch goals

- OAuth on the server with step-up scopes.
- The Tasks extension for a long "export all orders" tool.
- One OpenTelemetry trace across the agent and the server.

## Interview questions

Each answer is in the file shown.

### Fundamentals (`01`, `03`, `06`)

1. What problem does MCP solve? What is M × N versus M + N?
2. Name the three roles with an example of each.
3. Tool, resource, prompt: what is each, and who controls it?
4. What message format does MCP use?
5. What are the two transports, and when do you use each?

### Current protocol (`02`)

6. MCP became stateless in `2026-07-28`. What was removed, and why does it
   help?
7. How does a server ask the user a question if it cannot send requests?
8. How does a stateless server keep a shopping cart across calls?
9. What is `server/discover` for?
10. Protocol error versus tool error: what is the difference, and which one
    does the model see?
11. Which features are deprecated, and what replaces them?

### Design (`03`, `05`, `09`)

12. You have 60 tools on 8 servers and the model picks badly. What do you do?
13. How do you write a tool description the model uses correctly?
14. A tool takes 15 minutes. How do you offer it?
15. When is a plain function tool better than an MCP server?
16. How do you make a write tool safe to retry?

### Security and authorization (`07`, `08`)

17. What is tool poisoning? What is a rug pull?
18. How can a tool result attack your agent, and how do you defend?
19. What is token passthrough, and why is it forbidden?
20. Explain the confused deputy problem for an MCP proxy.
21. Why must a local HTTP server check the `Origin` header?
22. The model sends an order id from another tenant. Where is it stopped?
23. Walk through the OAuth flow from the first 401.
24. Why does the client send the `resource` parameter?
25. What is step-up authorization?
26. How does a stdio server get credentials?

### Practical (`10`)

27. Your stdio server crashes at startup with garbage output. First guess?
28. How do you unit-test an MCP server without a network?
29. How do you measure whether the model picks the right tool?
30. Describe an agent you built: its tools, how it fails, how you measured it.

## Model answers for the six asked most

**Q6.** The `initialize` handshake, sessions and `Mcp-Session-Id` were
removed. Every request now carries the protocol version and client
capabilities in `_meta`. Any server copy can answer any request, so servers
scale behind a normal load balancer and fit serverless platforms.

**Q7.** The server returns `resultType: "input_required"` with
`inputRequests` and an opaque `requestState`. The client collects the
answers and sends the original request again, with a new `id`,
`inputResponses` and the same `requestState`. This is MRTR.

**Q10.** A protocol error is a JSON-RPC error, for things like an unknown
tool or a broken request. A tool error is a normal result with
`isError: true` and readable text. The client gives it to the model, so the
model can fix its arguments and retry.

**Q18.** A tool result can contain instructions (prompt injection). Treat
results as data, mark them with delimiters, require approval for side
effects, remove risky tools after reading untrusted content, and test with
an injection dataset.

**Q19.** It means accepting a token that was not made for this server and
forwarding it to another API. It skips audience checks, rate limits and
audit logs. The server must check the audience and use its own credentials
for downstream calls.

**Q22.** Inside the tool. On every call the server compares the object's
tenant with the logged-in caller. Never rely on the model or on a handle for
access control.
