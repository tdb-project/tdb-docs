# Snowflake Connector

The Snowflake connector provides read-only SQL access to a Snowflake database.
A source can be registered in two modes: **database-wide** (all tables in the
configured schema) or **single-table** (scoped to one specific table).

---

## Connection configuration

When registering a Snowflake source, the `connection` object requires:

| Key | Type | Required | Default | Description |
|---|---|---|---|---|
| `account` | string | Yes | — | Snowflake account identifier (e.g. `xy12345.us-east-1` or `orgname-accountname`) |
| `user` | string | Yes | — | Snowflake username |
| `password` | string | Yes | — | User password |
| `database` | string | Yes | — | Database name |
| `table` | string | No | — | Omit to register the whole database; set to scope to one table |
| `schema` | string | No | `"PUBLIC"` | Schema name (scopes both modes) |
| `warehouse` | string | No | — | Virtual warehouse to use; uses the account default if omitted |
| `role` | string | No | — | Snowflake role to assume; uses the user's default role if omitted |

### Account identifier format

Snowflake supports two account identifier formats:

- **Legacy format**: `xy12345.us-east-1` (account locator with region)
- **New format**: `orgname-accountname` (organisation and account name)

Use the format shown in your Snowflake URL or provided by your Snowflake administrator.
If you are unsure which format to use, run `SELECT CURRENT_ACCOUNT()` in a Snowflake
worksheet — the result is your account locator.

---

## Registration modes

### Database-wide (recommended)

Omit `table` to register an entire database as a single source. One registration
covers all tables in the configured `schema` (default `PUBLIC`). Users can query
any table or write JOINs across tables using standard SQL.

```bash
curl -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer <YOUR_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "sales_db",
    "source_type": "snowflake",
    "connection": {
      "account": "xy12345.us-east-1",
      "user": "tdb_reader",
      "password": "s3cret",
      "database": "SALES_DB",
      "schema": "PUBLIC",
      "warehouse": "COMPUTE_WH",
      "role": "TDB_READER"
    },
    "description": "Sales database — all tables"
  }'
```

The schema endpoint returns all tables and their columns:

```json
{
  "source_name": "sales_db",
  "columns": [],
  "tables": [
    { "name": "CUSTOMERS", "columns": [{"name": "ID", "type": "NUMBER"}, ...] },
    { "name": "ORDERS",    "columns": [{"name": "ID", "type": "NUMBER"}, ...] }
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
    "source_type": "snowflake",
    "connection": {
      "account": "xy12345.us-east-1",
      "user": "tdb_reader",
      "password": "s3cret",
      "database": "SALES_DB",
      "table": "ORDERS",
      "schema": "PUBLIC",
      "warehouse": "COMPUTE_WH",
      "role": "TDB_READER"
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
POST /v1/sources  →  { "name": "orders",   "connection": { ..., "table": "ORDERS" } }
POST /v1/sources  →  { "name": "products", "connection": { ..., "table": "PRODUCTS" } }
```

Cautions when doing this:

