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
  "source_id": "a1b2c3d4-e5f6-...",
  "sql": "SELECT id, email, country FROM data WHERE country = 'US'",
  "limit": 100
}
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `source_id` | string (UUID) | Yes | — | ID returned when the source was registered |
| `sql` | string (1–10,000 chars) | Yes | — | SQL SELECT statement |
| `limit` | integer (1–1,000) | No | `100` | Maximum rows to return |

**Response (200):**

```json
{
  "source_id": "a1b2c3d4-...",
  "sql": "SELECT id, email, country FROM data WHERE country = 'US'",
  "columns": ["id", "email", "country"],
  "rows": [
    {"id": 1, "email": "alice@example.com", "country": "US"},
    {"id": 2, "email": "bob@example.com", "country": "US"}
  ],
  "rows_returned": 2,
  "truncated": false,
  "executed_at": "2026-05-22T09:15:00Z"
}
```

---

## Table name in queries

For the PostgreSQL connector, write your SQL against the **registered table name**,
or use `data` as a universal alias — TDB resolves it to the correct table.

```sql
-- Using the actual table name
SELECT * FROM orders WHERE status = 'shipped' LIMIT 20

-- Using the 'data' alias (works for all connector types)
SELECT COUNT(*) AS total FROM data
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

```bash
curl -X POST http://localhost:8000/v1/query \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{"source_id":"...","sql":"DELETE FROM orders"}'

# HTTP 400
# {"detail":"SQL validation failed: Only SELECT statements are allowed"}
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
| 404 | `source_id` not found |
| 429 | Rate limit exceeded (DB-managed keys only) |
| 500 | Query execution error — check that the source is reachable |

---

## Examples

**Count rows:**

```bash
curl -X POST http://localhost:8000/v1/query \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{"source_id":"<ID>","sql":"SELECT COUNT(*) AS total FROM data"}'
```

**Aggregate with GROUP BY:**

```bash
curl -X POST http://localhost:8000/v1/query \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "<ID>",
    "sql": "SELECT country, COUNT(*) AS n FROM data GROUP BY country ORDER BY n DESC",
    "limit": 20
  }'
```

**Filter with a date range:**

```bash
curl -X POST http://localhost:8000/v1/query \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "<ID>",
    "sql": "SELECT * FROM data WHERE created_at >= '\''2026-01-01'\'' AND created_at < '\''2026-02-01'\''",
    "limit": 500
  }'
```
