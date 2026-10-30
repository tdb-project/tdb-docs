# DuckDB Connector

!!! info "Enterprise, from 0.23.0"
    The DuckDB connector is part of TDB Enterprise. Community reads CSV only.

The DuckDB connector gives read-only SQL access to a DuckDB database file: every table
and view in one of its schemas, or a single table. Queries use the file's real table
names, as on a PostgreSQL source. TDB opens the file with its built-in DuckDB engine;
nothing is copied or loaded ahead of time, and the process that writes the file can keep
writing to it.

---

## Connection configuration

| Key | Type | Required | Description |
|---|---|---|---|
| `file_path` | string | Yes | Path to the `.duckdb` file, as the TDB server sees it |
| `schema` | string | No | The schema the source covers. Default `main` |
| `table` | string | No | One table instead of the whole schema |

In Docker, put the file in the directory mounted at `/data` and use the path inside the
container, for example `/data/analytics.duckdb`.

```bash
curl -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer <YOUR_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "analytics",
    "source_type": "duckdb",
    "connection": {"file_path": "/data/analytics.duckdb"}
  }'
```

Registration checks that the file exists, is readable, is a DuckDB database, and that
TDB's DuckDB can open it. A damaged file, or one written by a newer DuckDB than TDB
ships, returns `400` naming TDB's DuckDB version, and nothing is registered. A file that
another process is writing at that moment is accepted: TDB can't open it to check, and
its queries will say it is busy until the writer finishes.

---

## Querying

Without `table`, the source covers the whole schema. Use the real table and view names,
and join across them:

```sql
SELECT c.region, SUM(o.amount) AS total
FROM orders o
JOIN customers c ON c.id = o.customer_id
GROUP BY c.region
```

`GET /v1/sources/{name}/schema` and the MCP `schema_source` tool list every table and
view in the schema with DuckDB's own types, such as `DECIMAL(10,2)`. The MCP tools that
build SQL for you (`preview_source`, `filter_source`, `aggregate_source`) take a `table`
argument on a database-wide source. [Annotations](../api/sources.md) and named
[views](../api/views.md) work as on any source.

Each query sees the file as it is when the query starts, so rows the owner writes between
queries are in the next result.

---

## The file stays yours

DuckDB lets only one process open a file for writing, and while that process has it open
no other process can read it. TDB therefore opens the file only while a query runs and
closes it when the last overlapping query finishes:

- **Between queries**, the process that maintains the file can open it and write.
- **While that process has the file open**, a query returns **`503`**, audited as
  **`source_busy`**, with *"Another process is writing to this DuckDB file; try again when
  it has finished."* Over MCP, `query_source` returns the same message as a tool error.
  Retry after the write, or have the writer open the file only for as long as it writes.
- **Opening per query costs about 25 ms** on a small file.

If the file is written by a scheduled job, a query during the job is refused rather than
answered from half-written data.

---

## How values are returned

DuckDB values follow the [same rules as every source](../api/query.md#how-values-are-returned):
`DECIMAL` as a string with its exact digits, `BLOB` as base64, `UUID` as a string,
`LIST` and `STRUCT` as JSON arrays and objects, and timestamps as ISO 8601. REST and
MCP return the same values.

---

## Security

- **Read-only.** TDB opens the file read-only, and DuckDB's own parser must see exactly
  one `SELECT`. Anything else returns `400` and is audited as `sql_validation_failed`.
- **The file and nothing else.** Each file has its own engine, which can open no other
  file: not one outside the data directory, and not one beside it. A query such as
  `read_csv('/data/other.csv')` returns `403`, audited as `sql_file_access`. So does a
  view stored in the file that reads an outside file. One DuckDB source can't reach
  another's tables.
- **Confined to the data directory.** With [`TDB_ALLOWED_DATA_DIR`](../reference/environment-variables.md)
  set (the Docker images set `/data`), registering a file outside it returns `403`.
- **Column and row policies** apply (Mid-Market). Under a rule, use the bare or
  `main.`-qualified table name; a name qualified with the file's catalog
  (`analytics.main.orders`) is refused. See [Column and row policies](../security/policies.md).
- **Row cap.** `limit` and `TDB_MAX_ROWS` apply as on every source. A cut result reports
  `"truncated": true`.
- **No timeout.** Like CSV and Parquet, a DuckDB query runs in TDB's own engine, so
  `TDB_QUERY_TIMEOUT` can't cancel it.

---

## Not supported

- **The [PostgreSQL wire gateway](../integrations/postgres-wire.md)** serves PostgreSQL
  sources only. Reach a DuckDB source over REST, MCP or views.
- **MotherDuck** (`md:` paths). Local files only.

---

## Troubleshooting

### `DuckDB file not found or not readable` on registration

The path doesn't exist inside the container, the TDB user can't read it, or the file
isn't a DuckDB database. Check the path against your `/data` mount.

### `This DuckDB (x.y.z) cannot open the file` on registration

The file is damaged, or was written by a newer DuckDB than the one TDB ships. Write it
with the DuckDB version named in the message, or an older one.

### `503` with *Another process is writing to this DuckDB file*

The process that maintains the file has it open. Queries work again once it closes the
file.

### `503` with *not accessible*

The file was there at registration but is missing, unreadable or damaged now. The
response doesn't include the server path. Restore the file, or delete the source and
register it again.
