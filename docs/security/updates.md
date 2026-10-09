# Security & Updates

How TDB Enterprise handles security issues, how you receive fixes, and how to
report a vulnerability.

---

## Reporting a vulnerability

Please report security issues **privately** — do not open a public issue.

- Email **security@tdb.jiracorp.co.in**, or
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
published. Updating is a `docker pull` + restart (see below).

## How you receive a fix

TDB is **self-hosted** — it never calls home and nothing auto-updates, so you stay
in control (including in air-gapped environments). When we publish a fix, you update
to the patched image. The steps depend on how you receive TDB:

=== "Image tarball (current)"

    We deliver the patched version as a versioned image tarball. The image is generic
    and your license is supplied at runtime, so updating never touches your license:

    ```bash
    docker load < tdb-enterprise-vX.Y.Z.tar.gz
    # restart your container — your existing -e TDB_LICENSE is unchanged
    ```

    This works the same way for air-gapped deployments.

=== "Registry pull (future)"

    As we grow we'll offer registry-based updates, so you can pull a new version
    directly:

    ```bash
    docker pull <registry>/tdb-enterprise:vX.Y.Z
    # restart your container — your token is unchanged
    ```

=== "Trial image"

    We re-issue your trial image on the patched version; load it and restart.

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

