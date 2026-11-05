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
| `page_key` | array of strings | No | — | Enterprise, from 0.31.0. Column or columns that uniquely order the result, to [read it page by page](#reading-more-than-one-page). |
| `cursor` | string | No | — | Enterprise, from 0.31.0. The `next_cursor` from the last page. |

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

## How values are returned

Rows come back as JSON, in the same form on REST (`rows`) and MCP (the tool text and
`structuredContent`), for every source type:

| Source type | Returned as | Example |
|---|---|---|
| Integers, floats, booleans, text | JSON number, boolean or string | `42`, `1.5`, `true`, `"EU"` |
| `NULL` | `null` | `null` |
| `NaN`, `Infinity`, `-Infinity` | String | `"NaN"`, `"Infinity"`, `"-Infinity"` |
| `DECIMAL` / `NUMERIC` | String with the exact digits | `"149.99"` |
| `DATE`, `TIME` | ISO 8601 string | `"2026-01-01"`, `"10:00:00"` |
| `TIMESTAMP` | ISO 8601 string | `"2026-01-01T10:00:00"`; with a time zone, `"2026-01-01T10:00:00Z"` in UTC or `"…+05:30"` |
| `INTERVAL` | ISO 8601 duration | `"P1D"` |
| `UUID` | String | `"4b48af8e-b7dd-45e2-836e-e8efa86e41c2"` |
| Binary (`BLOB`, `bytea`, `varbinary`, `BINARY`) | Base64 string | `"AP8="` for bytes `00 ff` |
| Arrays, `LIST`, `STRUCT` | JSON array or object, with these rules applied inside | `["AP8="]`, `{"k": 1}` |

`DECIMAL` is a string so that `0.10` keeps its exact digits instead of becoming a
binary floating-point number. `NaN` and infinities are strings because JSON has no
such numbers; they are spelled the way PostgreSQL's `to_json` spells them, and stay
distinct from `null`.

!!! note "Changed in community 0.12.2 / enterprise 0.20.1"
    Before these releases a binary value made `POST /v1/query` fail with `500` after
    the query had run; `NaN` and infinities were `null` on REST and invalid bare
    tokens on MCP; and MCP wrote timestamps with a space (`2026-01-01 10:00:00`) and
    intervals as `1 day, 0:00:00`.

---

## Table name in queries

The table name to use in your SQL depends on the connector — there is **no universal
`data` alias** across connector types:

- **CSV** (Community), and **Parquet** (Enterprise): always use `data`. TDB reads the
  file in place under that fixed name.
- **PostgreSQL / MySQL / SQL Server / Snowflake / DuckDB**: use the **actual table name** from the
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
    Check the flag before treating a response as complete. Narrow the query, or in
    enterprise from 0.31.0 [read on with `page_key`](#reading-more-than-one-page) or
    [stream the result](#streaming-a-whole-result). From 0.31.0 a cut enterprise
    response also carries a `hint` saying so.

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
in memory, so raise it deliberately. Community is fixed at 1,000, with no paging or
streaming.

---

## Reading more than one page

!!! info "Enterprise, from 0.31.0"

Name the column or columns that uniquely order the result in `page_key`. TDB wraps your
query, orders it by them and continues after the last row, so a row deleted between
pages cannot make another row go missing, as it can with `OFFSET`.

```json
{
  "source_id": "orders",
  "sql": "SELECT id, status, total FROM orders WHERE status = 'shipped'",
  "page_key": ["id"],
  "limit": 500
}
```

The response is the usual one, plus `next_cursor` when there is more:

```json
{
  "columns": ["id", "status", "total"],
  "rows": ["..."],
  "rows_returned": 500,
  "truncated": true,
  "next_cursor": "gAAAAAB..."
}
```

Send the same `sql` and `page_key` with `cursor` set to it for the next page. With no
`next_cursor`, the result has ended. Page keys work on every source type, though
[Snowflake has not run against a real Snowflake](../connectors/snowflake.md).

**A key that could lose or repeat a row is refused, never guessed at:**

| Reason | Status | The key |
|---|---|---|
| `page_key_has_nulls` | 400 | holds a NULL, and those rows would be lost. Choose another key, or filter them out |
| `page_key_not_unique` | 400 | ties across a page boundary. Add a column that makes it unique, such as the primary key. Ties inside a page are harmless, so this can appear pages into a result |
| `page_key_type` | 400 | is a float, JSON or binary value, or a timestamp finer than a microsecond: its value does not survive the trip back to the database exactly |
| `page_key_invalid` | 400 | names no single column of the result, or is more than 8 columns |
| `page_key_too_long` | 400 | holds values too long for a cursor at this boundary |
| `page_key_required` | 400 | is missing: a cursor needs the `page_key` it began with |

**A cursor is bound to who asked and what they asked.** It holds values from the last
row, so it is encrypted, and TDB stores nothing on the server. It lasts `TDB_CURSOR_TTL`
(one hour), and rotating the credential key ends every open cursor.

| Reason | Status | What happened |
|---|---|---|
| `cursor_invalid` | 400 | The cursor is damaged or was not issued by this deployment |
| `cursor_wrong_caller` | 403 | Another caller was issued it. Rotating an API key counts: start again |
| `cursor_expired` | 409 | It is older than `TDB_CURSOR_TTL`. Start again |
| `cursor_stale` | 409 | The SQL, the `page_key` or the policy file has changed since. Start again |
| `max_result_rows` | 409 | The chain has already reached the cap below |

**One result is capped at `TDB_MAX_RESULT_ROWS`** (100,000) across all its pages, and
`TDB_MAX_ROWS` still bounds each page. The page that reaches the cap has
`truncated: true`, a `hint` and no `next_cursor`. Each page is one audit entry and counts
once against the rate limit, and a [policy](../security/policies.md) applies to every
page: a withheld key column is refused, and a masked one pages on its mask.

**Every page runs the query again.** A row present throughout is never lost or repeated.
A row inserted behind the boundary is not seen; a row whose key changes can move past
it or appear twice. If your SQL has its own `LIMIT` with no `ORDER BY`, the engine can
choose different rows on each page, so give it an `ORDER BY` or drop the `LIMIT`.

---

## Streaming a whole result

!!! info "Enterprise, from 0.31.0"

`POST /v1/query/stream` sends a whole result in one read session, past `TDB_MAX_ROWS`,
at flat memory. It takes `source_id`, `sql` and `format` (`ndjson`, the default, or
`csv`), and refuses a `limit`. A key needs the `query_source` tool.

```bash
curl -N -X POST http://localhost:8000/v1/query/stream \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{"source_id": "orders", "sql": "SELECT id, total FROM orders"}'
```

```
{"id":1,"total":"149.99"}
{"id":2,"total":"59.00"}
{"_end":true,"rows":2,"complete":true,"columns":["id","total"]}
```

!!! danger "Without the `_end` line, the result is not complete"
    A failure after the first row can no longer be a status code. On NDJSON TDB sends
    `{"_error": {"reason": "..."}, "rows": N, "complete": false}` and drops the
    connection. **CSV has no end line, so a dropped connection is the only sign**, and
    your client must treat it as a failure. Over HTTP/1.0, or through a proxy that
    buffers and closes for you, a cut CSV can look complete. Use NDJSON where it matters.

Everything that can be refused is refused before the first row, with the status codes on
[Errors](../reference/errors.md). A later failure is one of: `max_result_rows` (the result
reached `TDB_MAX_RESULT_ROWS`), `stream_timeout` (it ran past `TDB_STREAM_TIMEOUT`), or
a source failure such as `query_failed`.

- **NDJSON needs unique column names**, and none called `_end` or `_error`: refused with
  400 `stream_column_names`. Name them with `AS`, or use CSV.
- **A stream holds its source open.** On PostgreSQL an open read session stops vacuum on
  that database, which is why the timeout cannot be turned off. On a
  [DuckDB source](../connectors/duckdb.md) the file's owner cannot write until it ends.
  A client that stops reading holds the session until the timeout. Streams use their own
  connection, never the pool.
- **`TDB_MAX_STREAMS`** (4) bounds streams per process; past it, 429 `too_many_streams`.
  The slots are shared, so one key can hold them all: keep streaming keys to callers you
  trust, with [`allowed_tools`](../auth/api-keys.md).
- **Opening a CSV in a spreadsheet:** a cell starting with `=`, `+`, `-` or `@` can run as a
  formula. TDB writes values as they are, so treat a stream of untrusted data accordingly.
- **Audit:** `query_started` before the query runs, refusing if it cannot be written, then
  `query` with `rows_returned` and `stream_outcome`. A stream counts once against the
  rate limit. [Audit log](../security/audit.md).

---

## What SQL is accepted

**The statement must begin with `SELECT` or `WITH`**, ignoring leading
whitespace and leading comments. That rule is stricter than "read-only", and the
difference catches people out:

| SQL | Accepted |
|---|---|
| `SELECT * FROM orders` | ✅ |
| `select * from orders` (any case) | ✅ |
| `SELECT * FROM orders -- comment` | ✅ trailing comments are fine |
| `SELECT * FROM (SELECT …) t` | ✅ subqueries |
| `SELECT … UNION SELECT …` | ✅ |
| `SELECT * FROM t WHERE note = 'update pending'` | ✅ keywords inside string literals |
| `SELECT replace(name, 'a', 'b') FROM t` | ✅ the `replace()` string function, **from 0.8.0 / 0.13.0** (earlier: `Blocked keyword: replace`) |
| `REPLACE INTO t VALUES (…)` | ❌ MySQL's `REPLACE` statement, in any form |
| `SELECT has_table_privilege(t, 'UPDATE') AS update` | ✅ a write keyword as an alias, **from 0.9.0 / 0.14.0** |
| `WITH x AS (…) SELECT * FROM x` | ✅ CTEs, **from 0.6.0 / 0.10.0** |
| `WITH RECURSIVE t(n) AS (…) SELECT * FROM t` | ✅ recursive CTEs |
| `/* comment */ SELECT 1` | ✅ a leading comment, **from 0.6.0 / 0.10.0** |
| `-- comment`<br>`SELECT 1` | ✅ a leading line comment |
| `WITH x AS (DELETE FROM t RETURNING *) SELECT * FROM x` | ❌ `Blocked keyword: DELETE` |
| `(SELECT 1)` | ❌ parenthesised |
| `EXPLAIN SELECT 1` | ❌ |
| `SHOW TABLES`, `VALUES (1)`, `TABLE orders` | ❌ |

Rejections return HTTP 400. A statement that opens with the wrong token gives
`SQL validation failed: Only SELECT and WITH statements are allowed`; one
containing a write keyword gives `Blocked keyword: <keyword>` instead, and that
check runs first.

!!! info "CTEs are accepted from community 0.6.0 / enterprise 0.10.0"

    Earlier versions refused `WITH … SELECT` — the validator required the
    statement to *start* with `SELECT`. If you are on an older release, rewrite
    the CTE as a subquery:

    ```sql
    -- accepted everywhere
    SELECT customer, COUNT(*)
    FROM (SELECT * FROM orders WHERE created_at > '2026-01-01') recent
    GROUP BY customer
    ```

    A **data-modifying** CTE is still refused on every version, by the
    write-keyword scan rather than by the opening token:
    `WITH w AS (INSERT INTO t … RETURNING *) SELECT * FROM w` returns 400 with
    `Blocked keyword: INSERT`.

    On **SQL Server** the row ceiling is not pushed into a CTE as a `TOP`
    clause, because `TOP` has to sit on the statement's final `SELECT`. The
    response is capped and `truncated` is still accurate — the source simply
    does more work before the rows are cut. Every other source pushes a `LIMIT`
    down as usual.

### Multiple statements

**One statement per request, from community 0.7.0 / enterprise 0.11.0.**
`SELECT 1; SELECT 2` returns 400 with `Only one statement per query is
allowed`, and is audited as `sql_validation_failed`. A `;` inside a string
literal or comment does not count, and a single trailing `;` is fine. (Before
community 0.7.1 / enterprise 0.11.1 a trailing `;` on a query with no `LIMIT`
of its own failed with a 500, because the row cap was appended after it.)

Earlier releases accepted several statements, ran them all and returned only
the last result set. Only the first statement's opening keyword was checked, so
a second statement could be something the write-keyword list does not name —
which is why this is now refused rather than documented.

### Literals and comments are read the way every engine reads them

**From community 0.7.2 / enterprise 0.12.0.** The engines TDB runs on disagree
about where a string or comment ends: PostgreSQL and DuckDB read `$$…$$` and
`E'…'` strings and nest `/* */` comments; MySQL treats a backslash inside a
string as an escape and starts a comment at `#`; Snowflake honours backslashes
and `//` comments. TDB checks the SQL under **each** of those readings, and
refuses it if any reading finds a write keyword or a second statement. On a CSV
source, DuckDB's own parser must also read the query as exactly one `SELECT`.

| SQL | Accepted |
|---|---|
| `SELECT $$it's$$ AS a` | ✅ dollar-quoted strings |
| `SELECT data #>> '{a,b}' FROM t` | ✅ Postgres JSON operators |
| `SELECT 1 /* outer /* inner */ still a comment */` | ✅ nested comments |
| a query that one engine would read as two statements | ❌ `Only one statement per query is allowed` |

Both refusals are HTTP 400 and audited as `sql_validation_failed`. Ordinary SQL
is unaffected; what is refused is SQL whose meaning depends on which engine
reads it. Earlier releases read literals the ANSI way only — see the
[security advisory](../security/updates.md#where-advisories-are-published).

### SQL reads only the registered file

A query on a CSV or Parquet source can read that source's own file and no other, **from
community 0.15.0 / enterprise 0.26.0**. `read_csv`, `read_text`, `glob` and the like on any
other file, including one in the same directory, return **403** and are audited as
`sql_file_access`. From community 0.7.0 / enterprise 0.11.0 until then, the limit was the
source's data directory. Earlier releases let DuckDB open any path written in the SQL — see the
[security advisory](../security/updates.md#where-advisories-are-published).

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
{"detail": "SQL validation failed: Blocked keyword: DELETE"}
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
| 400 | `Query failed: …` — the source rejected the SQL: an unknown column or table, a syntax error, a value that does not convert. The source's own message follows. *From community 0.12.4 / enterprise 0.20.3; earlier releases returned 500.* |
| 401 | Missing or invalid auth token |
| 404 | `source_id` (name or UUID) not found |
| 429 | Rate limit exceeded (DB-managed keys only) |
| 500 | Query execution error — the source or TDB failed (unreachable source, out of memory). Not the caller's SQL. |

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
