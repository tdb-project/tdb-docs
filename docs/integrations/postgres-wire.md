# PostgreSQL wire gateway

**Enterprise, from 0.14.0. Off unless `TDB_PG_PORT` is set.**

Point an existing PostgreSQL client at TDB instead of at the database. TDB
speaks the PostgreSQL protocol, proxies the connection to a registered
PostgreSQL source, and governs and audits every statement on the way —
the same rules REST and MCP follow.

**An existing app changes three things: host, port and password.** The
password becomes a TDB API key. The driver, the SQL, the database name and the
username stay as they are:

```text
before: postgresql://app:dbpass@db.internal:5432/sales
after:  postgresql://app:<TDB_API_KEY>@tdb.internal:5432/sales
```

The database name selects the TDB source: a source **named** `sales`, or, if
there is none, the single PostgreSQL source whose `dbname` is `sales`.

Verified unchanged with psql, psycopg 3, psycopg2 + SQLAlchemy + pandas,
pgjdbc 42.7 (and so DBeaver and most JVM tools) and Metabase.

---

## Turning it on

```bash
docker run -p 8000:8000 -p 5432:5432 \
  -e TDB_PG_PORT=5432 \
  -e TDB_PG_TLS_CERT=/certs/tdb.crt -e TDB_PG_TLS_KEY=/certs/tdb.key \
  -v /srv/tdb-certs:/certs:ro \
  ... tdb-enterprise:<version>
```

Run TDB with **one** worker process while the gateway is on. A second process
cannot bind the same port; it logs `pgwire_listen_failed` and serves REST and
MCP without the gateway.

Then create a key for the client. **It must have the `read` role**, and if it
is scoped with `allowed_tools`, the scope must include `query_source`:

```bash
curl -X POST http://localhost:8000/v1/auth/keys \
  -H "Authorization: Bearer <ADMIN_KEY>" -H "Content-Type: application/json" \
  -d '{"name": "metabase", "role": "read", "rate_limit": 600}'
```

!!! tip "Give BI tools a higher rate limit"
    Every statement that reads data counts against the key's per-minute limit
    (default 60), and that includes catalog queries. A BI tool browsing a
    schema sends dozens at once. Session statements (`SET`, `SHOW`, `BEGIN`,
    `COMMIT`) are not counted.

---

## Connecting

=== "psql"

    ```bash
    psql "host=tdb.internal port=5432 dbname=sales user=analyst sslmode=verify-full sslrootcert=tdb-ca.pem"
    # password: the TDB API key
    ```

=== "Python (SQLAlchemy / pandas)"

    ```python
    import pandas as pd, sqlalchemy as sa
    eng = sa.create_engine("postgresql+psycopg2://analyst:<KEY>@tdb.internal:5432/sales"
                           "?sslmode=verify-full&sslrootcert=tdb-ca.pem")
    pd.read_sql("SELECT customer, amount FROM orders", eng)
    ```

=== "JDBC"

    ```text
    jdbc:postgresql://tdb.internal:5432/sales?sslmode=verify-full&sslrootcert=/path/tdb-ca.pem
    user=analyst  password=<KEY>
    ```

=== "Metabase"

    Add a **PostgreSQL** database with host `tdb.internal`, port `5432`,
    database `sales`, and the TDB key as the password, and **turn on "Use a
    secure connection (SSL)"**. That toggle is off by default and Metabase then
    connects without TLS, which TDB refuses. This is the one changed setting
    for Metabase.

---

## Security model

**TLS is required, and it is checked before a password is requested.** A
client that connects without TLS is refused before it can send its key, because
libpq's default `sslmode=prefer` re-sends the password without TLS after any
failed attempt. Set `TDB_PG_TLS_CERT` and `TDB_PG_TLS_KEY` to a certificate your
clients trust, and have them use `sslmode=verify-full`. Without those settings
TDB generates a self-signed certificate beside its registry and logs
`pgwire_tls_self_signed`. Clients can still encrypt, but they cannot tell TDB from
an impostor, so a key is exposed to anyone able to intercept the connection.
`TDB_PG_ALLOW_INSECURE=true` accepts plaintext; use it only on a trusted
loopback.

