# SQL Server Connector

The SQL Server connector provides read-only SQL access to a SQL Server database.
A source can be registered in two modes: **database-wide** (all tables in the
configured schema) or **single-table** (scoped to one specific table).

---

## Prerequisites

The connector uses **Microsoft ODBC Driver 18 for SQL Server**, which must be
installed on the host running TDB before you can register a SQL Server source.

=== "Ubuntu / Debian"

    ```bash
    # Add the Microsoft package repository
    curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | sudo gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg
    curl https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/prod.list \
      | sudo tee /etc/apt/sources.list.d/mssql-release.list

    sudo apt-get update
    sudo ACCEPT_EULA=Y apt-get install -y msodbcsql18
    ```

=== "macOS"

    ```bash
    brew tap microsoft/mssql-release https://github.com/Microsoft/homebrew-mssql-release
    brew update
    HOMEBREW_ACCEPT_EULA=Y brew install msodbcsql18
    ```

=== "Docker"

    Use a base image that already includes the driver, or add the install steps to
    your `Dockerfile` before installing TDB:

    ```dockerfile
    FROM ubuntu:22.04
    RUN apt-get update && apt-get install -y curl gnupg lsb-release && \
        curl -fsSL https://packages.microsoft.com/keys/microsoft.asc \
          | gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg && \
        curl https://packages.microsoft.com/config/ubuntu/22.04/prod.list \
          > /etc/apt/sources.list.d/mssql-release.list && \
        apt-get update && ACCEPT_EULA=Y apt-get install -y msodbcsql18
    # ... remainder of your TDB install
    ```

---

## Connection configuration

When registering a SQL Server source, the `connection` object requires:

| Key | Type | Required | Default | Description |
|---|---|---|---|---|
| `host` | string | Yes | — | SQL Server host or IP |
| `port` | integer | Yes | — | SQL Server port (typically 1433) |
| `database` | string | Yes | — | Database name |
| `user` | string | Yes | — | SQL Server login |
| `password` | string | Yes | — | Login password |
| `table` | string | No | — | Omit to register the whole database; set to scope to one table |
| `schema` | string | No | `"dbo"` | Schema name (scopes both modes) |
| `driver` | string | No | `"ODBC Driver 18 for SQL Server"` | ODBC driver name as listed in `odbcinst.ini` |
| `trust_server_certificate` | boolean | No | `false` | Set `true` to accept self-signed certificates (dev / Docker only) |

### Connection string format

TDB constructs the pyodbc connection string internally as:

```
DRIVER={ODBC Driver 18 for SQL Server};SERVER=host,port;DATABASE=...;UID=...;PWD=...;Encrypt=yes;TrustServerCertificate=yes/no;
```

Encryption is always enabled (`Encrypt=yes`). The `trust_server_certificate` field
controls whether the certificate chain is validated.

---

## Registration modes

### Database-wide (recommended)

Omit `table` to register an entire database as a single source. One registration
covers all tables in the configured `schema` (default `dbo`). Users can query any
table or write JOINs across tables using standard SQL.

```bash
curl -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer <YOUR_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production_db",
    "source_type": "sqlserver",
    "connection": {
      "host": "sql.internal",
      "port": 1433,
      "database": "production",
      "user": "tdb_reader",
      "password": "s3cret",
      "schema": "dbo"
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
    "source_type": "sqlserver",
    "connection": {
      "host": "sql.internal",
      "port": 1433,
      "database": "production",
      "user": "tdb_reader",
      "password": "s3cret",
      "table": "orders",
      "schema": "dbo"
    },
    "description": "Order records only",
    "tags": ["production", "finance"]
  }'
```

The schema endpoint returns that table's columns in the `columns` field (`tables` is null).

