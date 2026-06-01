# Quickstart

This guide walks through registering a PostgreSQL table as a TDB source and running
your first query — REST and MCP. Estimated time: **5 minutes**.

**Prerequisites:**

- TDB is running on `http://localhost:8000` (see [Installation](installation.md))
- You have a PostgreSQL database with at least one table
- `TDB_API_KEYS` is set to a key you know (used as `<YOUR_KEY>` below)

---

## Step 1 — Verify TDB is running

```bash
curl http://localhost:8000/health
```

Expected response:

```json
{"status": "ok"}
```

---

## Step 2 — Register a PostgreSQL source

```bash
curl -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer <YOUR_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "customers",
    "source_type": "postgres",
    "connection": {
      "host": "your-postgres-host",
      "port": 5432,
      "dbname": "your_database",
      "user": "your_user",
      "password": "your_password",
      "table": "customers"
    },
    "description": "Customer records"
  }'
```

Expected response (HTTP 201):

```json
{
  "id": "a1b2c3d4-...",
  "name": "customers",
  "source_type": "postgres",
  "connection": { "host": "...", "port": 5432, "dbname": "...", ... },
  "description": "Customer records",
  "tags": [],
  "registered_by": "<YOUR_KEY>",
  "registered_at": "2026-05-22T09:00:00Z",
  "status": "active"
}
```

Save the `id` — you'll use it in the next step.

!!! tip "Registering multiple sources"
    TDB Enterprise supports multiple simultaneous registered sources. Run the same
    `POST /v1/sources` call with different names and connection details for each table
    or database you want to expose.

---

## Step 3 — Inspect the schema

```bash
curl http://localhost:8000/v1/sources/<SOURCE_ID>/schema \
  -H "Authorization: Bearer <YOUR_KEY>"
```

Expected response:

```json
{
  "source_id": "a1b2c3d4-...",
  "source_name": "customers",
  "columns": [
    {"name": "id", "type": "integer"},
    {"name": "email", "type": "character varying"},
    {"name": "country", "type": "character varying"},
    {"name": "created_at", "type": "timestamp without time zone"}
  ],
  "inspected_at": "2026-05-22T09:00:10Z"
}
```

Schema is introspected live from `information_schema.columns` — always reflects the
current table structure.

---

## Step 4 — Run a SQL query

```bash
curl -X POST http://localhost:8000/v1/query \
  -H "Authorization: Bearer <YOUR_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "<SOURCE_ID>",
    "sql": "SELECT id, email, country FROM customers WHERE country = '\''US'\'' LIMIT 5",
    "limit": 5
  }'
```

Expected response:

```json
{
  "source_id": "a1b2c3d4-...",
  "sql": "SELECT id, email, country FROM customers WHERE country = 'US' LIMIT 5",
  "columns": ["id", "email", "country"],
  "rows": [
    {"id": 1, "email": "alice@example.com", "country": "US"},
    {"id": 2, "email": "bob@example.com", "country": "US"}
  ],
  "rows_returned": 2,
  "truncated": false,
  "executed_at": "2026-05-22T09:00:15Z"
}
```

!!! info "Read-only enforcement"
    TDB rejects any SQL that is not a `SELECT` or `WITH` query at two levels:
    (1) the SQL validator, and (2) the Postgres `read_only = True` transaction flag.
    An `INSERT`, `UPDATE`, or `DELETE` returns HTTP 400 before it reaches the database.

---

## Step 5 — Query via MCP

MCP allows Claude Desktop, Cursor, and other AI tools to call your data source
directly. The MCP endpoint is `POST /v1/mcp` and uses JSON-RPC 2.0.

**Test the MCP handshake (no auth required):**

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}'
```

Expected response:

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

**List available tools (auth required):**

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Authorization: Bearer <YOUR_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```

**Run a query through MCP:**

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Authorization: Bearer <YOUR_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "query_source",
      "arguments": {
        "sql": "SELECT COUNT(*) AS total FROM data",
        "source_name": "customers"
      }
    }
  }'
```

!!! note "Table name in MCP queries"
    Use `data` as the table alias in MCP SQL queries (same as in the REST query
    endpoint). Pass `source_name` to target a specific registered source when you
    have multiple sources registered.

---

## Step 6 — Check the audit log

Every query writes a line to the audit log:

```bash
tail -n 5 tdb_audit.jsonl | python -m json.tool
```

You'll see an entry for each query executed, including timestamp, source ID, SQL,
rows returned, and the API key used.

---

## What's next

- [PostgreSQL connector reference →](../connectors/postgresql.md)
- [Set up JWT authentication →](../auth/jwt.md)
- [Connect Claude Desktop via OAuth →](../auth/oauth.md)
- [Create and rotate API keys →](../auth/api-keys.md)
