# MCP 06: Transports and Deployment

## Two transports

| | stdio | Streamable HTTP |
| --- | --- | --- |
| Where the server runs | A subprocess the client starts | Its own service |
| Who can reach it | Only that client | Many clients over the network |
| Format | One JSON message per line on stdin/stdout | One HTTP POST per message |
| Auth | Secrets from environment variables | OAuth 2.1 (see `07`) |
| Best for | Local tools, desktop hosts, development | Remote and multi-user servers |

The old **HTTP+SSE** transport is deprecated. Do not use it.

### How to choose

| Situation | Transport |
| --- | --- |
| Reads local files or runs local commands | stdio |
| One developer, development only | stdio |
| Shared by a team or customers | Streamable HTTP |
| Needs per-user permissions | Streamable HTTP + OAuth |
| Called by your own backend agent | Streamable HTTP inside your network |

## stdio

- The client starts the server and talks over stdin and stdout.
- stdout is for protocol messages **only**. Log to stderr.
- No network, so it is the safest choice for local tools.

## Streamable HTTP

- One endpoint, for example `https://example.com/mcp`. It takes **POST**.
- The server answers with either one JSON object
  (`application/json`) or a stream (`text/event-stream`) that carries
  progress and then the final result. The client must accept both.

### Required headers

```http
POST /mcp HTTP/1.1
Content-Type: application/json
Accept: application/json, text/event-stream
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: get_weather
```

| Header | Copies | Needed for |
| --- | --- | --- |
| `MCP-Protocol-Version` | The version in `_meta` | Every POST |
| `Mcp-Method` | `method` | Every request |
| `Mcp-Name` | `params.name` or `params.uri` | `tools/call`, `resources/read`, `prompts/get` |
| `Mcp-Param-{Name}` | A tool argument marked `x-mcp-header` | Only when the tool asks for it |

Why repeat the body in headers? Load balancers and firewalls can route and
rate-limit without reading JSON. The body is the truth. If a header does not
match, the server replies HTTP 400 with error `-32020`. Do not mark secret
arguments with `x-mcp-header`: proxies can read headers.

### Removed in `2026-07-28`

- The GET endpoint.
- `Mcp-Session-Id` and HTTP DELETE.
- Stream resume (`Last-Event-ID`). If a stream breaks, send the request
  again with a new `id`.

### Security rules from the spec

1. **Check the `Origin` header.** This stops DNS rebinding (a web page
   calling your local server). Reply 403 if it is wrong.
2. Local servers bind to `127.0.0.1`, not `0.0.0.0`.
3. Authenticate every connection.

## Deploying with the Python SDK

`mcp.run()` starts one process. For more, hand the app to an ASGI server:

```python
# server.py
app = mcp.streamable_http_app()
```

```bash
uvicorn server:app --host 0.0.0.0 --port 8000 --workers 4 \
  --proxy-headers --forwarded-allow-ips='<proxy address>'
```

Things that surprise people:

- **Host check.** By default the app only accepts `localhost` as `Host`. On
  a real domain you get `421 Misdirected Request` until you pass
  `transport_security=TransportSecuritySettings(allowed_hosts=[...],
  allowed_origins=[...])`. List both `"mcp.example.com"` and
  `"mcp.example.com:*"`.
- **Binding to `0.0.0.0` turns that check off.** Then every Host and Origin
  is accepted. Set the allow-list yourself.
- **`json_response=True`** returns plain JSON, but progress messages are
  dropped.
- **`stateless_http`** only matters for old clients. New requests never use
  sessions.
- **Many copies:** give every copy the same `RequestStateSecurity` key (see
  `04`), or an MRTR retry that lands on another copy fails.
- **Health check:** `@mcp.custom_route("/health", methods=["GET"])`. Custom
  routes are never authenticated. Keep them harmless.

### Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev
COPY . .
ENV PYTHONUNBUFFERED=1
EXPOSE 8000
CMD ["uv", "run", "uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Proxies

- Turn off buffering for streams (`X-Accel-Buffering: no` for Nginx).
- For long `subscriptions/listen` streams, send SSE comment lines (`:`) as
  keep-alives so idle timeouts do not close them.

## Production checklist

1. HTTPS in front.
2. Auth on every request.
3. Host and Origin allow-list.
4. No state in the process. State lives in a database or cache, by user.
5. Request size limit and timeouts.
6. Rate limit per user.
7. Health endpoint.
8. Logs to stderr with trace ids. Pass OpenTelemetry context along.
9. Proxy does not buffer streams.
10. Secrets from a secrets manager, never in the image.

Scaling is simple: the server is stateless, so add copies behind a load
balancer. For tools that run for minutes, use the Tasks extension (see `09`).

## Check yourself

1. Why does `print()` break a stdio server?
2. What stops a bad web page from calling an MCP server on localhost?
3. Why can a `2026-07-28` server sit behind a simple round-robin load
   balancer?
