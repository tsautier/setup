## 1. Transport overview

- The **streamable HTTP** transport (introduced in MCP spec 2025-03-26, replacing the older HTTP+SSE transport) uses a single HTTP endpoint (e.g. `https://server.example.com/mcp`) for all communications
- The client sends **JSON-RPC 2.0** messages over HTTP `POST`, and the server can respond with either a single JSON response or an `SSE` (Server-Sent Events) stream
- An optional `GET` on the same endpoint opens a standalone SSE stream for server-initiated notifications

### 1.1. Initialization handshake

#### → _Client_ request: `initialize`

`POST` **JSON-RPC** request with `"method": "initialize"`:

```http
POST /mcp HTTP/1.1
Host: server.example.com
Content-Type: application/json
Accept: application/json, text/event-stream

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-03-26",
    "capabilities": {
      "roots": { "listChanged": true }
    },
    "clientInfo": {
      "name": "MyAgent",
      "version": "1.0.0"
    }
  }
}
```

#### ← _Server_ response capabilities

Server replies with capabilities (HTTP 200, `application/json`):

```http
HTTP/1.1 200 OK
Content-Type: application/json
Mcp-Session-Id: abc123-session-id

{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-03-26",
    "capabilities": {
      "tools": { "listChanged": true }
    },
    "serverInfo": {
      "name": "WeatherServer",
      "version": "2.0.0"
    }
  }
}
```

> [!Note]
>
> _Client_ **must** include the `Mcp-Session-Id: abc123-session-id` header received from _Server_ in all subsequent requests

#### → _Client_ sends: `initialized` notification

```http
POST /mcp HTTP/1.1
Host: server.example.com
Content-Type: application/json
Mcp-Session-Id: abc123-session-id

{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

#### ← Server response: `HTTP 202 Accepted` (no body, since notifications have no response)

### 1.2. Listing Tools

#### → _Client_ request: `tools/list`

```http
POST /mcp HTTP/1.1
Host: server.example.com
Content-Type: application/json
Accept: application/json, text/event-stream
Mcp-Session-Id: abc123-session-id

{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list",
  "params": {}
}
```

#### ← Server response: tool definitions with JSON schema parameters

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "description": "Get current weather for a city",
        "inputSchema": {
          "type": "object",
          "properties": {
            "city": {
              "type": "string",
              "description": "City name, e.g. 'London'"
            },
            "units": {
              "type": "string",
              "enum": ["celsius", "fahrenheit"],
              "description": "Temperature unit"
            }
          },
          "required": ["city"]
        }
      },
      {
        "name": "get_forecast",
        "description": "Get 5-day weather forecast",
        "inputSchema": {
          "type": "object",
          "properties": {
            "city": {
              "type": "string"
            },
            "days": {
              "type": "integer",
              "minimum": 1,
              "maximum": 5
            }
          },
          "required": ["city"]
        }
      }
    ]
  }
}
```

> [!Important]
>
> The `inputSchema` is a standard **JSON schema** object
>
> The LLM/agent framework reads this schema to know exactly what parameters to provide
>
> The schema includes tool name, descriptions, required fields, types, enums, defaults, etc; everything a model needs to construct a valid call

### 1.3. Calling a tool

#### → _Client_ request: `tools/call`

````http
POST /mcp HTTP/1.1
Host: server.example.com
Content-Type: application/json
Accept: application/json, text/event-stream
Mcp-Session-Id: abc123-session-id

{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "London",
      "units": "celsius"
    }
  }
}
````

#### ← Server response:

##### A. Simple, non-streaming

````http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "London: 15°C, partly cloudy, humidity 72%"
      }
    ],
    "isError": false
  }
}
````

##### B. Long-running tools: streaming via SSE

← When _Server_ needs to stream progress or the operation is long-running, it responds with `Content-Type: text/event-stream`:

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream

