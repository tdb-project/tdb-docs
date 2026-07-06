# MySQL Connector

The MySQL connector provides read-only SQL access to a MySQL database.
A source can be registered in two modes: **database-wide** (all tables) or **single-table**
(scoped to one specific table).

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
| `table` | string | No | — | Omit to register the whole database; set to scope to one table |

Note: MySQL uses `database` (not `dbname`) for the database name, matching
MySQL's own terminology. There is no separate schema field — MySQL databases
are their own namespace.

---

## Registration modes

### Database-wide (recommended)

Omit `table` to register an entire database as a single source. One registration
covers all tables. Users can query any table or write JOINs across tables using
standard SQL.

```bash
curl -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer <YOUR_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production_db",
    "source_type": "mysql",
    "connection": {
      "host": "db.internal",
      "port": 3306,
      "database": "production",
      "user": "tdb_reader",
      "password": "s3cret"
    },
    "description": "Production database — all tables"
  }'
```

The schema endpoint returns all tables and their columns:

```json
{
  "source_name": "production_db",
  "columns": [],
  "tables": [
    { "name": "customers", "columns": [{"name": "id", "type": "int"}, ...] },
    { "name": "orders",    "columns": [{"name": "id", "type": "int"}, ...] }
  ]
}
```

### Single-table

Include `table` to scope the source to one specific table. Useful when you want
to expose individual tables as separate named sources (see
[Registering the same database more than once](#registering-the-same-database-more-than-once)).

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
    "description": "Order records only",
    "tags": ["production", "finance"]
  }'
```

The schema endpoint returns that table's columns in the `columns` field (`tables` is null).

---

## Registering the same database more than once

Source **names** must be unique (case-insensitive; a duplicate name returns HTTP 409) —
the connection details do not. Registering the same database several times is
supported: one single-table source per table, or a database-wide source alongside
table-scoped ones. If you prefer the per-table approach:

```bash
# Register two tables from the same database as separate sources
POST /v1/sources  →  { "name": "orders",   "connection": { ..., "table": "orders" } }
POST /v1/sources  →  { "name": "products", "connection": { ..., "table": "products" } }
```

Cautions when doing this:

- **Table scoping is not query enforcement.** The `table` field scopes what the
  schema endpoint reports — it does not restrict SQL. Queries are sent to MySQL
  as-is, so a query against the `orders` source can still read (or JOIN) any table
  the configured database user can see. If per-table sources are meant to be real
  access boundaries, give each source its own database user whose grants cover
  only that table (see [Minimum required permissions](#minimum-required-database-permissions)).
- **Credentials are stored per source.** Each registration keeps its own copy of
  the connection config. When you rotate the database password, delete and
  re-register every source that points at that database — there is no update
  endpoint.
- **Overlapping sources duplicate the catalog.** A database-wide source plus
  single-table sources over the same tables expose the same data under several
  source names — in the sources list, in MCP tool responses, and in audit log
  entries. Pick one style per database unless you have a reason to mix them.

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

The response shape depends on the registration mode:

- **Database-wide:** `tables` array is populated; `columns` is empty.
  Each element names a table and lists its columns in `ordinal_position` order.
- **Single-table:** `columns` array is populated; `tables` is null.
  Lists the registered table's columns in `ordinal_position` order.

The introspection query runs on demand — it always reflects the current live
database structure.

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
  "sql": "SELECT * FROM orders",
  "limit": 500
}
```

Write your SQL using real database table names — your SQL is sent to MySQL as-is.
There is no `data` alias for database sources (that's CSV-only). For database-wide
sources you can JOIN across any tables in the database.

The `limit` field in the query request sets the per-query maximum. TDB appends `LIMIT <n>` to
your SQL if no `LIMIT` clause is present. If your SQL already contains a `LIMIT` clause, TDB
uses it as-is.

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
