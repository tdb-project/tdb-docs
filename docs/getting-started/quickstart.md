# Quickstart

!!! info "TDB Enterprise"
    This guide covers the **commercial enterprise edition**. For the free,
    open-source CSV edition, see the [Community Edition quickstart](community.md).

This guide walks through registering a PostgreSQL database as a TDB source and running
your first query — REST and MCP. Estimated time: **5 minutes**.

**Prerequisites:**

- TDB is running on `http://localhost:8000` (see [Installation](installation.md))
- You have a PostgreSQL database with at least one table
- You know your Bearer token — used as `<YOUR_KEY>` throughout this guide

!!! tip "Finding your Bearer token"
    Your token is the value you passed to `TDB_API_KEYS` when starting the container.
    If you started TDB via Docker Desktop without setting that variable, the fallback
    token is `dev-insecure-key-change-me` — use that to complete this guide, then
    replace it with a real key before going to production.

!!! tip "Windows users — use PowerShell 7+"
    The **PowerShell** tabs below use `Invoke-RestMethod`, which is built into
    PowerShell 7 and later. If you are on WSL or a Linux/macOS terminal, use the
    **Bash** tabs instead. Both sets of commands produce identical results.

---

## Step 1 — Verify TDB is running

**Why:** Confirm the server started correctly before making any data calls. A healthy
response here means the API is ready.

=== "Bash"

    ```bash
    curl http://localhost:8000/health
    ```

=== "PowerShell"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/health"
    ```

Expected response:

```json
{"status": "ok"}
```

---

## Step 2 — Register a PostgreSQL source

**Why:** TDB needs to know about your database before it can query it. You register
the connection once — after that you refer to the source by its friendly name
(`production_db` in this example) instead of typing connection details every time.

**Database-wide registration (recommended):** Omit `table` to register your entire
database as a single source. You can then query any table — or write JOINs across
tables — using standard SQL. One registration covers all tables.

=== "Bash"

    ```bash
    curl -X POST http://localhost:8000/v1/sources \
      -H "Authorization: Bearer <YOUR_KEY>" \
      -H "Content-Type: application/json" \
      -d '{
        "name": "production_db",
        "source_type": "postgres",
        "connection": {
          "host": "host.docker.internal",
          "port": 5432,
          "dbname": "your_database",
          "user": "your_user",
          "password": "your_password"
        },
        "description": "Production Postgres — all tables"
      }'
    ```

=== "PowerShell"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/sources" `
      -Method POST `
      -ContentType "application/json" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" } `
      -Body '{
        "name": "production_db",
        "source_type": "postgres",
        "connection": {
          "host": "host.docker.internal",
          "port": 5432,
          "dbname": "your_database",
          "user": "your_user",
          "password": "your_password"
        },
        "description": "Production Postgres - all tables"
      }'
    ```

Expected response (HTTP 201):

```json
{
  "id": "a1b2c3d4-...",
  "name": "production_db",
  "source_type": "postgres",
  "connection": { "host": "host.docker.internal", "port": 5432, "dbname": "your_database" },
  "description": "Production Postgres — all tables",
  "tags": [],
  "registered_by": "<YOUR_KEY>",
  "registered_at": "2026-05-22T09:00:00Z",
  "status": "active"
}
```

!!! note "host.docker.internal"
    Use `host.docker.internal` as the host when your PostgreSQL instance runs on the
    Windows or Mac host machine (or in WSL). Inside Docker, `localhost` resolves to the
    container itself — `host.docker.internal` always routes back to the host.

!!! tip "Single-table registration"
    To scope a source to one specific table, add `"table": "customers"` to the
    `connection` object. The schema endpoint will then return only that table's columns.
    This can be useful when you want to grant different API keys access to different
    tables as separate named sources.

---

## Step 3 — Verify registered sources

**Why:** Before inspecting schema or running queries, confirm the source registered
successfully and find its exact name. The name shown here is what you'll use in all
subsequent commands — no UUID needed.

=== "Bash"

    ```bash
    curl http://localhost:8000/v1/sources \
      -H "Authorization: Bearer <YOUR_KEY>"
    ```

=== "PowerShell"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/sources" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" }
    ```