Either mode pairs well with [**typed YAML views**](../api/views.md): pre-approved,
named queries — including cross-table JOINs — that consumers call by name instead
of writing raw SQL.

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
  schema endpoint reports — it does not restrict SQL. Queries are sent to SQL Server
  as-is, so a query against the `orders` source can still read (or JOIN) any table
  the configured login can see. If per-table sources are meant to be real access
  boundaries, give each source its own login whose grants cover only that table
  (see [Minimum required permissions](#minimum-required-database-permissions)).
- **Credentials are stored per source.** Each registration keeps its own copy of
  the connection config. When you rotate the login password, delete and
  re-register every source that points at that database — there is no update
  endpoint.
- **Overlapping sources duplicate the catalog.** A database-wide source plus
  single-table sources over the same tables expose the same data under several
  source names — in the sources list, in MCP tool responses, and in audit log
  entries. Pick one style per database unless you have a reason to mix them.

---

## Read-only enforcement

SQL Server has no session-level read-only flag equivalent to PostgreSQL's
`conn.read_only` or MySQL's `transaction_read_only`. Two layers are used instead:

1. **SQL validator** — TDB rejects any SQL that does not start with `SELECT` or `WITH`
   before it reaches the database. Returns HTTP 400 with a clear error message.

2. **Always-rollback transactions** — Every connection is opened with
   `autocommit=False`. After every query — including schema introspection and
   connection validation — the connection is **always rolled back** in the `finally`
   block. A write that somehow bypassed the validator would be rolled back and could
   not persist.

The production recommendation is to compound these application-level safeguards with
a database login that holds only `SELECT` permissions (see
[Minimum required permissions](#minimum-required-database-permissions)).

---

## Schema introspection

Schema is pulled live from `information_schema.columns`:

```bash
GET /v1/sources/<source_id>/schema
```

Results are filtered by the configured `schema` (default `dbo`). The response shape
depends on the registration mode:

- **Database-wide:** `tables` array is populated; `columns` is empty.
  Each element names a table and lists its columns in `ordinal_position` order.
- **Single-table:** `columns` array is populated; `tables` is null.
  Lists the registered table's columns in `ordinal_position` order.

The introspection query runs on demand — it always reflects the current live
database structure.

---

## Supported data types

The connector returns all SQL Server data types. Column types in the schema response
are the raw SQL Server `data_type` strings from `information_schema`. Examples:

| SQL Server type | Returned as |
|---|---|
| `int`, `bigint`, `smallint`, `tinyint` | `int`, `bigint`, `smallint`, `tinyint` |
| `decimal`, `numeric`, `float`, `real` | `decimal`, `numeric`, `float`, `real` |
| `varchar`, `nvarchar`, `char`, `nchar` | `varchar`, `nvarchar`, `char`, `nchar` |
| `datetime2`, `datetime`, `date`, `time` | `datetime2`, `datetime`, `date`, `time` |
| `bit` | `bit` |
| `uniqueidentifier` | `uniqueidentifier` |

Row values are serialised to JSON by the API layer. `datetime2` and
`uniqueidentifier` values are returned as strings.

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

Write your SQL using real database table names — your SQL is sent to SQL Server as-is.
There is no `data` alias for database sources (that's CSV-only). For database-wide
sources you can JOIN across any tables in the schema. SQL Server uses `TOP n` rather
than `LIMIT`. TDB injects `SELECT TOP <n>` into your query automatically if no `TOP`
clause is present. If your SQL already includes `TOP`, TDB leaves it unchanged.

Note: **do not use `LIMIT` in hand-written SQL for this connector** — SQL Server does
not support that syntax. Use `TOP n` instead:

```sql
-- Correct for SQL Server
SELECT TOP 200 * FROM orders WHERE status = 'open'

-- Incorrect — will cause a syntax error
SELECT * FROM orders WHERE status = 'open' LIMIT 200
```

---

## Minimum required database permissions

Create a dedicated read-only login for TDB:

```sql
-- Create a server-level login
CREATE LOGIN tdb_reader WITH PASSWORD = 'strong-password';

-- Create a database user mapped to the login
USE production;
CREATE USER tdb_reader FOR LOGIN tdb_reader;

-- Option 1: grant SELECT on a specific table
GRANT SELECT ON dbo.orders TO tdb_reader;

-- Option 2: use the db_datareader role (grants SELECT on all tables in the database)
ALTER ROLE db_datareader ADD MEMBER tdb_reader;

-- Option 3: grant SELECT on an entire schema
GRANT SELECT ON SCHEMA::dbo TO tdb_reader;
```

TDB does not need `INSERT`, `UPDATE`, `DELETE`, `CREATE`, or any other permission.

---

## Connection pooling

TDB opens a **fresh connection per query** and closes it immediately after. There is
no connection pool in the current release. For high-frequency query workloads, consider
placing a connection pooler between TDB and SQL Server.

---

## Troubleshooting

### `connection refused` on registration

TDB validates the connection by running `SELECT 1` when you call
`GET /v1/sources/<id>/schema`. If the host is unreachable or credentials are wrong,
you'll see HTTP 503:

```json
{"detail": "Source 'orders' (sqlserver) is not accessible."}
```

Check that:

- The `host` and `port` are reachable from the TDB server
- The `user` and `password` are correct
- The `database` exists and the login has `CONNECT` access

### Certificate errors in development

By default, TDB requires a trusted TLS certificate (`TrustServerCertificate=no`).
For development environments or Docker setups with self-signed certificates, set:

```json
"trust_server_certificate": true
```

Do not enable this in production — use a properly signed certificate instead.

### Named instances

If your SQL Server uses a named instance (e.g., `SQLSERVER\SQLEXPRESS`), supply the
instance name in the `host` field using a backslash:

```json
"host": "SQLSERVER\\SQLEXPRESS"
```

Named instances typically use a dynamic port. Either configure SQL Server Browser
to resolve the instance, or pin a static port and use that in the `port` field.

### ODBC driver not found

If TDB cannot find the driver, verify the driver name reported by your system:

```bash
# Linux
odbcinst -q -d

# macOS
/usr/local/etc/odbcinst.ini
```

Set the `driver` field in the connection config to exactly match the name shown there.

### `permission denied` on query

If TDB can connect but queries fail, the database login may lack `SELECT` on the
target table or schema. Check your SQL Server grants (see
[Minimum required permissions](#minimum-required-database-permissions)).
