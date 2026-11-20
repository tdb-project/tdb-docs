# ClickHouse Connector

**Enterprise, from 0.39.0.** Read-only SQL access to a ClickHouse database over its HTTP
interface: the whole database or one table, through REST, MCP, views, paging and streams,
under the same row cap, timeout, audit, approvals and policies as every other source.

!!! info "What it has run against"
    ClickHouse **26.8** (LTS), self-hosted, in TDB's CI on every change. **ClickHouse Cloud
    has not been run**: it speaks the same HTTPS interface, but until it has been tested we
    do not claim it. TDB connects over HTTP (8123) or HTTPS (8443) only, not the native
    protocol (9000).

---

## 1. Create a user that only reads

TDB refuses a ClickHouse user that can do more than read (see
[the grants check](#the-grants-check)). The user needs `SELECT` on the databases it serves
and **no settings profile**:

```sql
CREATE USER tdb_reader IDENTIFIED BY 's3cret';
GRANT SELECT ON analytics.* TO tdb_reader;
```

Narrower is fine: `GRANT SELECT ON analytics.events TO tdb_reader`, or a column list
`GRANT SELECT(id, ts, country) ON analytics.events TO tdb_reader`.

**Do not** give it a settings profile that sets `readonly`. TDB sets `readonly=1` on every
request itself, together with its own limits, and a profile that fixes `readonly` stops it
setting those limits; a profile that marks them `CHANGEABLE_IN_READONLY` lets any query
lift them. Either way TDB refuses the source (`403`, naming the setting).

## 2. Register it

```bash
curl -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer <ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "analytics",
    "source_type": "clickhouse",
    "connection": {
      "host": "clickhouse.internal",
      "port": 8443,
      "secure": true,
      "database": "analytics",
      "user": "tdb_reader",
      "password": "s3cret"
    }
  }'
```

| Key | Required | Default | Description |
|---|---|---|---|
| `host` | Yes | — | ClickHouse host |
| `port` | Yes | — | HTTP port: `8123`, or `8443` with `secure` |
| `database` | Yes | — | The database queries run in; bare table names resolve here |
| `user`, `password` | Yes | — | The reader from step 1 |
| `table` | No | — | Omit to register the whole database |
| `secure` | No | `false` | HTTPS |
| `verify` | No | `true` | Verify the server certificate. Leave it on outside a test |
| `ca_cert` | No | — | A CA bundle path inside the TDB container, for a private CA (see the warning below) |

!!! warning "HTTPS needs a publicly trusted certificate in 0.39.0"
    In 0.39.0 `ca_cert` and `verify` have no effect: TDB checks the server's certificate
    against the public certificate authorities only. A server with a certificate from your
    own CA registers, and then every query fails (`500`; the application log shows
    `CERTIFICATE_VERIFY_FAILED`). Until this is fixed, use a publicly trusted certificate,
    or plain HTTP (8123) on a network you control.

Registering connects to nothing (as for every database source). The first query checks the
user, and the schema endpoint and MCP `schema_source` list the database's tables **and
views**, with their ClickHouse types (`LowCardinality(String)`, `DateTime64(6, 'UTC')`, …).

---

## How a query is kept read-only

Three layers, each enough on its own for writes:

1. **TDB's validator** refuses anything that is not one `SELECT` or `WITH` query
   ([what SQL is accepted](../api/query.md#what-sql-is-accepted)).
2. **`readonly=1` on every request.** ClickHouse then refuses writes, `SET`, a query's own
   `SETTINGS`, and the table functions that reach files and other servers (`url()`,
   `file()`, `s3()`, `remote()`, `executable()`, …), **even for a user granted them**.
3. **The user's grants**: `SELECT` only, checked on every connection (below).

### Refused in the SQL itself

Whatever the user may do, TDB refuses these in a query on a ClickHouse source, as
`400 sql_validation_failed`:

| Refused | Why |
|---|---|
| A `SETTINGS` clause, anywhere in the query | A query must not change TDB's limits |
| A `FORMAT` clause | TDB reads the result in its own format |
| Every table function: `numbers()`, `merge()`, `view()`, `url()`, `file()`, … | `merge()` and `view()` name tables in a way a [policy](../security/policies.md) cannot see; the rest reach outside the database |
| SQL TDB cannot parse as ClickHouse | Every check above depends on reading it |

`ARRAY JOIN`, `FINAL`, `SAMPLE`, `PREWHERE`, `LIMIT … BY`, `WITH TOTALS`, lambdas and the
`-If` aggregate combinators all work, except under a policy (see [Policies](#policies)).

### The grants check

On every connection TDB reads the user's effective grants with
`SHOW GRANTS WITH IMPLICIT FINAL` (its roles' grants included) and allows only:

- `SELECT`, `SHOW …` and `dictGet` on named databases or tables;
- the `TABLE ENGINE` and `system.*` grants every ClickHouse user holds.

Anything else is refused with `403` and audited `privileged_db_role`, the message naming
the grant: `ALL` (the `default` user), `ACCESS MANAGEMENT`, any source grant (`URL`,
`FILE`, `S3`, `REMOTE`, …), `CREATE TEMPORARY TABLE`, any write, `KILL QUERY`, and
`SELECT` on `*.*` or on other `system` tables, which hold other users' queries
(`system.query_log`, `system.processes`). A grant through a role counts.
`TDB_ALLOW_PRIVILEGED_DB_ROLE=true` turns the check off; it does not turn off the settings
profile check.

---

## Limits

- **Timeout.** Each query carries `max_execution_time` = `TDB_QUERY_TIMEOUT` (streams:
  `TDB_STREAM_TIMEOUT`); ClickHouse ends it and TDB answers `504 query_timeout`.
- **Row cap.** TDB adds `LIMIT n + 1` where the query has none and reads the result only
  up to that row, whatever the query's own `LIMIT`; the rest is cut off at the socket and
  stops on the server. The same for a [stream](../api/query.md#streaming-a-whole-result)
  whose client goes away.

## Types

| ClickHouse | JSON |
|---|---|
| `(U)Int8`–`(U)Int256` | number, exact (above 2^53 a JavaScript client loses digits, as for PostgreSQL `bigint`) |
| `Float32/64` | number; `NaN` and infinities as the strings `"NaN"`, `"Infinity"` |
| `Decimal` | string, exact |
| `String`, `LowCardinality(String)`, `Enum` | string (an `Enum` as its name) |
| `FixedString` | base64 (the driver returns bytes; as binary on every engine) |
| `Date`, `Date32` | `"2026-11-20"` |
| `DateTime`, `DateTime64` | ISO 8601. In a time zone other than the server's, with its offset (`+05:30`); in the server's own zone (UTC by default), with none |
| `UUID`, `IPv4`, `IPv6` | string |
| `Array`, `Tuple` (named), `Map`, `JSON` | array or object |
| `Nullable(T)` | `T` or `null` |

## Policies

[Column and row policies](../security/policies.md) work on ClickHouse sources. Under a rule,
these clauses are refused (`403`, `policy_clause_not_supported`), because the rewrite cannot
yet carry them safely: `FINAL`, `SAMPLE`, `ARRAY JOIN`, `PREWHERE`, `LIMIT … BY`,
`SETTINGS`, `FORMAT`. ClickHouse functions outside the policy allow-list (such as
`uniqExact`) are refused under a rule, as on every engine. A caller with no rule is not
affected. The masks are `concat('***', right(toString(col), 4))` and
`lower(hex(SHA256(toString(col))))`: the hash is of ClickHouse's own text for the value, so
`Decimal` `7.50` hashes as `7.5` and does not match the same value hashed on PostgreSQL.

Views and materialized views over a protected table are named in the policy check's
warnings (`GET /v1/policies?validate=true`).

## Not supported

- **The PostgreSQL wire gateway** serves PostgreSQL sources only. Reach ClickHouse over
  REST, MCP and views.
- **Connection pooling** (`TDB_POOL_SIZE`) is PostgreSQL only; each ClickHouse query opens
  its own HTTP connection.