- **Table scoping is not query enforcement.** The `table` field scopes what the
  schema endpoint reports — it does not restrict SQL. Queries are sent to Snowflake
  as-is, so a query against the `orders` source can still read (or JOIN) any table
  the configured role can see. This matters even more on Snowflake, where there is
  no connector-level write safety net either (see
  [Read-only enforcement](#read-only-enforcement)). If per-table sources are meant
  to be real access boundaries, give each source its own role whose grants cover
  only that table (see [Minimum required permissions](#minimum-required-database-permissions)).
- **Credentials are stored per source.** Each registration keeps its own copy of
  the connection config. When you rotate the Snowflake password, delete and
  re-register every source that points at that database — there is no update
  endpoint.
- **Overlapping sources duplicate the catalog.** A database-wide source plus
  single-table sources over the same tables expose the same data under several
  source names — in the sources list, in MCP tool responses, and in audit log
  entries. Pick one style per database unless you have a reason to mix them.
- **Warehouse cost.** Every source registered against the same database runs its
  queries on the configured warehouse. Duplicated sources don't add cost by
  themselves, but each schema introspection is a live query — AI tools that
  enumerate many overlapping sources will trigger more warehouse activity.

---

## Read-only enforcement

Snowflake has no session-level read-only mode equivalent to PostgreSQL's `conn.read_only`
or MySQL's `transaction_read_only`, and Snowflake connections auto-commit by default —
there is no rollback safety net at the connector level.

Two layers are used instead:

1. **SQL validator** — TDB rejects any SQL that does not start with `SELECT` or `WITH`
   before it reaches Snowflake. Returns HTTP 400 with a clear error message.

2. **SELECT-only role** (strongly recommended) — Because there is no connector-level
   write prevention beyond the SQL validator, the production requirement for Snowflake
   is to connect with a role that holds only `SELECT` privilege on the target table or
   schema. The SQL validator is a defence-in-depth layer on top of this, not a
   substitute for it.

See [Minimum required permissions](#minimum-required-database-permissions) for the
full grant chain to set up a `tdb_reader` role.

---

## Schema introspection

Schema is pulled live from `information_schema.columns`:

```bash
GET /v1/sources/<source_id>/schema
```

Results are filtered by the configured `schema` (default `PUBLIC`). The response
shape depends on the registration mode:

- **Database-wide:** `tables` array is populated; `columns` is empty.
  Each element names a table and lists its columns in `ordinal_position` order.
- **Single-table:** `columns` array is populated; `tables` is null.
  Lists the registered table's columns in `ordinal_position` order.

The introspection query runs on demand — it always reflects the current live
database structure.

**Important:** Snowflake stores unquoted identifiers in UPPER CASE. Column names
returned by `GET /v1/sources/<id>/schema` will be uppercase unless the table was
created with quoted lowercase identifiers.

---

## Supported data types

The connector returns all Snowflake data types. Column types in the schema response
are the raw Snowflake `data_type` strings from `information_schema`. Examples:

| Snowflake type | Returned as |
|---|---|
| `NUMBER`, `INTEGER`, `BIGINT` | `NUMBER`, `INTEGER`, `BIGINT` |
| `FLOAT`, `DOUBLE` | `FLOAT`, `DOUBLE` |
| `TEXT`, `VARCHAR`, `CHAR` | `TEXT`, `VARCHAR`, `CHAR` |
| `BOOLEAN` | `BOOLEAN` |
| `DATE` | `DATE` |
| `TIMESTAMP_NTZ`, `TIMESTAMP_LTZ`, `TIMESTAMP_TZ` | `TIMESTAMP_NTZ`, `TIMESTAMP_LTZ`, `TIMESTAMP_TZ` |
| `VARIANT` | `VARIANT` |
| `ARRAY` | `ARRAY` |
| `OBJECT` | `OBJECT` |

Row values are serialised to JSON by the API layer. Timestamps are returned as
strings. `VARIANT`, `ARRAY`, and `OBJECT` columns are returned as their JSON
representation.

---

## Row limit

The default query limit is **100 rows**. The hard cap is **1,000 rows**.

```json
{
  "source_id": "...",
  "sql": "SELECT * FROM ORDERS",
  "limit": 500
}
```

Write your SQL using real database table names — your SQL is sent to Snowflake as-is.
There is no `data` alias for database sources (that's CSV-only). For database-wide
sources you can JOIN across any tables in the schema. The `limit` field in the query
request sets the per-query maximum. TDB appends `LIMIT <n>` to your SQL if no `LIMIT`
clause is present. If your SQL already contains a `LIMIT` clause, TDB uses it as-is.
Snowflake uses standard SQL `LIMIT` syntax.

---

## Minimum required database permissions

Create a dedicated read-only role for TDB. The full grant chain is required — each
level of the hierarchy (warehouse, database, schema, table) must be granted
independently:

```sql
-- Create a dedicated role
CREATE ROLE tdb_reader;

-- Grant warehouse usage (required to run queries)
GRANT USAGE ON WAREHOUSE compute_wh TO ROLE tdb_reader;

-- Grant database and schema access
GRANT USAGE ON DATABASE sales_db TO ROLE tdb_reader;
GRANT USAGE ON SCHEMA sales_db.public TO ROLE tdb_reader;

-- Grant SELECT on specific tables
GRANT SELECT ON TABLE sales_db.public.orders TO ROLE tdb_reader;

-- Assign the role to the TDB service user
GRANT ROLE tdb_reader TO USER tdb_service_user;
```

To grant access to multiple tables, add a `GRANT SELECT` line for each table, or
grant at the schema level to cover all current and future tables:

```sql
-- Grant SELECT on all current tables in a schema
GRANT SELECT ON ALL TABLES IN SCHEMA sales_db.public TO ROLE tdb_reader;

-- Automatically grant SELECT on future tables added to the schema
GRANT SELECT ON FUTURE TABLES IN SCHEMA sales_db.public TO ROLE tdb_reader;
```

TDB does not need `INSERT`, `UPDATE`, `DELETE`, `CREATE`, or any other privilege.

---

## Connection pooling

TDB opens a **fresh connection per query** and closes it immediately after. There is
no connection pool in the current release. Snowflake connections carry a small
overhead due to the authentication round-trip. For high-frequency query workloads,
this will be addressed in a future release with connection reuse.

---

## Troubleshooting

### `connection refused` or authentication failure on registration

TDB validates the connection by running `SELECT 1` when you call
`GET /v1/sources/<id>/schema`. If the account identifier is wrong or credentials
are incorrect, you'll see HTTP 503:

```json
{"detail": "Source 'orders' (snowflake) is not accessible."}
```

Check that:

- The `account` identifier is correct and matches your Snowflake URL
- The `user` and `password` are correct
- The `database` and `schema` exist and the role has `USAGE` on both

### No warehouse configured

If the TDB service user has no default warehouse configured in Snowflake and you
omit the `warehouse` field, queries will fail with a "no active warehouse" error.
Either set a default warehouse on the Snowflake user:

```sql
ALTER USER tdb_service_user SET DEFAULT_WAREHOUSE = compute_wh;
```

Or always supply the `warehouse` field in the TDB connection config.

### Account identifier format mismatch

Snowflake is migrating from legacy account locators (`xy12345.us-east-1`) to the
new organisation-based format (`orgname-accountname`). If one format fails, try
the other. You can find both formats in your Snowflake account settings under
**Admin > Accounts**.

For accounts in the AWS `us-east-1` region, the legacy locator does not include
the region suffix — use just the account locator (e.g., `xy12345`).

### Uppercase identifier mismatches

Snowflake stores unquoted object names in UPPER CASE. If your `table` or `schema`
field uses lowercase in the connection config, Snowflake will automatically
upper-case them. To reference an object created with a quoted lowercase name
(e.g., `CREATE TABLE "orders"`), supply the exact case in the connection config.

### `permission denied` on query

If TDB can connect but queries fail with an access error, check that each level of
the permission chain is in place: WAREHOUSE usage, DATABASE usage, SCHEMA usage,
and SELECT on the table. Missing any one level will produce an access error (see
[Minimum required permissions](#minimum-required-database-permissions)).