Expected response:

```json
[
  {
    "id": "a1b2c3d4-...",
    "name": "production_db",
    "source_type": "postgres",
    "description": "Production Postgres — all tables",
    "tags": [],
    "registered_at": "2026-05-22T09:00:00Z"
  }
]
```

You should see your source in the list. If the list is empty, the registration in
Step 2 did not succeed — re-check your connection details and the
`host.docker.internal` note above.

---

## Step 4 — Inspect the schema

**Why:** See what tables and columns are available before writing your first query.
This avoids trial-and-error SQL errors caused by misremembered table or column names.
AI agents also call this endpoint to understand the shape of your data before generating SQL.
You can use the source **name** (`production_db`) instead of the UUID.

=== "Bash"

    ```bash
    curl http://localhost:8000/v1/sources/production_db/schema \
      -H "Authorization: Bearer <YOUR_KEY>"
    ```

=== "PowerShell"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/sources/production_db/schema" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" }
    ```

Expected response (database-wide source — all tables in the `public` schema):

```json
{
  "source_id": "a1b2c3d4-...",
  "source_name": "production_db",
  "columns": [],
  "tables": [
    {
      "name": "customers",
      "columns": [
        {"name": "id", "type": "integer"},
        {"name": "email", "type": "character varying"},
        {"name": "country", "type": "character varying"},
        {"name": "created_at", "type": "timestamp without time zone"}
      ]
    },
    {
      "name": "orders",
      "columns": [
        {"name": "id", "type": "integer"},
        {"name": "customer_id", "type": "integer"},
        {"name": "total", "type": "numeric"},
        {"name": "status", "type": "character varying"}
      ]
    }
  ],
  "inspected_at": "2026-05-22T09:00:10Z"
}
```

Schema is introspected live from `information_schema.columns` — always reflects the
current database structure.

---

## Step 5 — Run a SQL query

**Why:** Verify that TDB can retrieve actual data from your source end-to-end.
Use the source **name** in `source_id`. With a database-wide source, you can query
any table — or JOIN across tables — using real table names in your SQL.

=== "Bash (single table)"

    ```bash
    curl -X POST http://localhost:8000/v1/query \
      -H "Authorization: Bearer <YOUR_KEY>" \
      -H "Content-Type: application/json" \
      -d '{
        "source_id": "production_db",
        "sql": "SELECT id, email, country FROM customers WHERE country = '\''US'\'' LIMIT 5"
      }'
    ```

=== "Bash (JOIN across tables)"

    ```bash
    curl -X POST http://localhost:8000/v1/query \
      -H "Authorization: Bearer <YOUR_KEY>" \
      -H "Content-Type: application/json" \
      -d '{
        "source_id": "production_db",
        "sql": "SELECT c.email, o.total, o.status FROM customers c JOIN orders o ON o.customer_id = c.id WHERE o.status = '\''pending'\'' LIMIT 10"
      }'
    ```

=== "PowerShell (single table)"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/query" `
      -Method POST `
      -ContentType "application/json" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" } `
      -Body '{
        "source_id": "production_db",
        "sql": "SELECT id, email, country FROM customers WHERE country = ''US'' LIMIT 5"
      }'
    ```

=== "PowerShell (JOIN across tables)"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/query" `
      -Method POST `
      -ContentType "application/json" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" } `
      -Body '{
        "source_id": "production_db",
        "sql": "SELECT c.email, o.total, o.status FROM customers c JOIN orders o ON o.customer_id = c.id WHERE o.status = ''pending'' LIMIT 10"
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

## Step 6 — Query via MCP

**Why:** MCP is the protocol Claude Desktop, Cursor, and other AI tools use to call
your data source directly. Testing these three sub-steps confirms the full AI-to-data
path works before you connect a real AI client.

### 6a — Test the MCP handshake

**Why:** The `initialize` call confirms TDB's MCP server is up and returns the
protocol version the server speaks. No authentication is required for this step.

=== "Bash"

    ```bash
    curl -X POST http://localhost:8000/v1/mcp \
      -H "Content-Type: application/json" \
      -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}'
    ```

