# Errors & audit reasons

Every refusal TDB makes, in one place: what the caller sees, what the audit log records,
and what to do about it. Search this page for the code or the `reason` you are
looking at.

Each refusal writes a signed `denied` entry to the audit log with an `action` (what
was attempted) and a `reason` (why it was refused). See [Audit log](../security/audit.md)
for the entry format.

---

## REST status codes

| Status | Audit `reason` | What it means | What to do |
|---|---|---|---|
| 400 | `sql_validation_failed` | The SQL is not a single read-only `SELECT` (or `WITH … SELECT`) | Remove the write or the second statement. See [Query](../api/query.md) |
| 400 | — | `Query failed: …` — the source rejected the SQL: unknown column or table, syntax error, a value that does not convert (from community 0.12.4 / enterprise 0.20.3) | Fix the SQL; the source's own message follows |
| 400 | `limit_exceeds_max` | `limit` is above the deployment's row ceiling (`TDB_MAX_ROWS`, enterprise) | Lower `limit`, aggregate in SQL, or raise `TDB_MAX_ROWS` |
| 400 | `invalid_connection` | Registration: the `connection` block is not valid for the `source_type` | Check the connector page for the required fields |
| 400 | `unsupported_source_type` | Registration: the `source_type` is not available in this edition | Use `csv` (community) or a commercial connector |
| 401 | `missing_credentials` | No `Authorization` header | Send `Authorization: Bearer <key>` |
| 401 | `invalid_api_key` | The key is not recognised, or has been revoked | Check the key; rotate if lost |
| 401 | `invalid_or_expired_token` | A JWT failed its signature or has expired | Issue a fresh token |
| 403 | `insufficient_role_required_<role>_have_<role>` | The key's role is below what the endpoint needs (enterprise) | Use a key with the role named in the reason. See [RBAC](../security/rbac.md) |
| 403 | `tool_not_permitted_<tool>` | The key's `allowed_tools` does not include the tool this endpoint uses (enterprise) | Add the tool to the key, or use another key |
| 403 | `sql_file_access` | The SQL named a file outside the source's data directory (CSV, Parquet) | Query the registered table (`data`) only |
| 403 | `path_outside_allowed_dir` | The registered file resolves outside `TDB_ALLOWED_DATA_DIR` | Move the file under the data directory |
| 403 | `privileged_db_role` | The source connects as a database role that can reach the server's files or programs (enterprise, from 0.12.0) | Register it with a least-privileged role. See [PostgreSQL](../connectors/postgresql.md#use-a-least-privileged-role) |
| 403 | `invalid_origin` | A request from a browser page on another origin | Add the origin to `TDB_CORS_ORIGINS` (enterprise) |
| 403 | — | `license_expired` body: the licence is expired or invalid (enterprise) | Contact us for a renewed licence |
| 404 | `source_not_found` | No source with that name or UUID | `GET /v1/sources` lists them |
| 409 | `registry_conflict` | A source with that name already exists | Pick another name, or delete the old source |
| 409 | — | An audit export to the same destination is already running (enterprise) | Wait for it, or treat 409 as "still running" |
| 422 | — | The request body failed validation (wrong type, value out of range; in community a `limit` over 1,000) | Read the `detail` array; it names the field |
| 429 | `rate_limit_exceeded` | The key's per-minute budget is used up (DB-managed keys) | Retry after the `X-RateLimit-Reset` time |
| 500 | — | TDB or the source failed: unreachable source, out of memory. Not the caller's SQL | Check the source and the application log |
| 503 | `file_unreadable` | The file behind a CSV or Parquet source is missing or unreadable | Restore the file or fix its permissions |
| 503 | — | `TDB_JWT_SECRET` is not set and a JWT was presented (enterprise) | Set it, or use API keys |
| 504 | `query_timeout` | The query ran longer than `TDB_QUERY_TIMEOUT` and was cancelled at the source (enterprise database sources) | Narrow the query, add an index, or raise the timeout |

---

## MCP

An MCP call can fail in three places.