**Only read-role credentials.** A DB-managed key whose role is `read`, or a JWT
whose `role` claim is `read`. Static `TDB_API_KEYS` keys are admin keys and are
refused. The wire can only read, so a higher role adds nothing there — except
the damage if a connection string leaks. The key, its role and the licence are
re-checked before **every statement**: revoking a key ends its open sessions.

**Read-only by construction.**

- Every statement is checked by the same validator as REST and MCP (see
  [what SQL is accepted](../api/query.md#what-sql-is-accepted)).
- Before each batch of client messages, TDB re-asserts read-only and the
  timeouts on the database session. A client's `BEGIN` is made `READ ONLY`.
  After `COMMIT` or `ROLLBACK`, a client must end the batch (`Sync`) before its
  next statement.
- Session statements are allowlisted:
  - **Allowed:** `SET` of `application_name`, `search_path`, `TimeZone`,
    `DateStyle`, `client_encoding`, `extra_float_digits`, `IntervalStyle`,
    `client_min_messages` and `bytea_output`; `SHOW`; isolation levels;
    `DISCARD`; `DEALLOCATE`; cursors over a valid read.
  - **Refused:** anything that could make the session writable or lift a
    limit, such as `SET default_transaction_read_only`, `SET ROLE`,
    `BEGIN READ WRITE` or `SET statement_timeout`.
  - **Dropped:** a client's startup `options`.
- A refused statement fails on the database too, so the client's transaction
  behaves exactly as PostgreSQL would: the batch is abandoned, and an open
  transaction must be rolled back.

**The source's own role must be least-privileged.** TDB checks it on every
session, as on every other path — see
[use a least-privileged role](../connectors/postgresql.md#use-a-least-privileged-role).
The source's `sslmode`, `sslrootcert` and `require_auth` apply to TDB's own
connection to the database.

---

## Limits

| Setting | Default | Effect |
|---|---|---|
| `TDB_PG_MAX_ROWS` | `TDB_MAX_ROWS` (1000) | Rows one statement may return. A larger result **ends the connection** with `54000` and is audited `row_cap_exceeded`, rather than arriving silently short. `0` = no ceiling. Rows are streamed, not buffered, so memory does not limit this. |
| `TDB_QUERY_TIMEOUT` | `30` | TDB cancels a statement at this many seconds itself, whatever `statement_timeout` the session holds. |
| `TDB_PG_MAX_CLIENTS` | `50` | Connections at once, authenticated or not. Each authenticated connection holds one session on the source database; count it against that server's `max_connections`. |

An idle transaction is ended by the database after 60 seconds, so a client
cannot hold a snapshot open and block vacuum. Cancel requests from a client
(Ctrl-C in psql, a BI tool's timeout) reach the database.

---

## Audit trail

Every statement writes two signed entries to the same hash chain as REST and
MCP:

- **`query_started`**, written *before* the statement runs. If it cannot be
  written, the statement is refused.
- **`query`** with `rows_returned` and `outcome` (`ok`, `error` with its
  `sqlstate`, or `skipped` when an earlier error abandoned the batch).

Both carry `transport: "pgwire"`, `client_user` (the startup username),
`statement_kind`, and `params`. `params` holds the bound parameter values,
each truncated to 256 characters, with binary values in hex. REST already
audits the literal values written into SQL, and this keeps the wire's audit
able to say what was asked. Refusals are `denied` entries — see
[audit reasons](../security/audit.md).

---

## Errors a client may see

| SQLSTATE | When |
|---|---|
| `28000` | No TLS; credential not read-role; licence not valid |
| `28P01` | Invalid key or token |
| `3D000` | No TDB source matches the database name |
| `42501` | Statement refused by the validator or the session allowlist; privileged source role |
| `25001` | A statement after `COMMIT`/`ROLLBACK` in the same batch |
| `53300` | `TDB_PG_MAX_CLIENTS` reached |
| `53400` | The key's rate limit |
| `54000` | Result over `TDB_PG_MAX_ROWS` (connection closed) |
| `57014` | Cancelled: `TDB_QUERY_TIMEOUT` or a client cancel |

---

## Not supported

`COPY`, the function-call protocol, replication connections, `LISTEN`/`NOTIFY`,
and server-side `PREPARE`/`EXECUTE` in SQL. Prepared statements through the
protocol, which drivers use, work. Only PostgreSQL sources are served on the
wire.
