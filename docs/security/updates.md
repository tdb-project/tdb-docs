# Security & Updates

How TDB Enterprise handles security issues, how you receive fixes, and how to
report a vulnerability.

---

## Reporting a vulnerability

Please report security issues **privately** — do not open a public issue.

- Email **security@tdb.asvanlabs.com**, or
- Open a private security advisory on the relevant repository.

Include the version you're running (see [Checking your version](#checking-your-version))
and steps to reproduce. We follow coordinated disclosure: advisory details are
published after a fix is available and customers have had time to update.

### Response targets

| Stage | Target |
|---|---|
| Acknowledgement | within 48 hours |
| Assessment & severity rating | within 5 business days |
| Fix released — critical / high | within 14 days |
| Fix released — medium / low | within 60 days |

## Supported versions

Security fixes ship in the **latest release**. There are no backports to older
versions — to stay supported, run the current version and update when a new one is
published. Updating is loading the new image and restarting on the same data ([Upgrading](../deployment/upgrading.md)).

## How you receive a fix

TDB is **self-hosted** — it never calls home and nothing auto-updates, so you stay
in control (including in air-gapped environments). When we publish a fix, you update
to the patched image. The steps depend on how you receive TDB:

We send the patched version as an image file, with a download link that expires after
7 days and its SHA-256 checksum. The image is generic and your licence is supplied at
runtime, so updating never touches your licence. Air-gapped sites use the same file.

```bash
sha256sum -c tdb-enterprise-vX.Y.Z.tar.gz.sha256
docker load < tdb-enterprise-vX.Y.Z.tar.gz
# start it on the same data and settings — see Upgrading
```

Trial images with the licence built in are re-issued on the patched version.
The full procedure, with backup, checks and rollback: [Upgrading](../deployment/upgrading.md).

We notify the contact on your account when an update affecting you is available,
with the exact version and steps.

## Checking your version

Confirm what you're running at any time:

```bash
# Authenticated — includes the build commit:
curl -s -H "Authorization: Bearer <your-api-key>" http://<host>:8000/v1/version
# → {"version": "X.Y.Z", "build_sha": "abc1234"}

# Unauthenticated liveness banner also carries the version:
curl -s http://<host>:8000/
```

The running version is also printed in the container logs at startup
(`tdb_startup version=… build_sha=…`).

## Where advisories are published

