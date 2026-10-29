# Architecture & security model

This page is for the engineer who has to vouch for TDB: what runs, what a request
passes through, what each layer guarantees, and what TDB does **not** protect you
from. Every claim links to the page that documents it in full.

---

## What you deploy

One container running one process. It serves:

| Surface | Path / port | Edition |
|---|---|---|
| REST: sources, query, views | `/v1/...` | Both (views: enterprise) |
| MCP for AI tools | `POST /v1/mcp` | Both (community: `query_source` only) |
| PostgreSQL wire gateway | `TDB_PG_PORT` (opt-in) | Enterprise |
| Health, version, metrics | `/health`, `/`, `/metrics` | Both (`/metrics`: enterprise) |
| OpenAPI | `/docs` | Both |

State is two files on a mounted volume: the **registry** (SQLite: sources, and in
enterprise API keys, annotations and export cursors) and the **audit log** (NDJSON).
There is no database server to run beside it. Your data is never copied: TDB sends
each query to the source and streams back at most the row ceiling.

TDB serves plain HTTP. Put it behind a TLS-terminating proxy for anything beyond
localhost — see [TLS](deployment/it-admin-guide.md#7-tls-ssl).

```
 AI agents (MCP) ─┐
 Scripts (REST) ──┼──▶  TDB  ──read-only session──▶  PostgreSQL · MySQL · SQL Server
 BI tools (pgwire)┘      │                           Snowflake · CSV · Parquet
                         └──▶ audit log (hash-chained, on your volume) ──▶ Splunk / S3
```

---

## The path of a request

Every step that refuses writes a signed `denied` entry to the audit log, so a
refusal is evidence, not silence.

| # | Step | What it guarantees | Docs |
|---|---|---|---|
| 1 | **Origin check** | A request carrying a foreign browser `Origin` is refused (403, `invalid_origin`) | [MCP](api/mcp.md#browser-origins) |
| 2 | **Licence** (enterprise) | An expired or invalid licence answers 403 `license_expired` | [IT guide](deployment/it-admin-guide.md) |
| 3 | **Authentication** | API key, JWT or OAuth 2.1 bearer token; only `initialize`, `server/discover` and `ping` run without one | [Authentication](auth/overview.md) |
| 4 | **Rate limit, role, tool scope** (enterprise) | Per-key rate limit; `read` / `readwrite` / `admin`; a key's `allowed_tools` limit what it may call and what `tools/list` shows | [RBAC](security/rbac.md) |
| 5 | **Argument check** (MCP) | Tool arguments are checked against each tool's declared types; a bad Lean value is refused, never ignored | [MCP](api/mcp.md) |
| 6 | **Input filter** (enterprise MCP) | SQL carrying prompt-injection patterns is refused | [Prompt-injection filtering](api/mcp.md#prompt-injection-filtering) |
| 7 | **Read-only validation** | One `SELECT` (or `WITH … SELECT`). Literals and comments are masked under **every engine dialect**, and all must pass; on CSV and Parquet, DuckDB's own parser must also see exactly one `SELECT` | [Query](api/query.md) |
| 8 | **Read-only session at the source** | Read-only is also enforced by the database session, not only by the validator. Privileged database roles are refused unless you opt in | [PostgreSQL](connectors/postgresql.md#use-a-least-privileged-role) |
| 9 | **Bounded execution** | The row ceiling bounds the fetch (memory), not only the reply; `truncated: true` says more rows existed; on database sources `TDB_QUERY_TIMEOUT` cancels the query at the source (enterprise) | [Query](api/query.md) |
| 10 | **Locked file engine** (CSV, Parquet) | DuckDB can read only the data directory: no extensions, no configuration changes | [Security updates](security/updates.md) |
| 11 | **Plain JSON values** | Same values on REST and MCP: exact decimals, binary as base64, ISO 8601 timestamps | [Query](api/query.md) |
| 12 | **Output filter** (enterprise MCP) | Cells carrying injection patterns are redacted; if the filter fails, the result is withheld | [Prompt-injection filtering](api/mcp.md#prompt-injection-filtering) |
| 13 | **Lean Mode** (opt-in) | Shaped **after** the filter, so a compact format or a cut can't restore a redacted value | [Saving tokens](api/mcp.md#saving-tokens-lean-mode) |
| 14 | **Audit** | One entry per request: who (key hint, AI client, protocol), what SQL, how many rows, the Lean settings served. Enterprise signs it into a hash chain | [Audit log](security/audit.md) |

The **wire gateway** runs steps 3, 4 and 7 on every statement, forces each
transaction read-only at the source, and writes two audit entries per statement:
one before it runs, one with the outcome. See the
[wire gateway](integrations/postgres-wire.md).

---

## Secrets and logs

- **Source credentials** are encrypted at rest (Fernet) in the registry; there is no
  plaintext mode. See [Credentials at rest](security/credentials.md).
- **API keys** never appear in a log: the audit log and the application log carry a
  six-character hint (from enterprise 0.20.3; earlier enterprise releases did log
  full keys — see [Security updates](security/updates.md)).
- **Log lines can't be forged:** line breaks in logged values are escaped.

---

## What TDB does not protect you from

Read this part before your security review does.

- **Over-broad database credentials.** TDB reads with the credentials you register. A
  read-only session still sees every table that role can see. Use a least-privileged
  role per source; TDB refuses the roles that can reach the server's files or programs.
- **Everything a key may query, it may read in full.** Column- and row-level rules are
  not shipped yet: restrict what a source exposes with the database role or a
  single-table registration. Named views (enterprise) give agents a curated surface.
- **Prompt injection, completely.** The filters match known patterns. They reduce the
  risk of instructions hidden in data; they don't eliminate it.
- **Loss of the whole audit log.** The hash chain makes an edited, reordered or removed
  entry detectable; it can't stop someone with access to the host from deleting the
  file. Export to [Splunk](integrations/splunk.md) or [S3](integrations/s3.md) for a
  copy outside the host. The community audit log is not hash-chained.
- **Traffic on the wire.** TDB doesn't terminate TLS; your proxy does.

---

## Verify it yourself

- `GET /v1/audit/verify` walks the hash chain (enterprise, admin role).
- Every behaviour above is covered by tests that run in CI against real PostgreSQL,
  MySQL and SQL Server; the community edition is open source:
  [tdb-project/tdb-community](https://github.com/tdb-project/tdb-community).
- Vulnerabilities are published as advisories, with what was exposed and what to do:
  [Security & updates](security/updates.md).