**At the transport** (HTTP status, before the tool runs): 401 for a missing or invalid
key (audited `missing_api_key` or `invalid_api_key`), 403 for a foreign `Origin`
(`invalid_origin`), 429 for the rate limit (`rate_limit_exceeded`), and 400 or 404 for
protocol problems. The JSON-RPC codes are listed in
[MCP: error codes](../api/mcp.md#error-codes).

**As a tool error** (HTTP 200, `isError: true`, a readable message in `content`): the
same refusals as REST — `sql_validation_failed`, `sql_file_access`,
`source_not_found`, `limit_exceeds_max`, `privileged_db_role`, `query_timeout` — plus:

| Audit `reason` | What it means | What to do |
|---|---|---|
| `prompt_injection_detected` | The input filter found instruction-like text in the SQL (enterprise) | Remove the embedded instructions |
| `tool_not_permitted_<tool>` | The key's `allowed_tools` does not include this tool (enterprise) | Add the tool to the key |
| — | A Lean Mode value is invalid, e.g. `max_cell_chars: 10` | Use a value the message lists |

**As a redaction** (the call succeeds): a cell the output filter flags is replaced in
the result, and the redaction is counted in the server log (enterprise).

---

## PostgreSQL wire gateway

The gateway answers with standard PostgreSQL error codes (`SQLSTATE`), which every
client shows. Enterprise only.

| SQLSTATE | Audit `reason` | What it means | What to do |
|---|---|---|---|
| `28000` | `wire_requires_read_role` | The key or token is not `read`-role | Use a `read`-role key as the password |
| `28000` | — | The client did not use TLS, or the licence is not valid | Connect with `sslmode=require` or `verify-full` |
| `28P01` | `missing_credentials`, `invalid_api_key`, `invalid_or_expired_token` | No password, or it is not a valid key or token | Check the key |
| `3D000` | — | No TDB source matches the database name | Use a registered PostgreSQL source's name as the database |
| `42501` | `sql_validation_failed`, `privileged_db_role` | A statement was refused, or the source's role is privileged | As for REST |
| `25001` | `statement_after_transaction_end` | A statement followed `COMMIT`/`ROLLBACK` in the same batch | Send it in a new batch |
| `53300` | — | `TDB_PG_MAX_CLIENTS` connections are already open | Close idle clients or raise the limit |
| `53400` | `rate_limit_exceeded` | The key's rate limit | Slow down, or raise the key's limit |
| `54000` | `row_cap_exceeded` | A result went over `TDB_PG_MAX_ROWS`; the connection is closed | Aggregate, or add `LIMIT` |
| `57014` | `query_timeout` | Cancelled by `TDB_QUERY_TIMEOUT` or by the client | Narrow the query or raise the timeout |
| `58000` | — | TDB could not write the audit entry, so the statement was refused (fails closed) | Check the audit log volume: space and permissions |

More on the gateway: [PostgreSQL wire gateway](../integrations/postgres-wire.md).

---

## Column and row policies

Enterprise Mid-Market, from 0.22.0, while `TDB_POLICIES_FILE` is set. A refusal is
**403** `Refused by policy: …` on REST, a **tool error** on MCP and **`42501`** on the
wire gateway, audited with the reason below. Under a rule the entry also carries
`policy_rule`. See [Column and row policies](../security/policies.md).

| Audit `reason` | What it means | What to do |
|---|---|---|
| `policy_no_rule` | No rule names this key or its role, and the source's `default` is `deny` | Add a rule for the key, or use a key that has one |
| `policy_missing_attribute` | The key's rule has a row filter needing an attribute the caller lacks | Set it: `PUT /v1/auth/keys/{id}/attributes`, or the JWT `attributes` claim |
| `policy_table_not_permitted` | The SQL reads a table, view or copy the caller's rule does not name | Query only the rule's tables, or add the table to the rule |
| `policy_function_not_allowed` | A function outside the allow-list for policied callers (functions that run SQL text or read files, among others) | Rewrite without it; ask us if an everyday function is missing |
| `policy_unclassified_table` | A table expression TDB cannot place (a table function, a name passed as a string) | Name the table directly |
| `policy_unparseable` | TDB could not parse the SQL, or it holds more than one statement | Simplify the SQL; send one statement |
| `policy_not_a_query` | Not a `SELECT`, or a `SELECT INTO` / locking clause | Send a plain `SELECT` |
| `policy_no_visible_columns` | The rule leaves no column of a table visible | Fix the rule |
| `policy_schema_unavailable` | TDB could not read the table's columns, so it cannot build the rewrite (fails closed) | Check the source is reachable |
| `policy_unknown_caller` | TDB could not tell who is asking | Authenticate with an API key or JWT |
| `policy_unsupported_source` | The source type has no policy support | Remove the source from the policy file |
| `policy_catalog_refused` | Gateway: a catalog that holds data (`pg_stats` and similar) | Not available to policied keys |
| `policy_registry_change` | A non-admin key a policy restricts tried to register or delete a source | Use an admin key |

A **SQL error** under a rule (`400 Query failed: …`) shows only the reason, such as
`column "diagnosis" does not exist`. A withheld column reads exactly like one that is
not there.

**At startup**, TDB refuses to start, naming the problem, when the policy file is
missing, not valid YAML or has an invalid field, or when the licence's tier is `smb`
(`policies_tier_refused` in the application log).

---

## Not refusals

- **`truncated: true`** — not an error. More rows existed than the row ceiling or your
  `limit` allowed. Aggregate, filter, or ask for fewer columns.
- **`trimmed_cells`** in a Lean result — long text was cut because you asked for
  `max_cell_chars`; each cut is marked `…[+N chars]`.
