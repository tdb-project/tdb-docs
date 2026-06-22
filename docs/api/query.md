# Query API

Run read-only SQL against a registered data source.

---

## Run a query

```
POST /v1/query
```

Requires authentication (`Authorization: Bearer <token>`).

**Request body:**

```json
{
  "source_id": "orders",
  "sql": "SELECT id, status, total FROM orders WHERE status = 'shipped'",
  "limit": 100
}
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `source_id` | string | Yes | — | Registered source **name** (e.g. `orders`) or UUID |
| `sql` | string (1–10,000 chars) | Yes | — | SQL SELECT statement |
| `limit` | integer (1–1,000) | No | `100` | Maximum rows to return |

!!! tip "Use the source name"
    `source_id` accepts either the source's registered **name** (e.g. `orders`) or its
    UUID. Names are easier to read, write, and remember for daily operations.
    UUIDs remain useful in automation scripts and audit-trail lookups.

**Response (200):**

```json
{
  "source_id": "a1b2c3d4-...",
  "sql": "SELECT id, status, total FROM orders WHERE status = 'shipped'",
  "columns": ["id", "status", "total"],
  "rows": [
    {"id": 1001, "status": "shipped", "total": 149.99},
    {"id": 1002, "status": "shipped", "total": 59.00}
  ],
  "rows_returned": 2,
  "truncated": false,
  "executed_at": "2026-05-22T09:15:00Z"
}
```

---

## Table name in queries

The table name to use in your SQL depends on the connector — there is **no universal
`data` alias** across connector types:

- **CSV** (Community): always use `data`. TDB loads the file into an in-memory table
  under that fixed name.
- **PostgreSQL / MySQL / SQL Server / Snowflake**: use the **actual table name** from the
  source's `connection.table`. Your SQL is passed through to the database unchanged (apart
  from an appended `LIMIT`).

```sql
-- CSV source — always 'data'
SELECT COUNT(*) AS total FROM data

-- Database source registered with "table": "orders"
SELECT * FROM orders WHERE status = 'shipped' LIMIT 20
```

---

## Row limit behaviour

TDB applies the row limit at two levels:

1. **Request-level** — the `limit` field caps rows returned in this response (1–1,000).
2. **SQL injection** — if your SQL doesn't contain a `LIMIT` clause, TDB appends one
   automatically. If your SQL already has `LIMIT`, TDB uses it as-is.

Hard cap is 1,000 rows per request. Streaming / pagination beyond 1,000 rows is a
post-launch feature.

---

## Read-only enforcement

TDB rejects SQL that is not a `SELECT` or `WITH` statement:

=== "Bash"

    ```bash
    curl -X POST http://localhost:8000/v1/query \
      -H "Authorization: Bearer <YOUR_KEY>" \
      -H "Content-Type: application/json" \
      -d '{"source_id":"orders","sql":"DELETE FROM orders"}'
    ```

=== "PowerShell"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/query" `
      -Method POST `
      -ContentType "application/json" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" } `
      -Body '{"source_id":"orders","sql":"DELETE FROM orders"}'
    ```

Expected response (HTTP 400):

```json
{"detail": "SQL validation failed: Only SELECT statements are allowed"}
```

Even if the SQL validator is somehow bypassed, the Postgres connection is opened
with `read_only = True` — Postgres itself will reject write operations.

---

## Audit log

Every successful query writes a line to `tdb_audit.jsonl`:

```json
{
  "event": "query",
  "source_id": "a1b2c3d4-...",
  "sql": "SELECT ...",
  "rows_returned": 2,
  "api_key": "tdbk_a1b2...",
  "timestamp": "2026-05-22T09:15:00.123456Z"
}
```

Rejected queries (SQL validation failures, auth failures) are logged as warnings
but do not write a `query` audit event.

---

## Error responses

| Status | Meaning |
|---|---|
| 400 | SQL validation failed (not a SELECT) |
| 401 | Missing or invalid auth token |
| 404 | `source_id` (name or UUID) not found |
| 429 | Rate limit exceeded (DB-managed keys only) |
| 500 | Query execution error — check that the source is reachable |

---

## Examples

**Count rows (by source name):**

=== "Bash"

    ```bash
    curl -X POST http://localhost:8000/v1/query \
      -H "Authorization: Bearer <YOUR_KEY>" \
      -H "Content-Type: application/json" \
      -d '{"source_id": "orders", "sql": "SELECT COUNT(*) AS total FROM orders"}'
    ```

=== "PowerShell"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/query" `
      -Method POST `
      -ContentType "application/json" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" } `
      -Body '{"source_id": "orders", "sql": "SELECT COUNT(*) AS total FROM orders"}'
    ```

Expected response:

```json
{
  "source_id": "a1b2c3d4-...",
  "sql": "SELECT COUNT(*) AS total FROM orders",
  "columns": ["total"],
  "rows": [{"total": 4821}],
  "rows_returned": 1,
  "truncated": false,
  "executed_at": "2026-05-22T09:15:00Z"
}
```

**Aggregate with GROUP BY:**

=== "Bash"

    ```bash
    curl -X POST http://localhost:8000/v1/query \
      -H "Authorization: Bearer <YOUR_KEY>" \
      -H "Content-Type: application/json" \
      -d '{
        "source_id": "orders",
        "sql": "SELECT status, COUNT(*) AS n FROM orders GROUP BY status ORDER BY n DESC",
        "limit": 20
      }'
    ```

=== "PowerShell"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/query" `
      -Method POST `
      -ContentType "application/json" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" } `
      -Body '{
        "source_id": "orders",
        "sql": "SELECT status, COUNT(*) AS n FROM orders GROUP BY status ORDER BY n DESC",
        "limit": 20
      }'
    ```

**Filter with a date range:**

=== "Bash"

    ```bash
    curl -X POST http://localhost:8000/v1/query \
      -H "Authorization: Bearer <YOUR_KEY>" \
      -H "Content-Type: application/json" \
      -d '{
        "source_id": "orders",
        "sql": "SELECT * FROM orders WHERE created_at >= '\''2026-01-01'\'' AND created_at < '\''2026-02-01'\''",
        "limit": 500
      }'
    ```

=== "PowerShell"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/query" `
      -Method POST `
      -ContentType "application/json" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" } `
      -Body '{
        "source_id": "orders",
        "sql": "SELECT * FROM orders WHERE created_at >= ''2026-01-01'' AND created_at < ''2026-02-01''",
        "limit": 500
      }'
    ```
