# MySQL Connector

The MySQL connector provides read-only SQL access to a single MySQL table
per registered source. Multiple sources can point to different tables in the same
or different databases.

---

## Connection configuration

When registering a MySQL source, the `connection` object requires:

| Key | Type | Required | Default | Description |
|---|---|---|---|---|
| `host` | string | Yes | — | MySQL host or IP |
| `port` | integer | Yes | — | MySQL port (typically 3306) |
| `database` | string | Yes | — | Database name |
| `user` | string | Yes | — | Database user |
| `password` | string | Yes | — | Database password |
| `table` | string | Yes | — | Table name this source maps to |

Note: MySQL uses `database` (not `dbname`) for the database name, matching
MySQL's own terminology. There is no separate schema field — MySQL databases
are their own namespace.

### Registration example

```bash
curl -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer <YOUR_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "orders",
    "source_type": "mysql",
    "connection": {
      "host": "db.internal",
      "port": 3306,
      "database": "production",
      "user": "tdb_reader",
      "password": "s3cret",
      "table": "orders"
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

2. **Session-level read-only** — Every MySQL connection executes
   `SET @@SESSION.transaction_read_only = 1` immediately after opening. This means
   the MySQL engine itself will reject any write attempt, even if it somehow bypassed
   the validator. This mirrors the behaviour of psycopg3's `conn.read_only = True`
   for PostgreSQL — the enforcement is at the database engine level, not just
   application-level parsing.

This means your `tdb_reader` database user does not need `INSERT`, `UPDATE`, `DELETE`,
or `DDL` privileges — `SELECT` only is sufficient and recommended.

---

## Schema introspection

Schema is pulled live from `information_schema.columns`:

```bash
GET /v1/sources/<source_id>/schema
```

Returns column names and their MySQL data types in `ordinal_position` order.
The introspection query runs on demand — it always reflects the current table definition.

---

## Supported data types

The connector returns all MySQL data types. Column types in the schema response are
the raw MySQL `data_type` strings from `information_schema`. Examples:

| MySQL type | Returned as |
|---|---|
| `int`, `bigint`, `smallint`, `tinyint` | `int`, `bigint`, `smallint`, `tinyint` |
| `decimal`, `float`, `double` | `decimal`, `float`, `double` |
| `varchar`, `char` | `varchar`, `char` |
| `text`, `mediumtext`, `longtext` | `text`, `mediumtext`, `longtext` |
| `datetime`, `timestamp` | `datetime`, `timestamp` |
| `date` | `date` |
| `tinyint(1)` | `tinyint` (commonly used as boolean) |

Row values are serialised to JSON by the API layer. Datetime values are returned
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
-- Create a dedicated read-only user
CREATE USER 'tdb_reader'@'%' IDENTIFIED BY 'strong-password';

-- Grant SELECT on specific tables
GRANT SELECT ON production.orders TO 'tdb_reader'@'%';
GRANT SELECT ON production.products TO 'tdb_reader'@'%';

-- Or grant SELECT on all tables in a database
GRANT SELECT ON production.* TO 'tdb_reader'@'%';

FLUSH PRIVILEGES;
```

Replace `production` with your database name and `%` with a specific host if you
want to restrict connections by source IP.

TDB does not need `INSERT`, `UPDATE`, `DELETE`, `CREATE`, or any other privilege.

---

## Connection pooling

TDB opens a **fresh connection per query** and closes it immediately after. There is
no connection pool in the current release. For high-frequency query workloads, place
a connection pooler (ProxySQL, MySQL Router) between TDB and MySQL.

---

## Troubleshooting

### `connection refused` on registration

TDB validates the connection by running `SELECT 1` when you call
`GET /v1/sources/<id>/schema`. If the host is unreachable or credentials are wrong,
you'll see HTTP 503:

```json
{"detail": "Source 'orders' (mysql) is not accessible."}
```

Check that:

- The `host` and `port` are reachable from the TDB server
- The `user` and `password` are correct
- The `database` exists and the user has been granted access

### `permission denied` on query

If TDB can connect but queries fail, the database user may lack `SELECT` on the
target table. Check your MySQL grants (see [Minimum required permissions](#minimum-required-database-permissions)).

### `Access denied` when connecting remotely

MySQL grants are host-specific. A user created with `'tdb_reader'@'localhost'`
cannot connect from a remote TDB server. Use `'tdb_reader'@'%'` or specify the
exact TDB server IP in the host portion of the grant.

### SSL connections

SSL support is not yet configurable in the `connection` object. Connections use
the PyMySQL default (SSL not required). SSL configuration will be added to the
connection schema in a future release.