event: message
data: {"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"London: 15°C, partly cloudy"}],"isError":false}}
```

← _Server_ may also send **progress notifications** before the final SSE event:

```http
event: message
data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"tok-3","progress":50,"total":100}}
```

← The final SSE event contains the JSON-RPC result with the matching `id`:

```http
event: message
data: {"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"Done. London: 15°C"}],"isError":false}}
````

### 1.4. Server-to-Client stream (`GET`)

→ _Client_ can open a `GET` request to listen for server-initiated messages (notifications, requests):

````http
GET /mcp HTTP/1.1
Host: server.example.com
Accept: text/event-stream
Mcp-Session-Id: abc123-session-id
````

← _Server_ holds this open and sends SSE events like `notifications/tools/list_changed` when tools change; this replaces the dedicated SSE endpoint from the old transport

### 1.5. Session Termination

→ _Client sends:

````http
DELETE /mcp HTTP/1.1
Host: server.example.com
Mcp-Session-Id: abc123-session-id
````

_Server_ response: `HTTP 200 OK` and invalidates the session

### 1.6. HTTP methods in MCP

| HTTP Method | Purpose | Body | Response |
|---|---|---|---|
| `POST` | All client→server JSON-RPC messages (requests, notifications, batches) | JSON-RPC message(s) | `application/json` or `text/event-stream` |
| `GET` | Open server→client SSE stream for server-initiated messages (e.g. `notifications/tools/list_changed`) | _None_ | `text/event-stream` |
| `DELETE` | Client sends `DELETE` to the endpoint with `Mcp-Session-Id` to cleanly close the session | _None_ | 200 OK |

## 2. OAuth 2.0 authentication for MCP servers

MCP uses **OAuth 2.1** (RFC 6749 profile) with **PKCE** (RFC 7636) for authorization

### 2.1. Step-by-step flow

#### Step 1: _Client_ tries to access the MCP server without a token

← _Server_ response: 

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer resource_metadata="https://server.example.com/.well-known/oauth-protected-resource"
```

#### Step 2: _Client_ fetches _protected resource_ metadata

→ _Client_ request:

```http
GET /.well-known/oauth-protected-resource HTTP/1.1
Host: server.example.com
```

← _Server_ response:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "resource": "https://server.example.com/mcp",
  "authorization_servers": [
    "https://idp.example.com"
  ],
  "bearer_methods_supported": [
    "header"
  ]
}
```

#### Step 3: Client fetches the _authorization server_ metadata** (RFC 8414)

→ _Client_ request:

````http
GET /.well-known/oauth-authorization-server HTTP/1.1
Host: idp.example.com
````

← _Server_ response:

````http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "issuer": "https://idp.example.com",
  "authorization_endpoint": "https://idp.example.com/authorize",
  "token_endpoint": "https://idp.example.com/token",
  "registration_endpoint": "https://idp.example.com/register",
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "code_challenge_methods_supported": ["S256"]
}
````

#### Step 4: Dynamic Client Registration (optional, RFC 7591)

> [!Tip]
>
> Some identity providers like Entra does not support DCR and requires clients to be pre-registered

→ _Client_ request:

```http
POST /register HTTP/1.1
Host: idp.example.com
Content-Type: application/json

{
  "client_name": "MyAgent",
  "redirect_uris": ["http://localhost:9999/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
```

← _Server_ response: `client_id`

#### Step 5: MCP Client redirects user to sign in

→ _Client_ opens the user's browser to the authorization endpoint:

```url
https://idp.example.com/authorize?
  response_type=code&
  client_id=abc123&
  redirect_uri=http://localhost:9999/callback&
  code_challenge=HkmpvI2t_krWGIBwyPqWdZzOz4uqzOt7h0hgrv38aN8
  code_challenge_method=S256&
  state=random-state-value
  scope=mcp:tools
```

← _User_ authenticates in the browser (password, MFA, etc) and is redirected back to _Client_'s local redirect URI with an **authorization code**

````url
http://localhost:9999/callback?code=AUTH_CODE_HERE&state=random-state-value
````

##### Step 6: _Client_ redeems the authorization code for access + refresh token

→ _Client_ request:

````http
POST /token HTTP/1.1
Host: idp.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=AUTH_CODE_HERE&
redirect_uri=http://localhost:9999/callback&
client_id=abc123&
code_verifier=I0JROoeYuAwrW40X42rDW9rAmwLZz7_yijbhqEBalgDQMbjCzaL_l27KXQUDFKwWsBZY6V5E9XhbG20mfQz6hkvbC0CBKixeSVgIpHyGzmSZcp6M5aGCFjPhxiClQ5UV
````

← _Server_ response:

```http
{
  "access_token": "eyJhbGciOi...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "dhl8Yp3..."
}
```

##### Step 7: _Client_ uses token on all MCP requests

````http
POST /mcp HTTP/1.1
Host: server.example.com
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json
Mcp-Session-Id: abc123-session-token

{ "jsonrpc": "2.0", "id": 2, "method": "tools/list", "params": {} }
````

##### Step 8: _Client_ refreshs access token

When the access token expires (server returns `HTTP 401`), _Client_ refreshes the access token using the refresh token:

````http
POST /token HTTP/1.1
Host: idp.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&
refresh_token=dhl8Yp3...&
client_id=abc123
````

### 2.2. Who Handles Token Request & Refresh?

The **MCP client SDK** (e.g. `@modelcontextprotocol/sdk`) handles the HTTP requests, session management, and owns the entire OAuth lifecycle:
- The **MCP client** handles all OAuth token management - not the LLM, and not the agent framework (unless the agent framework *is* the MCP client)
- The agent framework just passes the server URL and possibly a client ID
- The LLM never sees tokens, it only sees the tool names, descriptions, and schema

| Actor | Responsibility |
|---|---|
| MCP client | Discovering auth metadata |
| MCP Client (opens browser) | Redirecting user to sign in |
| MCP Client (local listener) | Receive authorization code from user sign in |
| MCP Client | Redeem for access + refresh token |
| MCP Client | Storing and attaching access tokens to `Authorization: Bearer` header |
| MCP Client | Detecting 401 and refreshing expired access tokens (`grant_type=refresh_token`) |
| LLM | Does **nothing** with auth - it only sees tool schemas and results |
| Agent framework (e.g., LangChain, Semantic Kernel) | Instantiates/configures the MCP client but delegates all OAuth to it |
