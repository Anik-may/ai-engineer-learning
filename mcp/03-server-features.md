# MCP 03: Tools, Resources, Prompts, Elicitation

## Tools

Functions the model can call. The model decides when. The host decides
whether to allow it.

### Tool definition

| Field | Purpose |
| --- | --- |
| `name` | Unique id. 1 to 128 characters: letters, digits, `_`, `-`, `.` |
| `title` | Optional display name |
| `description` | What it does. **The model reads this to choose the tool** |
| `inputSchema` | JSON Schema for the arguments |
| `outputSchema` | Optional JSON Schema for the result |
| `annotations` | Optional hints: `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint` |

Clients must **not trust annotations** unless they trust the server.

### Call and result

```json
{ "method": "tools/call",
  "params": { "name": "get_weather", "arguments": { "location": "Dhaka" } } }
```

```json
{ "resultType": "complete",
  "content": [ { "type": "text", "text": "{\"temperature\": 32}" } ],
  "structuredContent": { "temperature": 32 },
  "isError": false }
```

- `content` types: `text`, `image`, `audio`, `resource_link`, `resource`.
- `structuredContent` can be any JSON value. If the tool has an
  `outputSchema`, the result **must** match it.
- Also put the same data as JSON text in `content`, for older clients.

### Two kinds of error

| Kind | When | How | Model sees it? |
| --- | --- | --- | --- |
| Protocol error | Unknown tool, broken request, server crash | JSON-RPC `error` | Usually no |
| Tool error | API failed, bad value, business rule | Normal result with `isError: true` | Yes, so it can fix and retry |

Write tool errors as advice the model can act on:
"Invalid date: must be in the future. Today is 2026-11-21."

### How to write good tools

1. Name by action + object: `search_orders`, not `orders`.
2. Say when to use it, and when not to.
3. Describe every parameter. Give the format and an example.
4. A few clear tools beat many similar ones.
5. Return only what is needed. Big results waste the model's context.
6. Limit and paginate by default.
7. Make write tools safe to retry (take an idempotency key).
8. Keep read and write in separate tools, so hosts can approve them
   differently.

## Resources

Read-only data with a URI. The **app or user** picks what to load.

- `resources/list`: what is available.
- `resources/read`: content for a URI (text or base64).
- `resources/templates/list`: URI patterns such as `orders://{order_id}`.
- Not found returns error `-32602`.
- For updates, the client asks for `resourceSubscriptions` on
  `subscriptions/listen`.

| Use a resource when | Use a tool when |
| --- | --- |
| It is data to read | It does an action or a calculation |
| The app or user picks it | The model should decide |
| It has a natural address | It takes free arguments |

## Prompts

Ready-made templates the **user** picks, often shown as slash commands.

- `prompts/list`: names, descriptions, arguments.
- `prompts/get`: returns the messages to send to the model.

## Elicitation

The server asks the user for input during a request. It travels through MRTR
(see `02`) as an `elicitation/create` input request.

| Mode | Use for | How |
| --- | --- | --- |
| `form` | Simple input | A flat JSON Schema. The client shows a form |
| `url` | Login, payment, secrets | The client opens a URL in the browser. The data never passes through the model |

The user can `accept`, `decline` or `cancel`.

Rules:

- **Never** ask for passwords, API keys or card numbers in form mode. Use
  URL mode.
- Keep forms small: simple fields only (text, number, yes/no, choice).
- Handle decline and cancel well.

## Small helpers

- **Progress**: `notifications/progress`, on the request's own response.
- **Cancel**: on stdio send `notifications/cancelled`. On HTTP the client
  closes the response stream.
- **Pagination**: `cursor` in, `nextCursor` out.
- **Tracing**: OpenTelemetry values (`traceparent`, `tracestate`, `baggage`)
  can travel in `_meta`. One trace can cover the agent and the server.

## Deprecated: do not build on these

| Feature | Use this |
| --- | --- |
| Sampling (server asks the client's model to write text) | Call an LLM API from the server |
| Roots (client tells the server which folders matter) | Pass paths as tool arguments, resource URIs or config |
| Logging over the protocol | Log to stderr, or use OpenTelemetry |
| HTTP+SSE transport | Streamable HTTP |

They still work for at least 12 months after the revision.

## Check yourself

1. Your tool's upstream API returns HTTP 500. Protocol error or tool error?
2. "Customer profile for id 123": tool or resource?
3. Why must login use URL mode, not a form?