Once a fix is released and customers have had time to update, advisories are
published as GitHub Security Advisories and summarized here. For the Community
Edition's security policy and known design constraints, see the
[`SECURITY.md`](https://github.com/tdb-project/tdb-community/blob/main/SECURITY.md)
in the open-source repository.

### 2026-11-18 — A read could hold a lock the source's own applications wait on

**Enterprise, every release with the PostgreSQL or MySQL connector; fixed in 0.38.8.
Community 0.18.4 refuses the same functions** (its CSV sources have none of them; the check
is shared). Found by a security review; we know of no use of it.

PostgreSQL's session advisory locks (`pg_advisory_lock` and the rest) and MySQL's named
locks (`GET_LOCK`) are held by a session, not a transaction, so a rollback does not release
them. Many applications use them to serialise migrations and jobs. A key allowed to query
could take one: with `TDB_POOL_SIZE` set, the pooled connection kept the lock after the
query ended and returned to the pool, idle, where it stayed until the connection closed;
through the wire gateway it lasted the client's session; on any other connection, until
the query ended. Anything in your applications waiting on that lock waited too. No data
was exposed.

From 0.38.8 / 0.18.4 SQL naming `pg_advisory_*`, `pg_try_advisory_*`, `GET_LOCK`,
`RELEASE_LOCK` or `RELEASE_ALL_LOCKS` is refused (matched inside quotes and comments too),
and the pool releases every advisory lock when a connection comes back.

**Action:** upgrade if a PostgreSQL or MySQL source's database uses advisory or named locks.
To look back, search the audit log for those function names.

### 2026-11-18 — A revoked key's stream ran on; an audit export could stop for good

**Enterprise, 0.31.0 to 0.38.7 (streams) and 0.4.0 to 0.38.7 (export); fixed in 0.38.8.
Community is not affected:** it has neither. Found by a security review.

- **Revocation.** A stream checked its key when it started. A key revoked, or a sign-in
  removed, while its stream ran kept receiving rows until the stream ended, up to
  `TDB_STREAM_TIMEOUT` (600 s) and `TDB_MAX_RESULT_ROWS`. From 0.38.8 a stream re-checks
  its credential and the licence every few seconds and ends with `credential_invalid`.
- **Export.** An audit entry holds its statement whole, and the wire gateway accepts
  statements up to 16 MiB. A key that sent enough long statements could fill one export
  batch (100 entries) past Splunk HEC's default `max_content_length` (800 MiB). Splunk
  refused it (413), the export stops at the first refused batch so that nothing is lost,
  and every later run retried the same batch: nothing after it reached Splunk. (S3 objects
  over 5 GB fail the same way.) From 0.38.8 a batch is also cut at 32 MiB.

**Action:** upgrade. If `GET /v1/audit/export/status` shows a destination whose `last_run`
keeps failing with HTTP 413, upgrading lets the export continue from where it stopped.

### 2026-11-18 — A query could read a column held for approval without asking

**Enterprise, 0.32.0 to 0.38.6; fixed in 0.38.7. Community is not affected:** it has no
approvals. Found by a security review of approvals; we know of no use of it. Deployments
whose policy file has no `approval` block are not affected.

A rule's `approval` block holds a query that reads the columns it names until an approver
decides. TDB works out which columns a query reads, and two shapes slipped past it:

- a whole-row reference, `SELECT data FROM data` (or the table's alias), which returns
  every column of the row as one value;
- an alias column list, `SELECT * FROM data AS a(p, n, q)`, which renames the columns
  before they are matched.

Both returned the held columns at once, with no approval request. Column withholding,
masking and row filters still applied: a caller saw only what its rule shows, but saw the
held columns without anyone approving it. From 0.38.7 both shapes count as reading every
column of the table, so the query waits for approval.

Also in 0.38.7: a table or column name holding a line break (PostgreSQL allows one in a
quoted identifier) could end a comment line in the draft from `tdb policy suggest` and add
a key such as `unlisted_sources: allow` to it. Names in the draft's comments now show
control characters as `\uXXXX`. Review a draft before loading it, as always.

**Action:** upgrade if your policy file uses `approval`. To look back, search the audit log
for `query` entries under a rule with an approval block whose SQL selects the table itself
or gives the table a column list.

### 2026-11-18 — `tdb query` printed data as terminal markup

**Community 0.18.2 and earlier, enterprise 0.38.6 and earlier; fixed in community 0.18.3
and enterprise 0.38.7.** Found by a security review of the CLI. Only the `tdb query`
command is affected; the server, the API and MCP are not.

`tdb query` printed values through a library that reads `[…]` as styling and passes
escape sequences to the terminal. A value holding `[b]…[/b]` or `[x]` lost its brackets
in table and `--output csv` output, a value holding `[/b]` stopped the command, and a
value holding an escape sequence reached the terminal as it was, where it could clear the
screen or rewrite what was printed before it. Anyone who can write to the queried data
could do this to the person running the command. From 0.18.3 / 0.38.7 the table shows
values as text with control characters made visible, and `--output csv` writes the data
exactly.

**Action:** upgrade the CLI. A CSV exported with `tdb query -o csv` from an earlier
release may have lost square-bracketed text; export it again.

### 2026-11-18 — Client metadata fetches could hold the server's worker threads

**Enterprise, 0.18.0 to 0.38.5; fixed in 0.38.6. Community is not affected:** it has no
OAuth. Found by a security review of Client ID Metadata Documents; we know of no use of it.
Deployments that do not set `TDB_OAUTH_CIMD_HOSTS` (the default) are not affected.

With `TDB_OAUTH_CIMD_HOSTS` set, the authorization page fetches a client's metadata from a
URL the client names, on one of the allowed hosts. The page needs no credential, and each
fetch holds one of the server's worker threads until it ends. Two things let that go on too
long:

- the 5-second limit applied to each read, not to the fetch, so a server on an allowed host
  that sent its response a byte at a time held the thread for as long as it liked;
- nothing bounded how many fetches ran at once. Each new URL started one, so a caller
  sending many requests for an allowed host that was slow or not answering held every worker
  thread, and REST and MCP requests waited behind them. In our test, 80 such requests delayed
  `/health` by 2.5 seconds.

No data was exposed. From 0.38.6 the whole fetch ends at the 5-second deadline, at most 4
fetches run at once, and a request past that is refused at once (`invalid_client`, logged
`oauth_cimd_refused … reason=busy`). NAT64 (`64:ff9b::/96`) and IPv4-compatible (`::/96`)
IPv6 addresses are now checked by the IPv4 address they carry.

**Action:** upgrade if you set `TDB_OAUTH_CIMD_HOSTS`. To look back, search the application
log for `oauth_cimd_refused`.

### 2026-11-17 — One key could hold every stream slot

**Enterprise, 0.31.0 to 0.38.4; fixed in 0.38.5. Community is not affected:** it has no
streaming. Found by a security review of streaming; we know of no use of it. Deployments that
do not use `POST /v1/query/stream` are not affected.

`TDB_MAX_STREAMS` (4) bounds the streams one process serves, but nothing bounded them per
caller. A key allowed to query could open four streams over a large result and stop reading:
each stream then waited on its client and held its slot until `TDB_STREAM_TIMEOUT` (600 s),
so every other caller got `429 too_many_streams` for ten minutes, and four more requests
renewed it. On PostgreSQL each held stream also kept a read transaction open, which stops
vacuum. An agent that opens a stream and stops reading has the same effect, without intent.
No data was exposed.

From 0.38.5 one caller holds at most `TDB_MAX_STREAMS_PER_CALLER` (default 2) streams (`429
too_many_streams_for_caller`), and a stream whose client takes nothing for
`TDB_STREAM_STALL_TIMEOUT` (default 60 s) is cut and audited `stream_outcome: stream_stalled`.

**Action:** upgrade. To look back, search the audit log for denials with reason
`too_many_streams`.

### 2026-11-17 — One key could end other callers' PostgreSQL queries

**Enterprise, every release with the PostgreSQL connector; fixed in 0.38.4. Community 0.18.2
carries the same change (the validator is shared) but was not affected:** CSV sources have
none of these functions. Found by a security review of the database connectors; we know of
no use of it.

Every caller of a source runs as the source's one database role, and PostgreSQL lets a role
cancel or end its own sessions. Any key allowed to query a PostgreSQL source could run
`SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE usename = current_user`
through REST, MCP, a stream, a view or the wire gateway, and end every other caller's
running query on that source, gateway sessions included. It worked quoted, as a
unicode-escaped identifier, and inside `query_to_xml(…)` or `ts_stat(…)`, which run SQL given
as text. No data was exposed. A key under a column or row policy rule was already refused.
Snowflake's `SYSTEM$CANCEL_…` and `SYSTEM$ABORT_…` functions have the same shape.

From 0.38.4 the validator refuses SQL naming `pg_terminate_backend`, `pg_cancel_backend`,
`query_to_xml…`, `ts_stat`, `ts_rewrite`, `SYSTEM$CANCEL_…`, `SYSTEM$ABORT_…`, or any `U&"…"`
identifier, matched on the whole text (`400` / gateway `42501`, audited
`sql_validation_failed`).

**Action:** upgrade. On PostgreSQL, also revoke the two signal functions from `PUBLIC`
([how](../connectors/postgresql.md#stop-callers-ending-each-others-queries)): that stops the
call however it is written. To look back, search the audit log for queries naming those
functions.

### 2026-11-17 — A MySQL user holding FILE through a role passed the role check

**Enterprise, 0.12.0 to 0.38.2; fixed in 0.38.3. Community is not affected:** it has no
database connectors. Found by a security review of the database connectors; we know of no
use of it. Deployments with no MySQL source are not affected.

Since 0.12.0 TDB refuses a MySQL source whose user holds the `FILE` privilege, because
`FILE` lets a `SELECT` read files on the database server with `LOAD_FILE()` and write them
with `SELECT … INTO OUTFILE`, and a read-only transaction stops neither. The check read
only the privileges granted to the user itself. A user that holds `FILE` through a role
active at sign-in, such as a default role, passed, and every key allowed to query the
source had that reach: within `secure_file_priv`, or anywhere the server may write when it
is empty.

From 0.38.3 the check reads the session's own grants, active roles included, and refuses
such a user (`privileged_db_role`).

**Action:** upgrade. To see whether a source was exposed, sign in as its user and run
`SHOW GRANTS;`; if the output mentions `FILE` or `ALL PRIVILEGES ON *.*`, give the source a
user with `SELECT` only, and search the audit log for its queries that call `LOAD_FILE` or
`INTO OUTFILE`.

### 2026-11-17 — A wire gateway session could write; a superuser role's members passed the role check

**Enterprise; fixed in 0.38.2. Community is not affected:** it has no wire gateway and no
database connectors. Found by a security review of the gateway's session code; we know of no
use of any of them. Deployments that do not set `TDB_PG_PORT` are affected only by the
second item.

- **A gateway session could write (0.14.0 to 0.38.1).** PostgreSQL keeps a prepared
  statement when a new one is prepared under the same name and fails, and when an earlier
  error makes it skip a `Close`. TDB recorded the new statement anyway, so a `COMMIT`
  prepared under a name could later run while TDB took it for a `SELECT`. That escaped the
  rule that a batch ends at `COMMIT`. After `set_config('default_transaction_read_only',
  'off', false)` in the same transaction, the statement after it ran read-write: any
  function the source's database user may call that writes (`nextval()`, your own
  functions). The same mismatch could audit one statement as another, without a
  rate-limit charge. Tables stayed read-only to `INSERT`, `UPDATE` and `DELETE`, which
  the validator refuses.
- **A member of a superuser role passed the privileged-role check (0.12.0 to 0.38.1).** The
  check read only the login role's own flag. A source registered with a role that is a
  member of a superuser role was accepted. Through the gateway, `SELECT
  set_config('role', '<that role>', false)` made the session superuser for the
  statements after it, and `pg_read_file()` read the database server's files. REST and
  MCP refused the function call (PostgreSQL checks it before `set_config` in the same
  statement runs). Every PostgreSQL connection now refuses such a role.
- **`search_path` moved a table under a policy (0.22.0 to 0.38.1).** On the gateway, a
  key under a column or row rule could `SET search_path` to a schema the policy refuses
  and read a same-named table there through a bare table name. The rule's columns and
  masks still applied, to the other table.

From 0.38.2 the gateway names statements and portals on the database itself and refuses a
name it never recorded (`unknown_statement`, `unknown_portal`). While a policy file is
loaded it refuses `SET search_path` (`policy_search_path`; `RESET` is allowed). A source
whose role is a member of a superuser role is refused (`privileged_db_role`).

**Action:** upgrade. If a source's database role is a member of a superuser role, sessions
on it now fail: grant it only what it reads. To look back, search the audit log for gateway
statements (`transport: pgwire`) that call `set_config`, and check the sequences and any
writing functions the source's role may use.

### 2026-11-16 — A parenthesised join read past a column or row policy

**Enterprise, 0.22.0 to 0.38.0; fixed in 0.38.1. Community is not affected:** it has no
policies. Found by a security review of the policy rewrite; we know of no use of it.

- **A key under a rule could read a table its rule names in full.** Written with an alias,
  `SELECT * FROM (customers CROSS JOIN (SELECT 1) z) AS j` was not rewritten: it returned
  every row, outside the row filter, and every column, withheld and masked ones
  included, on REST, MCP, views and the PostgreSQL wire gateway. Shown on CSV, Parquet,
  DuckDB and PostgreSQL sources; TDB passed it for MySQL, SQL Server and Snowflake too
  (MySQL's own parser rejects the syntax). Tables the rule does not name stayed refused.
- **PIVOT and UNPIVOT on a table ran before the policy** (DuckDB, CSV, Parquet, Snowflake,
  SQL Server sources): a withheld column's values came back under a visible column's name.
  The row filter still held.
- **Through the wire gateway, a key under a rule could read `pg_stat_activity`**, which
  shows the SQL other sessions are running. Every TDB session on a source uses the
  source's database user, so that was other callers' queries and the values in them.

From 0.38.1 these shapes are refused (`policy_unclassified_table`), and the gateway's
catalog queries read only an allow-list of `pg_catalog` relations; the others are
`policy_catalog_refused`.

**Action:** upgrade. To look back, search the audit log for queries by keys under a rule
(`policy_rule`) whose `sql` holds a `(` straight after `FROM` or `JOIN` and a `) AS` or
`) <alias>` closing it, `PIVOT`, `UNPIVOT`, or `pg_stat`.

### 2026-11-16 — Password login unlimited; listings past a policy; one MCP call held the server

**Enterprise, fixed in 0.38.0; the MCP part also community, fixed in 0.18.1.** From the
same security review of the code as 0.37.1; we know of no use of any of them.

- **The administrator password login had no limit and no record** (every release up to
  0.37.1). `POST /v1/auth/token` and the OAuth sign-in form cost a wrong guess 0.2 seconds
  and nothing else, and neither a failed nor a successful admin sign-in was in the audit
  log. From 0.38.0, ten failed attempts in a minute across the deployment close both until
  the minute ends (`429`, `login_throttled`), and every attempt is audited (`login`,
  `invalid_credentials`). The limit is per deployment, so someone who can reach the login
  can hold it closed; API keys and SSO are unaffected. The sign-in form now names where
  the access goes and cannot be shown inside another site's page.
- **With a [policy file](policies.md), listings showed what the rule refuses** (0.22.0 to
  0.37.1). `GET /v1/sources` and `GET /v1/sources/{id}` returned every source's name and
  connection details (host, database user, file path; never passwords), the view lists
  returned every view with its description and parameters, and MCP "not found" errors named every source. From
  0.38.0 a key under a rule is told only about the sources it may query and the views on
  them; a refused source's record is `403` (`source_read`).
- **One slow MCP call held every other request** (both editions, every release). The MCP
  endpoint checked the key and ran the query on the server's event loop, so while one ran,
  nothing else was answered, health checks included, for up to `TDB_QUERY_TIMEOUT`
  (enterprise; community has no timeout). From enterprise 0.38.0 and community 0.18.1 it
  runs on a worker thread.

**Action:** upgrade. If the password login has been reachable from an untrusted network,
consider changing `TDB_ADMIN_PASSWORD`; earlier releases left no record of attempts. To see
attempts from now on, search the audit log for `"action": "login"`.

### 2026-11-16 — A source record carried the key that registered it

**Enterprise, every release up to 0.37.0; fixed in 0.37.1. Community is not affected:** it
has always stored a six-character hint.

When a source was registered, TDB stored the registrant's whole credential (a database
key, a `TDB_API_KEYS` key or a sign-in token) in the source's `registered_by` field, in
plain text in the registry. `GET /v1/sources/{id}` and the registration response returned
that field to any authenticated caller, so **a `read` key could read the key that
registered a source and use it**: an admin key if an admin registered it. Found by a
security review of the code; we know of no use of it.

From 0.37.1 the field holds only the hint the audit log records (`tdbk_a...`), and at the
first start on 0.37.1 the stored values of existing sources are cut to that hint.

**Action:** upgrade, then **rotate every key that has registered a source**
([key rotation](../auth/api-keys.md)), and replace any `TDB_API_KEYS` value that has. The
old values are also in any backup of the registry taken before the upgrade: treat those
backups as holding live keys until the keys are rotated. To look back, search the audit
log for requests you do not recognise made with those keys: each entry carries the key's
six-character hint.

The same release closes a second, smaller gap, for deployments with a
[policy file](policies.md) (0.22.0 to 0.37.0): `GET /v1/sources/{ref}/annotations` did
not apply the caller's rule. A key under a rule could read the **names and descriptions**
(never values) of columns the rule withholds, and of a source the policy refuses. From
0.37.1 a key under a rule reads only the annotations on what its rule shows, is refused a
source the policy refuses, and cannot write or delete annotations
(`policy_annotation_change`); admin keys are unchanged.

### 2026-11-10 — MCP tools named tables and columns outside a key's policy

**Enterprise 0.22.0 to 0.36.0, with a [policy file](policies.md); fixed in 0.36.1.
Community is not affected:** it has neither policies nor these tools.

When `preview_source`, `filter_source` or `aggregate_source` was given a table or column
it could not find, its error listed the whole database: every table, including those
outside the caller's rule, and every column of the table, including those the rule
withholds. A key under a policy could learn those **names** (never values) by asking for
a name that does not exist; `schema_source` and the REST schema already hid them. Found by
running policies against a real Snowflake account, where the tools also refused a table
or column name not written in Snowflake's upper case.

From 0.36.1 those tools check names against what the caller's rule shows, in any case,
and an error lists nothing else. A table outside the rule is refused
(`policy_table_not_permitted`) whether or not it exists, as SQL is.

**Action:** upgrade if a policy file hides tables or columns whose names should not be
known to the keys it restricts. To look back, search the audit log for `invalid_arguments`
entries from those keys on `mcp_query`.

### 2026-11-07 — A SQL Server driver could name a library to load

**Enterprise 0.34.0 and earlier; fixed in 0.35.0. Community is not affected:** it has no
SQL Server connector.

A SQL Server source's `driver` went into the ODBC connection string as written, and the ODBC
driver manager loads a library from a path given there. So a key allowed to register sources
(`readwrite` or `admin`) could make TDB load any shared library on its filesystem, which is
code execution where that caller can also put a file on the host, such as a tenant whose data
directory it can write. Reproduced on the 0.34.0 image. The other connection values were not
escaped either, so a `;` in a password broke the login and could add connection attributes.

From 0.35.0 `driver` must be an installed driver's name, such as `ODBC Driver 18 for SQL
Server`: anything else is refused at registration (400, audited `invalid_connection`) and a
stored source holding one fails before anything is loaded. Every other value is escaped.

Found in the same review, and fixed in 0.35.0: a PostgreSQL query opening with a comment read
its whole result into TDB's memory before the row cap applied.

**Action:** upgrade. Then list your sources and check each SQL Server `driver` is a driver
name; re-register any that is not. To look back, search the audit log for `register` entries
by keys you do not fully trust.

### 2026-11-06 — One reading of the Authorization header

**Enterprise 0.33.0. A hardening change, not a vulnerability report: no data was exposed.**

Two parts of TDB read the `Authorization` header in different ways. Authentication took the
first header and the bearer scheme in any case. The audit log's identity took the last header,
and MCP and the policy caller lookup read only `Bearer `. So a request with two headers could be
authenticated as one credential and recorded as another, and a lowercase `bearer` was refused
on MCP. From 0.33.0 every reader uses authentication's reading, and **a request with two
`Authorization` headers is refused with 400**. The same review closed two holes in the
unreleased tenant code before 0.33.0 shipped.

**Action:** none, unless a client sends the header twice; it now gets 400.

### 2026-11-05 — SQL injection through filter_source

**Enterprise 0.30.0 and earlier; fixed in 0.31.0. Community is not affected:** it has no
`filter_source`.

`filter_source` escaped its `value` by doubling `'` only. On MySQL and Snowflake a backslash
is also an escape, so a value holding `\'` ended the string and the rest ran as SQL. A
key limited to `filter_source` could read any table the source's database user could.
A policy file's rewrite still applied to the injected SQL. On MySQL it ran inside TDB's
read-only session, so it could not write. **Snowflake has no such session, and this SQL
did not pass through TDB's read-only check, so there only the role you registered limits
it:** check that it holds nothing beyond `SELECT`. Reproduced on a real MySQL. Snowflake
came from reading its rules; on 2026-11-10 the fixed `filter_source` was run against a
real Snowflake account with both quoting attacks, and each returned no rows. PostgreSQL,
SQL Server, CSV, Parquet and DuckDB treat a backslash as an ordinary character by default
and were not exposed this way.

Found in the same pass, and fixed in 0.31.0:

- **`standard_conforming_strings=off` on a PostgreSQL source** let `filter_source` values and
  view parameters break out of their strings the same way. TDB now pins it on for its own
  sessions. On MySQL it now removes `NO_BACKSLASH_ESCAPES` for the same reason.
- **`PUT /v1/auth/keys/{id}/tools` with a missing or misspelt field** answered 200 and removed
  the key's tool limits. It is now a 422. So is `.../rate-limit`.

**Action:** upgrade if you have MySQL or Snowflake sources, or a PostgreSQL role or database
with `standard_conforming_strings` off, and keys you do not fully trust. To look back, search
the audit log's `filter_source` entries for a `sql` holding a backslash. Then list your keys
and check that each `allowed_tools` is what you set.

### 2026-11-04 — Audit and policy hardening

**Enterprise 0.27.0 to 0.30.0, community 0.16.0 to 0.17.1.** Found by two independent reviews and the
testing that followed. None has an advisory: each closes a gap in evidence or a default, not a way in.

- **Values in SQL can be kept out of the logs (0.27.0 / 0.16.0).** A statement is stored as written, so
  a value in it sits in the log. `TDB_AUDIT_SQL=redacted` replaces values with `?`. It is opt-in. New
  audit files are also created `0600`.
- **A query that fails is now audited (0.28.0 / 0.17.0).** One the engine rejected, or that hit a
  missing file or an unexpected error, reached only the application log. In enterprise, MCP tools and
  view runs had also left timeouts and busy sources unrecorded. A failure under a column or row
  policy names the rule from 0.29.0.
- **A policy file refuses sources it does not name (0.30.0).** They were open to every key, as
  documented, so a typo or a new source silently removed every rule.

**Action:** upgrade. **If you use `TDB_POLICIES_FILE`, list every source first,** or set
`unlisted_sources: allow`. Before 0.30.0, check that every source you meant to restrict is named in
the file, with exactly its registered name. Before 0.28.0, the audit log has no entry for a failed
query, so look in the application log for `query_error` and `query_execution_error`.

### 2026-11-04 — SQL could read other files in the data directory

**Fixed in community 0.15.0 and enterprise 0.26.0.** Found by an independent review. Advisory: [GHSA-c2ww-8m4q-8q58](https://github.com/tdb-project/tdb-community/security/advisories/GHSA-c2ww-8m4q-8q58).

SQL on a CSV or Parquet source could read any other file in the source's data directory.
`read_csv` and `read_text` returned contents, and `glob` listed names, on REST and MCP. Nothing
outside the directory was reachable.

**Who was affected:** any deployment with other files in the data directory, on community 0.7.0 to
0.14.0 and enterprise 0.11.0 to 0.25.0. Before those, the wider issue in the 2026-09-28 entry applied.

- **Enterprise:** a key for one source could read another source's file, whatever its role,
  including a `read` key.
- **Community:** the one key could already register another file in that directory instead.
  This also read files that are not CSVs, and got around the one-source limit.

**How to check:** search the audit log for `read_csv`, `read_text`, `read_blob` or `glob` in the
`sql` field of successful queries on versions before the fix.

**Action:** upgrade. If a key that should not see a file in that directory could have read it,
review the audit log, and treat secrets in those files as exposed.

**On upgrade:** SQL that read another file now returns 403. Each file has its own engine, limited
to 4 threads in enterprise, so a single large query can be slower:
[Performance](../deployment/performance.md).

### 2026-11-04 — A built-in API key worked when none was set

**Fixed in community 0.14.0 and enterprise 0.25.0.** Found by an independent review. Advisory: [GHSA-8mfc-29h8-v8rm](https://github.com/tdb-project/tdb-community/security/advisories/GHSA-8mfc-29h8-v8rm).

With `TDB_API_KEYS` unset, TDB fell back to `dev-insecure-key-change-me`, which is published in
its own source and in these docs. Our docs also told Docker Desktop users to use it.

**Who was affected:** any deployment started without `TDB_API_KEYS`, on every earlier release.
The community Compose file used the same default and published the port on every interface.

- **Enterprise:** static keys are always admin, so anyone who could reach the port had admin
  access: all sources, keys and audit. It printed no warning.
- **Community:** anyone who could reach the port could query and replace the source. It printed a
  warning at startup.

**How to check:** `curl -H "Authorization: Bearer dev-insecure-key-change-me" http://<host>:8000/v1/sources`.
A 200 means you are exposed. The audit log's `key_hint` for that key starts with `dev-in`.

**Action:** upgrade, and set a generated `TDB_API_KEYS`. TDB now refuses to start with an example
value, or when unset unless another login exists (enterprise). If the port was reachable by
anyone you do not trust, review the audit log, and rotate your API keys and the credentials of your
registered sources.

### 2026-11-04 — filelock advisories closed

Enterprise 0.23.1 moves `filelock` to 4.2.7, which closes the advisories image scans
reported against it (GHSA-8mx5-7m5m-r77g, GHSA-cwqr-hh33-gh44, GHSA-rfpc-qcw5-3pc4,
GHSA-69hj-f68j-2879, GHSA-j8f7-rjxc-mr56, GHSA-fq56-3cx3-fjxp and earlier ones).

**TDB was not known to be exposed.** Only the Snowflake connector uses `filelock`, for
caches in a private directory, in a container that runs as one user.

**Action:** upgrade, so your image scans stop reporting them. If you use Snowflake, try it
on the new version first: the connector's own limit on `filelock` is overridden. (That
pairing has since run against a real Snowflake account, on 0.35.0, on 2026-11-10.)

### 2026-10-29 — A rotated API key could call every MCP tool

**Fixed in enterprise 0.21.2.** Community is not affected (it has no key management).
Rotating a DB-managed key (`POST /v1/auth/keys/{id}/rotate`) issued a replacement with
the same name, expiry, rate limit and role, but **without its `allowed_tools` list or
its Lean Mode setting**. A key with no `allowed_tools` may call every tool, so a key
limited to, for example, `query_source` came back from rotation able to call all seven
MCP tools. Every enterprise release from per-key tool scoping up to 0.21.1
is affected.

This matters most if you followed the 2026-10-27 advice below and rotated keys after
upgrading to 0.20.3.

**Action:**

1. Upgrade to 0.21.2.
2. List your keys (`GET /v1/auth/keys`). For any key that should be limited and shows
   `"allowed_tools": null`, set its tools again (`PUT /v1/auth/keys/{id}/tools`), and its
   Lean Mode default if it had one (`PUT /v1/auth/keys/{id}/lean-mode`).

Roles were always carried over; no key gained a role it did not have.

### 2026-10-27 — Enterprise wrote API keys to the application log

**Fixed in enterprise 0.20.3.** Community is not affected. TDB Enterprise wrote the
caller's **full API key** into its plain-text application log (the container's
standard output) as `by=…` on every successful query, source registration and
deletion, refused, failed or timed-out query, and view run, and on a prompt-injection
refusal over MCP. Anyone able to read those logs, or a log store they are shipped to,
could use the keys. Every enterprise release before 0.20.3 is affected.

Log lines now carry only the first six characters, as the audit log always has. The
signed audit log never held a full key.

**Action:**

1. Upgrade to 0.20.3.
2. **Rotate API keys** (`POST /v1/auth/keys/{id}/rotate`, or a new value in
   `TDB_API_KEYS`) unless everyone with access to TDB's application logs, including
   any log store they are forwarded to, is trusted with every key.
3. **Purge retained application logs** from before the upgrade, or restrict them as
   you would a secrets store.

JWTs and OAuth tokens were not written: for those, the log carried the token's
subject.

### 2026-10-27 — Log lines could be forged through caller input

**Fixed in enterprise 0.20.2 and community 0.12.3.** TDB writes source names, SQL
fragments and database error messages into its plain-text application log (the
container's standard output). A value containing a line break was written as is, so
a caller with query access could make text appear as a separate log line, for
example one that looks like an unrelated event. Database errors also spanned several
lines on their own.

Line breaks in everything TDB logs are now escaped (`\n`, `\r`), so each log call is
one line. Ordinary values log exactly as before, and tracebacks are unchanged.

- **The signed audit log was never affected.** It is JSON, one entry per line, and
  hash-chained; this concerns only the application log.
- Medium severity: it needs a valid API key, and it misleads a reader of the log
  rather than exposing data.

**Action:** upgrade. If you alert on, or parse, the application log, a log line
can no longer be split by caller input; nothing needs changing on your side.

### 2026-10-26 — certifi trusted CAs with compromised keys

**Fixed in enterprise 0.19.1 and community 0.12.1.** `certifi`, the CA bundle
Python's HTTPS clients verify against, trusted emSign root CAs whose intermediate
CA keys were compromised (GHSA-6q48-6c7w-9gw2, critical). A certificate issued
from those keys passed verification for any hostname.

- **Enterprise was exposed** on the three outbound connections that use certifi's
  bundle: the **Splunk** exporter, the **S3** exporter and the **Snowflake**
  connector. An attacker already on the network path, holding such a certificate,
  could have read or altered that traffic, including exported audit entries and
  Snowflake credentials and results. The CIMD fetch and the PostgreSQL wire
  gateway use the system trust store and were not affected.
- **Community's server makes no outbound TLS connection.** Only the `tdb` CLI,
  pointed at a TDB server over HTTPS, used the bundle.

Enterprise 0.19.1 also reports three `filelock` advisories in image scans
(GHSA-v4x8-3wr5-3wq8, GHSA-9vhv-jxww-xfm5, GHSA-567q-5qvw-pf92). **TDB is not
exposed**: they cover locks TDB does not use and Windows-only behaviour, and the
fixed `filelock` 4.x cannot be installed alongside the Snowflake connector yet.
Enterprise 0.23.1 moves to `filelock` 4.2.7, which closes these and the later
`filelock` advisories.

**Action:** upgrade. If you export to Splunk or S3, or query Snowflake, over a
network you do not control, treat the credentials those connections carry as
worth rotating.

### 2026-10-10 — Enterprise 0.16.0 and 0.17.0: two hardening fixes

**Fixed in enterprise 0.16.0 and 0.17.0.** Community is not affected.

- **`TDB_MAX_ROWS` now bounds the MCP tools and view runs (0.16.0).**
  `preview_source`, `filter_source`, `aggregate_source`, `run_view` and
  `POST /v1/views/{name}/run` capped `limit` at a fixed 1,000, whatever
  `TDB_MAX_ROWS` said. An operator who lowered the ceiling to limit how much
  data one call can return still got up to 1,000 rows from these paths. A
  `limit` above the ceiling is now refused and audited `limit_exceeds_max`, as
  on `POST /v1/query`.
- **The prompt-injection output filter fails closed (0.17.0).** If the filter
  itself raised an error, the tool result was returned unfiltered. It is now a
  tool error, and the rows are withheld. No input that triggers such an error is
  known; this closes the gap rather than a demonstrated bypass.

**Action:** upgrade. If you lowered `TDB_MAX_ROWS`, check the audit log for MCP
`query` entries whose `rows_returned` exceeded it before the upgrade.

### 2026-10-09 — pip removed from the images

**Community 0.9.1 and enterprise 0.14.1.** The `python:3.12-slim` base image
ships pip 25.0.1, which image scanners report for six CVEs, among them
CVE-2026-13346 (GHSA-qwm4-qh6w-59xr). **TDB was not exposed**: it never runs
pip, because its dependencies are installed with uv at build time. Rather than
upgrade a tool nothing uses, the images no longer contain pip, including
`ensurepip`'s bundled copy. A Trivy scan of the rebuilt images reports no
Python-package findings.

**Action:** upgrade if your image scanner reports pip in a TDB image. Nothing
else changes.

### 2026-10-09 — The read-only check could be shown one statement while the engine ran several

**Fixed in community 0.7.2 and enterprise 0.12.0. Affects every earlier
release.** Advisory
[GHSA-qmj5-8fv9-h74r](https://github.com/tdb-project/tdb-community/security/advisories/GHSA-qmj5-8fv9-h74r).
The SQL validator decided where string literals and comments end
the ANSI way only, while the engines TDB runs on also read dollar-quoted
strings, backslash escapes and nested or engine-specific comments. Quoted
carefully, SQL could therefore carry a second statement past the
one-statement rule introduced in 0.7.0 / 0.11.0, and the CSV engine executed
it. The engine lock from 0.7.0 / 0.11.0 still confined file access to the data
directory, but within that directory a second statement could write files —
including the registered CSV. Exploiting it requires a valid API key or token.

The fixed releases check the SQL under each supported engine's reading of it,
and the CSV connector runs a query only if DuckDB's own parser reads exactly
one `SELECT`. Refusals are 400, audited as `sql_validation_failed`.

**Action:** upgrade. If you ran an earlier release with keys held by people or
agents you would not give write access to the data directory, check that the
CSV files there are the ones you expect.

### 2026-10-09 — Enterprise 0.12.0: four further hardening fixes

**Fixed in enterprise 0.12.0.** Found in the same review:

- **`allowed_tools` now applies on REST.** A key scoped to MCP tools could
  still use the equivalent REST routes — notably `POST /v1/query` — so a key
  meant only to run views could send its own SQL. See
  [RBAC → Restricting tool access](rbac.md#restricting-tool-access).
- **Privileged database roles are refused.** A source registered as a
  PostgreSQL superuser, a MySQL user with `FILE`, or a SQL Server
  `sysadmin`/`bulkadmin` login let any key allowed to query it read files on
  the database host, because a read-only session does not stop those
  functions. Such sources now return 403 (`privileged_db_role`) until they use
  a least-privileged role, or `TDB_ALLOW_PRIVILEGED_DB_ROLE=true` is set. See
  [use a least-privileged role](../connectors/postgresql.md#use-a-least-privileged-role).
- **PostgreSQL passwords are never sent in cleartext to an unverified
  server.** TDB now sets `require_auth=!password` unless the source uses
  `sslmode=verify-full`, and accepts `sslmode` / `sslrootcert` keys. See
  [SSL connections](../connectors/postgresql.md#ssl-connections).
- **View string parameters are escaped for MySQL and Snowflake**, whose
  literals honour backslash escapes. See [views](../api/views.md).

**Action:** upgrade; then re-register any database source that used a
superuser, `root` or `sa` with a `SELECT`-only role, and review keys whose
`allowed_tools` you relied on — check the audit log for `POST /v1/query`
entries from them.

### 2026-09-28 — SQL on a CSV source could read files outside the data directory

**Fixed in community 0.7.0 and enterprise 0.11.0. Affects every earlier
release.** DuckDB resolves file paths written *inside* a query — for example
`read_csv('/etc/passwd')` — and `TDB_ALLOWED_DATA_DIR` only ever checked the
path a source was registered with. A caller able to query a CSV source (in
enterprise, any role, `read` included) could therefore read any file the server
process could read. Exploiting it requires a valid API key or token.

The fixed releases confine SQL to the source's data directory, disable
extension loading and lock the engine's settings. A refused read returns 403
and is audited as `sql_file_access`. They also refuse a query containing more
than one statement, and in enterprise a key created without a `role` is now
`read` rather than `admin`.

**Action:** upgrade. If you ran an earlier release with keys held by people or
agents you would not give shell-level file read to, review the audit log for
`read_csv`, `read_text` or other file paths in `sql`, and rotate any secret
stored in a file the server process could read.

