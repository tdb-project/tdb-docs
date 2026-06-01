# PostgreSQL Connector

The PostgreSQL connector provides read-only SQL access to a single Postgres table
per registered source. Multiple sources can point to different tables in the same
or different databases.

---

## Connection configuration

When registering a PostgreSQL source, the `connection` object requires:

| Key | Type | Required | Default | Description |
|---|---|---|---|---|
| `host` | string | Yes | — | Postgres host or IP |
| `port` | integer | Yes | — | Postgres port (typically 5432) |
| `dbname` | string | Yes | — | Database name |
| `user` | string | Yes | — | Database user |
| `password` | string | Yes | — | Database password |
| `table` | string | Yes | — | Table name this source maps to |
| `schema` | string | No | `"public"` | Postgres schema name |

### Registration example

```bash
curl -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer <YOUR_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "orders",
    "source_type": "postgres",
    "connection": {
      "host": "db.internal",
      "port": 5432,
      "dbname": "production",
      "user": "tdb_reader",
      "password": "s3cret",
      "table": "orders",
      "schema": "public"
    },
    "description": "Order records from the production database",
    "tags": ["production", "finance"]
  }'
```

---

## One source = one table

Each registered source maps to exactly one table. This is by design — it matches the
CSV connector's model and keeps the permission surface predictable. To expose multiple
tables, register multiple sources:

```bash
# Register two tables from the same database
POST /v1/sources  →  { "name": "orders",   "connection": { ..., "table": "orders" } }
POST /v1/sources  →  { "name": "products", "connection": { ..., "table": "products" } }
```

Cross-table JOINs and multi-table queries are supported via [**typed YAML views**](../api/views.md).

---

## Read-only enforcement

Read-only access is enforced at two independent layers:

1. **SQL validator** — TDB rejects any SQL that does not start with `SELECT` or `WITH`
   before it reaches the database. Returns HTTP 400 with a clear error message.

2. **Transaction-level read-only** — Every Postgres connection is opened with
   `connection.read_only = True`. This means the Postgres engine itself will reject
   any write attempt, even if it somehow bypassed the validator. You cannot accidentally
   grant write access by misconfiguring the validator.

This means your `tdb_reader` database user does not need `INSERT`, `UPDATE`, `DELETE`,
or `DDL` privileges — `SELECT` only is sufficient and recommended.

---

## Schema introspection

Schema is pulled live from `information_schema.columns`:

```bash
GET /v1/sources/<source_id>/schema
```

Returns column names and their Postgres data types in `ordinal_position` order.
The introspection query runs on demand — it always reflects the current table definition.

---

## Supported data types

The connector returns all Postgres data types. Column types in the schema response are
the raw Postgres `data_type` strings from `information_schema`. Examples:

| Postgres type | Returned as |
|---|---|
| `integer`, `bigint`, `smallint` | `integer`, `bigint`, `smallint` |
| `numeric`, `real`, `double precision` | `numeric`, `real`, `double precision` |
| `character varying`, `text` | `character varying`, `text` |
| `boolean` | `boolean` |
| `date` | `date` |
| `timestamp without time zone` | `timestamp without time zone` |
| `timestamp with time zone` | `timestamp with time zone` |
| `jsonb`, `json` | `jsonb`, `json` |
| `uuid` | `uuid` |

Row values are serialised to JSON by the API layer. Timestamps and UUIDs are returned
as strings.

---

## Row limit

The default query limit is **100 rows**. The hard cap is **1,000 rows**.

```json
{
  "source_id": "...",
  "sql": "SELECT * FROM data",
  "limit": 500
}
```

The `limit` field in the query request sets the per-query maximum. TDB appends
`LIMIT <n>` to your SQL if no `LIMIT` clause is present. If your SQL already
contains a `LIMIT` clause, TDB uses it as-is.

---

## Minimum required database permissions

Create a dedicated read-only user for TDB:

```sql
-- Create a dedicated read-only role
CREATE ROLE tdb_reader LOGIN PASSWORD 'strong-password';

-- Grant SELECT on specific tables
GRANT SELECT ON TABLE public.orders TO tdb_reader;
GRANT SELECT ON TABLE public.products TO tdb_reader;

-- Or grant SELECT on all current and future tables in a schema
GRANT SELECT ON ALL TABLES IN SCHEMA public TO tdb_reader;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO tdb_reader;

-- Grant schema usage (required for any table access)
GRANT USAGE ON SCHEMA public TO tdb_reader;
```

TDB does not need `CREATE`, `INSERT`, `UPDATE`, `DELETE`, or `TRUNCATE` permissions.

---

## Connection pooling

TDB opens a **fresh connection per query** and closes it immediately after. There is
no connection pool in the current release. For high-frequency query workloads, place
a connection pooler (PgBouncer, RDS Proxy) between TDB and Postgres.

---

## Troubleshooting

### `connection refused` on registration

TDB validates the connection by running `SELECT 1` when you call
`GET /v1/sources/<id>/schema`. If the host is unreachable or credentials are wrong,
you'll see HTTP 503:

```json
{"detail": "Source 'orders' (postgres) is not accessible."}
```

Check that:
- The `host` and `port` are reachable from the TDB server
- The `user` and `password` are correct
- The `dbname` exists and the user has `CONNECT` privilege

### `permission denied` on query

If TDB can connect but queries fail, the database user may lack `SELECT` on the
target table. Check your Postgres grants (see [Minimum required permissions](#minimum-required-database-permissions)).

### SSL connections

SSL support (sslmode, client certificates) is not yet configurable in the
`connection` object. Connections use the psycopg3 default (prefer SSL when available).
SSL configuration will be added to the connection schema in a future release.
