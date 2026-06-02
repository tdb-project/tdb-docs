# MCP Endpoint

TDB exposes a [Model Context Protocol](https://modelcontextprotocol.io/) endpoint
at `POST /v1/mcp`. MCP uses JSON-RPC 2.0 over HTTP.

This endpoint is how Claude Desktop, Cursor, and other AI tools query your data
sources directly — without building a custom integration.

---

## Supported methods

| JSON-RPC method | Auth required | Description |
|---|---|---|
| `initialize` | No | MCP handshake — returns protocol version and capabilities |
| `tools/list` | Yes | Lists available tools |
| `tools/call` | Yes | Executes a tool |

Only `initialize` is unauthenticated. This is intentional — MCP clients must complete
the handshake before presenting credentials, per the MCP spec.

---

## The `query_source` tool

The current release exposes one tool: **`query_source`**.

**Tool schema:**

```json
{
  "name": "query_source",
  "description": "Run a SQL SELECT query against the registered TDB data source. Use 'data' as the table name. Maximum 1,000 rows returned.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "sql": {
        "type": "string",
        "description": "SQL SELECT statement. Use 'data' as the table name."
      },
      "source_name": {
        "type": "string",
        "description": "Optional. Name of the registered source. Defaults to the only registered source."
      }
    },
    "required": ["sql"]
  }
}
```

---

## Authentication

All MCP methods except `initialize` require a Bearer token:

```
Authorization: Bearer <token>
```

Any valid TDB credential works: static env key, DB-managed key, or JWT.

If the token is missing or invalid, TDB returns HTTP 401 with a `WWW-Authenticate`
header that MCP-aware clients use to discover the OAuth authorization server:

```
WWW-Authenticate: Bearer realm="TDB", resource_metadata="/.well-known/oauth-protected-resource"
```

Claude Desktop and Cursor use this header to trigger the OAuth 2.1 PKCE flow
automatically. See [OAuth 2.1 →](../auth/oauth.md).

---

## Request format

All requests are JSON-RPC 2.0 objects sent to `POST /v1/mcp`:

```json
{
  "jsonrpc": "2.0",
  "id": <integer or string>,
  "method": "<method>",
  "params": { ... }
}
```

---

## Method reference

### `initialize`

Completes the MCP handshake. No auth required.

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}'
```

Response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {"tools": {}},
    "serverInfo": {"name": "tdb-community", "version": "0.4.0"}
  }
}
```

---

### `tools/list`

Returns the list of available tools. Requires auth.

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```

Response:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "query_source",
        "description": "Run a SQL SELECT query against the registered TDB data source...",
        "inputSchema": { ... }
      }
    ]
  }
}
```

---

### `tools/call` — `query_source`

Executes a SQL query against a registered source. Requires auth.

Use the table name that matches the source: CSV sources are queried as `data`, while
database sources use their **registered table name** (the examples below query a `customers`
source). There is no `data` alias for database sources.

**With a single registered source (source_name optional):**

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "query_source",
      "arguments": {
        "sql": "SELECT country, COUNT(*) AS n FROM customers GROUP BY country ORDER BY n DESC LIMIT 10"
      }
    }
  }'
```

**With multiple registered sources (source_name required):**

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 4,
    "method": "tools/call",
    "params": {
      "name": "query_source",
      "arguments": {
        "sql": "SELECT * FROM customers WHERE status = '\''active'\'' LIMIT 20",
        "source_name": "customers"
      }
    }
  }'
```

**Successful response:**

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"source\":\"customers\",\"columns\":[\"country\",\"n\"],\"rows\":[{\"country\":\"US\",\"n\":1420},{\"country\":\"GB\",\"n\":380}],\"rows_returned\":2}"
      }
    ]
  }
}
```

The `text` field contains a JSON-serialised result object. AI tools receive this
and can present it as a table or process it programmatically.

**Error response (tool-level error, still HTTP 200):**

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [{"type": "text", "text": "SQL validation error: Only SELECT statements are allowed"}],
    "isError": true
  }
}
```

---

## Error codes

| HTTP status | JSON-RPC error code | Meaning |
|---|---|---|
| 200 | — | Success (check `isError` for tool-level errors) |
| 400 | -32700 | Parse error — invalid JSON |
| 400 | -32600 | Invalid JSON-RPC version |
| 400 | -32601 | Method not found |
| 401 | -32001 | Unauthorized (missing/invalid token) |
| 429 | -32000 | Rate limit exceeded |

---

## Rate limiting on MCP

DB-managed API keys are rate-limited on the MCP path in the same way as the REST
API. The rate limit check runs after authentication and before the tool call.
HTTP 429 is returned with `X-RateLimit-*` headers when the limit is exceeded.

---

## Audit log

Every successful `tools/call` writes a line to `tdb_audit.jsonl`, same format as
the REST query endpoint. Failed calls (auth failures, SQL validation errors) are
logged as warnings only.
