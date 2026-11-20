# MCP 02: Protocol Basics

All of this matches spec `2026-07-28`.

## Messages are JSON-RPC 2.0

```json
{ "jsonrpc": "2.0", "id": 1, "method": "tools/list", "params": {} }
```

| Kind | Looks like | Sent by |
| --- | --- | --- |
| Request | Has `id` and `method` | Client only |
| Response | Has the same `id`, plus `result` or `error` | Server only |
| Notification | Has `method`, no `id`, no reply | Both |

**Servers never send requests.** If a server needs something from the client,
it uses MRTR (below).

## Stateless: every request stands alone

There is no `initialize` handshake and no session. Each request carries its
own details in `params._meta`:

```json
"_meta": {
  "io.modelcontextprotocol/protocolVersion": "2026-07-28",
  "io.modelcontextprotocol/clientInfo": { "name": "MyAgent", "version": "1.0.0" },
  "io.modelcontextprotocol/clientCapabilities": {}
}
```

What this means:

- Any server copy can answer any request. You can put servers behind a normal
  load balancer. No sticky sessions.
- `tools/list` must be the same for every connection. It may differ by the
  caller's permissions (token scopes).
- `ping` and `logging/setLevel` are gone.

## `server/discover`

Every server must have it. It returns the server's supported versions,
capabilities and name. A client may call it first, or skip it. If the client
asks for a version the server does not support, the server replies with
`UnsupportedProtocolVersionError` and a list of versions it does support.

## `resultType`

Every result has one:

- `"complete"`: the final answer.
- `"input_required"`: the server needs more input first.

## MRTR: how a server asks for input

1. Client sends `tools/call`.
2. Server replies `resultType: "input_required"` with `inputRequests` (the
   questions) and `requestState` (an opaque string).
3. Client gets the answers, for example by asking the user.
4. Client sends the **same request again** with a **new `id`**, plus
   `inputResponses` and the same `requestState`.
5. Server replies with the final result.

```json
{ "resultType": "input_required",
  "inputRequests": {
    "confirm": { "method": "elicitation/create",
                 "params": { "mode": "form", "message": "Cancel order A100?",
                             "requestedSchema": { "type": "object",
                               "properties": { "ok": { "type": "boolean" } } } } } },
  "requestState": "eyJ..." }
```

The server remembers nothing between the two calls. `requestState` must hold
everything it needs. It comes back from the client, so it is untrusted:
sign or encrypt it. (The Python SDK does this for you.)

## Keeping state: use handles

No session means no hidden memory. If you need state (a cart, a browser
page), one tool returns a **handle** and later tools take it as an argument:

```
create_basket()           -> { "basket_id": "bsk_a1b2c3" }
add_item(basket_id, sku)  -> ...
```

Handle rules:

- Check the caller's permission on **every** call. A handle is a name, not a
  permission.
- Make it random and hard to guess (UUIDv4).
- Give it an expiry time and say so in the tool description.
- For an expired or unknown handle, return a clear tool error.

## Change notifications

The client opens one long request, `subscriptions/listen`, and picks what it
wants: `toolsListChanged`, `promptsListChanged`, `resourcesListChanged`,
`resourceSubscriptions`. The server streams matching notifications on it.

Progress and log messages for one request travel on that request's own
response, not on the listen stream.

## Caching

List and read results carry `ttlMs` (how long the client may cache) and
`cacheScope` (`"public"` or `"private"`). Return tools in the **same order**
every time. This helps client caching and LLM prompt caching.

## Error codes

| Code | Meaning |
| --- | --- |
| `-32601` | Method not found |
| `-32602` | Invalid params. Also unknown tool and resource not found |
| `-32020` | `HeaderMismatch`: HTTP header does not match the body |
| `-32021` | `MissingRequiredClientCapability` |
| `-32022` | `UnsupportedProtocolVersion` |

## Old and new side by side

| Topic | Up to `2025-11-25` | `2026-07-28` |
| --- | --- | --- |
| Connection | `initialize` handshake, session | None. Details on every request |
| Session header | `Mcp-Session-Id` | Removed |
| Server asks client | Server sends a request | `input_required`, client retries |
| Change notifications | GET stream, `resources/subscribe` | `subscriptions/listen` |
| Broken stream | Resume with `Last-Event-ID` | Send the request again, new `id` |
| Tasks | Experimental, in core | Official extension |
| Sampling, Roots, Logging | Active | Deprecated |

## Check yourself

1. Why is a stateless server easier to scale?
2. A tool needs the user to confirm. List the messages in order.
3. Where does a stateless server keep a shopping cart?
