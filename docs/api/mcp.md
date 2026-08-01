# MCP Endpoint

TDB exposes a [Model Context Protocol](https://modelcontextprotocol.io/) endpoint
at `POST /v1/mcp`. MCP uses JSON-RPC 2.0 over HTTP.

This endpoint is how Claude Desktop, Cursor, and other AI tools query your data
sources directly — without building a custom integration.

!!! tip "Setting up a client"
    This page is the protocol reference. For step-by-step config (Claude Desktop,
    VS Code, Cursor, JetBrains, Windsurf, Cline) and example queries to run once
    connected, see [Connect an IDE / AI Tool →](../getting-started/ide-setup.md).

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

## Available tools

TDB Enterprise exposes **seven tools**:

| Tool | What it does | Works with |
|---|---|---|
| [`query_source`](#toolscall-query_source) | Run a SQL SELECT against a source | All sources, including database-wide |
| [`schema_source`](#toolscall-schema_source) | Column names and types, no SQL required | All sources — lists every table for database-wide sources |
| [`preview_source`](#toolscall-preview_source) | First N rows, no SQL required | All sources — `table` argument required for database-wide |
| [`filter_source`](#toolscall-filter_source) | Rows matching one column condition | All sources — `table` argument required for database-wide |
| [`aggregate_source`](#toolscall-aggregate_source) | COUNT / SUM / AVG / MIN / MAX with optional GROUP BY | All sources — `table` argument required for database-wide |
| [`list_views`](#toolscall-list_views-and-run_view) | List the available [YAML views](views.md) | Requires `TDB_VIEWS_DIR` |
| [`run_view`](#toolscall-list_views-and-run_view) | Execute a named view with typed parameters | Requires `TDB_VIEWS_DIR` |

!!! note "Database-wide sources and the no-SQL tools"
    For a **database-wide** source (registered without `table`), the four no-SQL
    convenience tools (`schema_source`, `preview_source`, `filter_source`,
    `aggregate_source`) accept an optional `table` argument to pick which table
    to target, validated against the source's actual tables:

    - `schema_source` — omit `table` to list every table's columns; pass it to
      see one table's columns.
    - `preview_source`, `filter_source`, `aggregate_source` — `table` is
      **required**. Omitting it returns a tool-level error listing the
      available tables, so the AI tool can retry with a valid one.

    Column/value validation and the generated SQL are scoped to whichever
    table you name. `query_source` remains the only tool that can JOIN across
    tables in a single call.

Tools that return rows pass their output through the
[prompt-injection filter](#prompt-injection-filtering) before the result reaches
the MCP client. API keys can be restricted to a subset of tools — see
[Tool allow-lists per key](#tool-allow-lists-per-api-key).

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
    "serverInfo": {"name": "tdb-enterprise", "version": "0.2.2"}
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

Response (abbreviated — one entry per tool, seven in total):

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      { "name": "query_source",     "description": "...", "inputSchema": { ... } },
      { "name": "schema_source",    "description": "...", "inputSchema": { ... } },
      { "name": "preview_source",   "description": "...", "inputSchema": { ... } },
      { "name": "filter_source",    "description": "...", "inputSchema": { ... } },
      { "name": "aggregate_source", "description": "...", "inputSchema": { ... } },
      { "name": "list_views",       "description": "...", "inputSchema": { ... } },
      { "name": "run_view",         "description": "...", "inputSchema": { ... } }
    ]
  }
}
```

`tools/list` always advertises all seven tools. A key with a
[tool allow-list](#tool-allow-lists-per-api-key) still sees the full list but is
rejected at call time for tools outside its allow-list.

---

### `tools/call` — `query_source`

Executes a SQL query against a registered source. Maximum 1,000 rows returned.

| Argument | Type | Required | Default | Description |
|---|---|---|---|---|
| `sql` | string | Yes | — | SQL SELECT statement |
| `source_name` | string | No | First registered source | Registered source name (exact match) |

Use the table name that matches the source: CSV sources are queried as `data`;
database sources use **real table names**. Database-wide sources can JOIN across
any tables in the database.

!!! tip "Always pass `source_name` when more than one source is registered"
    Without it, the query runs against the **first** registered source, which may
    not be the one the AI tool intended.

**Single-table source:**

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
        "sql": "SELECT country, COUNT(*) AS n FROM customers GROUP BY country ORDER BY n DESC LIMIT 10",
        "source_name": "customers"
      }
    }
  }'
```

**Database-wide source (cross-table JOIN):**

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
        "sql": "SELECT c.name, SUM(o.total) AS spend FROM customers c JOIN orders o ON o.customer_id = c.id GROUP BY c.name ORDER BY spend DESC LIMIT 10",
        "source_name": "production_db"
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

### `tools/call` — `schema_source`

Returns column names and data types for a source — the tool an AI assistant calls
before writing a query. No SQL required.

| Argument | Type | Required | Default | Description |
|---|---|---|---|---|
| `source_name` | string | No | First registered source | Registered source name (exact match) |
| `table` | string | No | — | Table to inspect. Ignored for single-table/CSV sources. For **database-wide** sources: omit to list every table, or name one to see just its columns. |

**Single-table or CSV source:**

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 5,
    "method": "tools/call",
    "params": {"name": "schema_source", "arguments": {"source_name": "customers"}}
  }'
```

Result payload (inside `content[0].text`):

```json
{
  "source": "customers",
  "columns": [
    {"name": "id", "type": "integer"},
    {"name": "country", "type": "text"}
  ]
}
```

**Database-wide source, no `table` (lists every table):**

```json
{
  "source": "production_db",
  "tables": {
    "customers": [{"name": "id", "type": "integer"}, {"name": "country", "type": "text"}],
    "orders": [{"name": "id", "type": "integer"}, {"name": "customer_id", "type": "integer"}]
  }
}
```

**Database-wide source with `table: "orders"`:**

```json
{
  "source": "production_db",
  "table": "orders",
  "columns": [
    {"name": "id", "type": "integer"},
    {"name": "customer_id", "type": "integer"}
  ]
}
```

An unrecognized `table` returns a tool error listing the actual table names.

Schema results are served from the [schema cache](../reference/environment-variables.md)
when caching is enabled.

---

### `tools/call` — `preview_source`

Returns the first N rows of a source's table. No SQL required.

| Argument | Type | Required | Default | Description |
|---|---|---|---|---|
| `source_name` | string | No | First registered source | Registered source name (exact match) |
| `table` | string | No | — | Table to preview. **Required for database-wide sources** (registered without a fixed table) — ignored for single-table/CSV sources. |
| `limit` | integer | No | 10 | Rows to return (1–100) |

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 6,
    "method": "tools/call",
    "params": {"name": "preview_source", "arguments": {"source_name": "customers", "limit": 5}}
  }'
```

**Database-wide source** (`table` required):

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 6,
    "method": "tools/call",
    "params": {"name": "preview_source", "arguments": {"source_name": "production_db", "table": "orders", "limit": 5}}
  }'
```

Result payload: `{"source": ..., "columns": [...], "rows": [...], "rows_returned": N}`.
Omitting `table` on a database-wide source returns a tool error listing the
available tables.

---

### `tools/call` — `filter_source`

Returns rows matching a single column condition, without the AI tool writing SQL.
The column name is validated against the source schema and the operator against a
fixed allow-list, so the model cannot inject arbitrary SQL through this tool.

| Argument | Type | Required | Default | Description |
|---|---|---|---|---|
| `column` | string | Yes | — | Column to filter on (must exist in the schema) |
| `value` | string | Yes | — | Comparison value (interpreted by column type) |
| `operator` | string | No | `=` | One of `=`, `!=`, `>`, `<`, `>=`, `<=`, `LIKE` |
| `source_name` | string | No | First registered source | Registered source name (exact match) |
| `table` | string | No | — | Table to filter. **Required for database-wide sources** — ignored for single-table/CSV sources. `column` is validated against this table's schema. |
| `limit` | integer | No | 100 | Max rows to return (1–1,000) |

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 7,
    "method": "tools/call",
    "params": {
      "name": "filter_source",
      "arguments": {"source_name": "customers", "column": "country", "value": "US", "limit": 50}
    }
  }'
```

Result payload: `{"source": ..., "columns": [...], "rows": [...], "rows_returned": N}`.
An unknown column returns a tool error listing the valid column names.

---

### `tools/call` — `aggregate_source`

Runs a single aggregate over a column, optionally grouped. Column names are
validated against the source schema; the function is restricted to the five
listed below.

| Argument | Type | Required | Default | Description |
|---|---|---|---|---|
| `function` | string | Yes | — | One of `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` |
| `column` | string | Yes | — | Column to aggregate; `*` is allowed for `COUNT(*)` |
| `group_by` | string | No | — | Column to group results by |
| `source_name` | string | No | First registered source | Registered source name (exact match) |
| `table` | string | No | — | Table to aggregate. **Required for database-wide sources** — ignored for single-table/CSV sources. `column`/`group_by` are validated against this table's schema. |
| `limit` | integer | No | 100 | Max groups to return (1–1,000) |

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 8,
    "method": "tools/call",
    "params": {
      "name": "aggregate_source",
      "arguments": {"source_name": "customers", "function": "COUNT", "column": "*", "group_by": "country"}
    }
  }'
```

Result payload: `{"source": ..., "function": "COUNT", "column": "*", "group_by": "country",
"columns": [...], "rows": [...], "rows_returned": N}`.

---

### `tools/call` — `list_views` and `run_view`

Expose [YAML named views](views.md) to AI tools. Views are administrator-approved,
pre-defined queries — the safest way to give an AI assistant multi-table access,
because the SQL is fixed and only typed parameters vary. Both tools require views
to be configured (`TDB_VIEWS_DIR`); with no views loaded, `list_views` returns an
empty list.

**`list_views`** takes no arguments and returns every view with its description,
source, and parameter definitions.

**`run_view`:**

| Argument | Type | Required | Default | Description |
|---|---|---|---|---|
| `view_name` | string | Yes | — | Name of the view to execute |
| `parameters` | object | No | `{}` | Parameter values required by the view |
| `limit` | integer | No | 1,000 | Max rows to return (1–1,000) |

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 9,
    "method": "tools/call",
    "params": {
      "name": "run_view",
      "arguments": {"view_name": "daily_signups", "parameters": {"country": "US"}}
    }
  }'
```

Result payload: `{"view": ..., "source": ..., "columns": [...], "rows": [...], "rows_returned": N}`.

---

## Tool allow-lists per API key

DB-managed API keys can carry an `allowed_tools` list that restricts which MCP
tools the key may call — for example, a reporting agent's key limited to
`["schema_source", "run_view", "list_views"]`. Keys without an allow-list can call
every tool.

A disallowed call returns a tool-level error (HTTP 200, `isError: true`) so MCP
clients handle it gracefully:

```json
{
  "content": [{"type": "text", "text": "Tool 'query_source' is not permitted for this API key."}],
  "isError": true
}
```

See [API Keys → tool allow-lists](../auth/api-keys.md) for how to set
`allowed_tools` when creating or updating a key.

---

## Prompt-injection filtering

Two filters protect the MCP path:

- **Input:** the `sql` argument of `query_source` is screened before validation.
  A flagged input is rejected with a tool error
  (`Input rejected: potential prompt injection detected.`) and logged.
- **Output:** rows returned by any tool are screened before the response is
  serialised. Cells that contain injection patterns (e.g. instructions embedded in
  data that try to steer the AI model) are redacted, and the redaction count is
  written to the server log.

The filters run server-side on every call — there is nothing to configure on the
MCP client.

---

## Error codes

| HTTP status | JSON-RPC error code | Meaning |
|---|---|---|
| 200 | — | Success (check `isError` for tool-level errors) |
| 200 | -32700 | Parse error — invalid JSON |
| 200 | -32600 | Invalid JSON-RPC version |
| 200 | -32601 | Method not found / unknown tool |
| 401 | -32001 | Unauthorized (missing/invalid token) |
| 429 | -32000 | Rate limit exceeded |

Protocol-level errors (parse, version, unknown method) follow JSON-RPC-over-HTTP
convention and are returned with HTTP 200 and an `error` object; only auth and
rate-limit failures use HTTP status codes, so MCP clients can react to them at the
transport layer.

---

## Rate limiting on MCP

DB-managed API keys are rate-limited on the MCP path in the same way as the REST
API. The rate limit check runs after authentication and before the tool call.
HTTP 429 is returned with `X-RateLimit-*` headers when the limit is exceeded.

---

## Audit log

Every successful `tools/call` that touches data (`query_source`, `preview_source`,
`filter_source`, `aggregate_source`, `run_view`) writes a line to `tdb_audit.jsonl`,
same format as the REST query endpoint — including the SQL that the no-SQL tools
generated on the caller's behalf (`run_view` entries record `<view:name>`). Failed
calls (auth failures, SQL validation errors) are logged as warnings only.
