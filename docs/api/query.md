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
| `limit` | integer, min 1 | No | `100` | Maximum rows to return. The ceiling is the deployment's `TDB_MAX_ROWS` (default `1,000`); a higher value is rejected with `400`, never silently reduced. |

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

TDB applies the row limit at three levels:

1. **Request-level** — the `limit` field caps rows returned in this response. It may
   not exceed the deployment's `TDB_MAX_ROWS`; a larger value is rejected with `400`.
2. **SQL injection** — if your SQL doesn't contain a `LIMIT` clause, TDB appends
   `LIMIT limit + 1`. If your SQL already has one, it is left alone. The extra row
   is never returned; it exists so TDB can tell a result that exactly fills the
   limit from one that was cut.
3. **After fetching** — the result is cut to `limit` regardless of what the SQL asked
   for, and `truncated` is set to `true` when rows were dropped. This is the ceiling
   that actually holds: a `LIMIT 100000` in your own SQL does **not** raise it.

!!! warning "`truncated: true` means you did not receive the whole result"
    Check the flag before treating a response as complete. Narrow the query — filter,
    aggregate, or page with `LIMIT`/`OFFSET` in your own SQL against a database
    source — rather than assuming the rows you got are all of them.

!!! danger "On 0.7.0 / community 0.4.6 and earlier, `truncated` under-reported"
    Those versions appended `LIMIT limit` rather than `limit + 1`, so a query with
    no `LIMIT` of its own could not distinguish "exactly `limit` rows exist" from
    "the result was cut" — and always reported `truncated: false`. A 5-row table
    queried with `limit: 2` returned 2 rows and `truncated: false`.

    Only queries carrying their own larger `LIMIT` set the flag correctly. **If you
    are on an affected version, do not treat `truncated: false` as proof of
    completeness** — compare `rows_returned` against your `limit` instead, and treat
    equality as "possibly more". Fixed in **0.7.1** and **community 0.4.7**.

**Enterprise deployments can raise the ceiling** by setting `TDB_MAX_ROWS`
([reference](../reference/environment-variables.md)). Every row of a response is held
in memory, so raise it deliberately. Community is fixed at 1,000. Cursor-based
streaming and pagination remain a post-launch feature.

---

## What SQL is accepted

**The statement must begin with `SELECT`**, after leading whitespace and nothing
else. That rule is stricter than "read-only", and the difference catches people
out:

| SQL | Accepted |
|---|---|
| `SELECT * FROM orders` | ✅ |
| `select * from orders` (any case) | ✅ |
| `SELECT * FROM orders -- comment` | ✅ trailing comments are fine |
| `SELECT * FROM (SELECT …) t` | ✅ subqueries |
| `SELECT … UNION SELECT …` | ✅ |
| `SELECT * FROM t WHERE note = 'update pending'` | ✅ keywords inside string literals |
| `WITH x AS (…) SELECT * FROM x` | ❌ **CTEs are not supported** |
| `/* comment */ SELECT 1` | ❌ a *leading* comment |
| `-- comment`<br>`SELECT 1` | ❌ a leading line comment |
| `(SELECT 1)` | ❌ parenthesised |
| `EXPLAIN SELECT 1` | ❌ |
| `SHOW TABLES`, `VALUES (1)`, `TABLE orders` | ❌ |

All rejections return HTTP 400 with
`SQL validation failed: Only SELECT statements are allowed`.

!!! warning "CTEs (`WITH … SELECT`) are rejected"

    A common table expression is read-only and valid SQL, but TDB's validator
    requires the statement to *start* with `SELECT`, so `WITH` is refused. Rewrite
    the CTE as a subquery:

    ```sql
    -- rejected
    WITH recent AS (SELECT * FROM orders WHERE created_at > '2026-01-01')
    SELECT customer, COUNT(*) FROM recent GROUP BY customer

    -- accepted
    SELECT customer, COUNT(*)
    FROM (SELECT * FROM orders WHERE created_at > '2026-01-01') recent
    GROUP BY customer
    ```

### Multiple statements

Semicolon-separated statements are accepted if each one is a `SELECT`, but
**only the last result set is returned** — earlier statements run and their
output is discarded. `SELECT 1 AS a; SELECT 2 AS b` returns `[{"b": 2}]`. Send
one statement per request.

A write keyword anywhere in the string is still rejected, including after a
semicolon: `SELECT 1; DELETE FROM orders` returns 400 with
`Blocked keyword: DELETE`.

---

## Read-only enforcement

TDB rejects SQL containing a write keyword:

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

Every query writes a line to `tdb_audit.jsonl`:

```json
{
  "event": "query",
  "source_id": "a1b2c3d4-...",
  "sql": "SELECT ...",
  "rows_returned": 2,
  "key_hint": "tdbk_a...",
  "ts": "2026-05-22T09:15:00.123456+00:00"
}
```

Rejected queries — SQL validation failures, auth failures, unknown sources — write
an `event: "denied"` entry carrying the `action` and a machine-readable `reason`:

```json
{
  "event": "denied",
  "action": "query",
  "reason": "sql_validation_failed",
  "source_id": "a1b2c3d4-...",
  "sql": "DROP TABLE data",
  "key_hint": "tdbk_a...",
  "ts": "2026-05-22T09:15:02.884101+00:00"
}
```

See [Audit Log](../security/audit.md) for the full `action` and `reason` sets.

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
