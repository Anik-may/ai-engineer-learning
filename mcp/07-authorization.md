# MCP 07: Authorization

The hardest MCP topic, and a common senior interview question.

## Basics

- Authorization is **optional**.
- It is only for **HTTP**. stdio servers read secrets from the environment.
- It is built on **OAuth 2.1**.

## Who is who

| OAuth role | In MCP |
| --- | --- |
| Resource server | The MCP server. It accepts tokens |
| Client | The MCP client. It acts for the user |
| Authorization server | Gives out tokens. Often your identity provider |
| Resource owner | The user |

Key point: the MCP server **only checks tokens**. It never issues them.

## The flow, step by step

1. Client calls the server with no token.
2. Server replies **401** and says where to look:

   ```http
   WWW-Authenticate: Bearer resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource",
                            scope="files:read"
   ```

3. Client reads that URL (**Protected Resource Metadata**, RFC 9728). It
   names the authorization server.
4. Client reads the authorization server's metadata (RFC 8414 or OpenID
   Connect Discovery).
5. Client gets a client id (see below).
6. Client starts the login with **PKCE** and the **`resource`** parameter
   (RFC 8707), which names the MCP server.
7. User logs in and agrees in the browser.
8. Browser comes back with a code and `iss`. Client checks `iss` is the
   issuer it expected (RFC 9207).
9. Client swaps the code for an access token, sending `resource` again.
10. Client sends the token on **every** request:
    `Authorization: Bearer <token>`

## Three ways to get a client id

| Way | Status | How |
| --- | --- | --- |
| **Client ID Metadata Document (CIMD)** | Preferred | The `client_id` is an HTTPS URL. The auth server reads a JSON file about the client from it |
| Pre-registration | Fine | Client id set up by hand before |
| Dynamic Client Registration (RFC 7591) | **Deprecated** | Client POSTs to `/register` |

## Server rules

1. Publish Protected Resource Metadata.
2. **Check the token was made for this server** (audience check).
3. 401 for a missing, bad or expired token.
4. 403 with `error="insufficient_scope"` if a scope is missing.
5. **Never accept a token made for another service.**
6. **Never forward the client's token to another API** (token passthrough).
   Use the server's own credentials for that.

## Client rules

1. Send `resource` in both the login request and the token request.
2. Use PKCE.
3. Check `iss` before using the code.
4. Token goes in the `Authorization` header, never in the URL.
5. Send a token only to the server it was made for.
6. Store credentials per issuer. Never reuse them with another auth server.

## Scopes and step-up

- Start with the smallest set of scopes.
- When a call needs more, the server replies 403 and names the scope:

  ```http
  WWW-Authenticate: Bearer error="insufficient_scope", scope="files:write"
  ```

- The client logs in again asking for old + new scopes, then retries. This is
  **step-up authorization**.
- Make scopes narrow: `orders:read`, `tickets:write`. Avoid `admin` or `*`.

| Code | Meaning |
| --- | --- |
| 401 | No token, or token bad or expired |
| 403 | Token is fine but lacks a scope or permission |

## In the Python SDK

```python
from pydantic import AnyHttpUrl
from mcp.server import MCPServer
from mcp.server.auth.middleware.auth_context import get_access_token
from mcp.server.auth.provider import AccessToken, TokenVerifier
from mcp.server.auth.settings import AuthSettings

class MyVerifier(TokenVerifier):
    async def verify_token(self, token: str) -> AccessToken | None:
        ...   # check signature, expiry, audience. Return None to reject

mcp = MCPServer(
    "support",
    token_verifier=MyVerifier(),
    auth=AuthSettings(
        issuer_url=AnyHttpUrl("https://auth.example.com"),
        resource_server_url=AnyHttpUrl("https://mcp.example.com/mcp"),
        required_scopes=["orders:read"],
        validate_token_resource=True,      # the audience check
    ),
)
```

`token_verifier` and `auth` always go together. Auth is skipped on stdio and
in the in-memory test client.

## Check permissions twice

The token tells you **who** is calling. Each tool must still check **what
they may touch**:

```python
@mcp.tool()
def get_order(order_id: str) -> Order:
    caller = get_access_token()                  # who is calling
    order = db.get(order_id)
    if order is None or order.owner != caller.client_id:
        raise ToolError("Order not found.")      # same message for both cases
    ...
```

Never let the model enforce access rules. Never trust an id the model sends
without checking it against the caller.

## What to do in your projects

| Situation | Do this |
| --- | --- |
| Local stdio server | No OAuth. Read an API key from the environment |
| Internal HTTP server, only your backend calls it | A service token or mutual TLS inside the network |
| Multi-user or public HTTP server | Full OAuth with an identity provider |

Do not write OAuth yourself. Use the SDK or a library.

## RFCs to know by name

| RFC | Topic |
| --- | --- |
| 7591 | Dynamic client registration (deprecated here) |
| 8414 | Authorization server metadata |
| 8707 | `resource` parameter |
| 9207 | `iss` in the response |
| 9728 | Protected resource metadata |

## Check yourself

1. Why must the client send `resource`?
2. What is token passthrough, and why is it forbidden?
3. A user with a valid token asks for another tenant's order. Where is it
   stopped?
4. What replaced Dynamic Client Registration?