=== "PowerShell"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/mcp" `
      -Method POST `
      -ContentType "application/json" `
      -Body '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}'
    ```

Expected response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {"tools": {}},
    "serverInfo": {"name": "tdb-enterprise", "version": "0.3.0"}
  }
}
```

### 6b — List available MCP tools

**Why:** Confirms your API key is valid on the MCP endpoint and shows every tool
an AI agent can call. You should see one tool per registered source.

=== "Bash"

    ```bash
    curl -X POST http://localhost:8000/v1/mcp \
      -H "Authorization: Bearer <YOUR_KEY>" \
      -H "Content-Type: application/json" \
      -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
    ```

=== "PowerShell"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/mcp" `
      -Method POST `
      -ContentType "application/json" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" } `
      -Body '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
    ```

Expected response (one entry per registered source):

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "query_customers",
        "description": "Run a read-only SQL SELECT against the 'customers' source",
        "inputSchema": {
          "type": "object",
          "properties": {
            "sql": {"type": "string"}
          },
          "required": ["sql"]
        }
      }
    ]
  }
}
```

### 6c — Run a query through MCP

**Why:** This is the exact call an AI agent makes when it queries your data. A
successful response here means Claude Desktop or Cursor can use this source.

=== "Bash"

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
            "sql": "SELECT COUNT(*) AS total FROM customers",
            "source_name": "customers"
          }
        }
      }'
    ```

=== "PowerShell"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/mcp" `
      -Method POST `
      -ContentType "application/json" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" } `
      -Body '{
        "jsonrpc": "2.0",
        "id": 3,
        "method": "tools/call",
        "params": {
          "name": "query_source",
          "arguments": {
            "sql": "SELECT COUNT(*) AS total FROM customers",
            "source_name": "customers"
          }
        }
      }'
    ```

Expected response:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"columns\":[\"total\"],\"rows\":[{\"total\":1423}],\"rows_returned\":1,\"truncated\":false}"
      }
    ]
  }
}
```

!!! note "Table name in MCP queries"
    Use the **registered table name** in your SQL — the same name you use at the REST
    query endpoint (here, `customers`). SQL is passed straight through to the database, so
    there is no `data` alias for database sources (that's CSV-only). Pass `source_name` to
    target a specific registered source when you have multiple registered.

---

## Step 7 — Check the audit log

**Why:** TDB's audit log is the compliance centrepiece — every query is recorded
with timestamp, source, SQL text, row count, and the API key used. Checking it now
confirms the log is working before you go to production.

Every query writes a line to the audit log. The log lives **inside the container**
at `/app/tdb_audit.jsonl` (configurable via `TDB_LOG_FILE`).

=== "Bash"

    ```bash
    docker exec tdb tail -n 5 /app/tdb_audit.jsonl | python -m json.tool
    ```

=== "PowerShell"

    ```powershell
    docker exec tdb tail -n 5 /app/tdb_audit.jsonl | python -m json.tool
    ```

Expected output (one JSON object per query):

```json
{
  "ts": "2026-05-22T09:00:15.123456+00:00",
  "event": "query",
  "source_id": "a1b2c3d4-...",
  "sql": "SELECT id, email, country FROM customers WHERE country = 'US' LIMIT 5",
  "rows_returned": 2,
  "key_hint": "tdbk_a..."
}
```

Attempts that were refused appear as `"event": "denied"` lines carrying an
`action` and a `reason` instead — see [Audit Log](../security/audit.md).

If you see entries for the queries you ran in Steps 5 and 6c, the audit trail is
working correctly.

!!! tip "Persist the audit log on the host"
    Mount a volume (e.g. `-v "$(pwd)/logs:/app/logs"` and set
    `TDB_LOG_FILE=/app/logs/tdb_audit.jsonl`) so the audit trail survives
    container restarts and can be tailed directly from the host.

---

## What's next

- [PostgreSQL connector reference →](../connectors/postgresql.md)
- [Set up JWT authentication →](../auth/jwt.md)
- [Connect Claude Desktop via OAuth →](../auth/oauth.md)
- [Create and rotate API keys →](../auth/api-keys.md)
- [Connect VS Code, Cursor, JetBrains, and other IDEs →](ide-setup.md)
