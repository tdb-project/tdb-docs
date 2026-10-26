# Parquet Connector

!!! info "Enterprise, from 0.20.0"
    The Parquet connector is part of TDB Enterprise. Community reads CSV only.

The Parquet connector gives read-only SQL access to a single Parquet file. Like a CSV
source, the file appears as one table named **`data`**, and TDB's built-in DuckDB engine
reads it in place. No database server is involved and nothing is copied or loaded ahead
of time.

---

## Connection configuration

| Key | Type | Required | Description |
|---|---|---|---|
| `file_path` | string | Yes | Path to the `.parquet` file, as the TDB server sees it |

In Docker, put the file in the directory mounted at `/data` and use the path inside the
container, for example `/data/orders.parquet`.

```bash
curl -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer <YOUR_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "orders",
    "source_type": "parquet",
    "connection": {"file_path": "/data/orders.parquet"}
  }'
```

Registration checks that the file exists, is readable, and is a complete Parquet file
(it starts and ends with Parquet's `PAR1` marker). Otherwise it returns `400` and nothing is registered.
That catches a CSV registered as Parquet, and a file that is still being copied.

---

## Querying

```sql
SELECT region, SUM(amount) AS total
FROM data
GROUP BY region
```

Every MCP tool works on a Parquet source: `query_source`, `schema_source`,
`preview_source`, `filter_source` and `aggregate_source`, plus named [views](../api/views.md).
`schema_source` and `GET /v1/sources/{name}/schema` report the file's own column types,
such as `DECIMAL(10,2)` or `TIMESTAMP`.

The file is read on every query, so a replaced or rewritten file is picked up by the next
query. One source is one file; to serve several files, register each one as its own source.

---

## How values are returned

Parquet has types a CSV never produces. REST and MCP responses return them as JSON
like this:

| Parquet / DuckDB type | Returned as | Example |
|---|---|---|
| Integers, `DOUBLE`, `BOOLEAN`, `VARCHAR` | JSON number, boolean or string | `42`, `true`, `"EU"` |
| `DECIMAL` | String with the exact digits | `"0.10"` |
| `DATE`, `TIMESTAMP` | String | REST `"2026-01-01T10:00:00"`; MCP `"2026-01-01 10:00:00"` |
| `UUID` | String | `"4b48af8e-b7dd-45e2-836e-e8efa86e41c2"` |
| `BLOB` | Base64 string | `"AP8="` for bytes `00 ff` |
| `LIST` | JSON array | `[1, 2]` |
| `STRUCT` | JSON object | `{"k": 1}` |
| `NULL` | `null` | `null` |

`DECIMAL` is a string so that a value like `0.10` keeps its exact digits rather than
becoming a binary floating-point number. Values inside lists and structs follow the same
rules. Timestamps are separated by `T` on REST and by a space on MCP, the same as for a
CSV source.

---

## Security

A Parquet source has the same protections as a CSV source:

- **Read-only.** DuckDB's own parser must see exactly one `SELECT`. Anything else returns
  `400` and is audited as `sql_validation_failed`.
- **Confined to the data directory.** With [`TDB_ALLOWED_DATA_DIR`](../reference/environment-variables.md)
  set (the Docker images set `/data`), registering a file outside it returns `403`. A
  query that names another file, such as `read_parquet('/elsewhere/x.parquet')`, returns
  `403` and is audited as `sql_file_access`.
- **Row cap.** `limit` and `TDB_MAX_ROWS` apply as on every source. A cut result reports
  `"truncated": true`.
- **No timeout.** Like CSV, a Parquet query runs in TDB's own engine, so
  `TDB_QUERY_TIMEOUT` cannot cancel it.

---

## Troubleshooting

### `Parquet file not found or not readable` on registration

The path doesn't exist inside the container, the TDB user can't read it, or the file isn't
a complete Parquet file. Check the path against your `/data` mount, and that the file was
written completely.

### `503` on a query

The file was there at registration but is missing or unreadable now. The response
doesn't include the server path. Restore the file, or delete the source and register it
again.
