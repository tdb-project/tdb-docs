---
template: home.html
hide:
  # With navigation.tabs, "Home" is already a tab. The left sidebar on this page
  # then renders a single entry that says "Home" too, so the landing page of the
  # docs showed the same link twice, side by side. Hiding the sidebar here drops
  # the duplicate and gives the feature cards the full content width.
  - navigation
  - toc
---

!!! info "Evaluating rather than deploying?"
    The **free community edition** runs one CSV source in a single Docker command,
    with the same read-only enforcement and the same audit log. It is the fastest
    way to answer "does this work for us?" — start at
    [Community Edition](getting-started/community.md), or compare the two at
    [Editions](pricing.md).

## Start here

<div class="grid cards" markdown>

-   **Evaluating it**

    ---

    Run the free edition against one CSV in a single Docker command, then compare
    editions.

    [:octicons-arrow-right-24: Community Edition](getting-started/community.md) ·
    [Editions](pricing.md)

-   **Deploying and running it**

    ---

    Requirements, topologies, configuration, TLS, capacity and day-2 operations.

    [:octicons-arrow-right-24: IT Administrator Guide](deployment/it-admin-guide.md) ·
    [Release notes](release-notes.md)

-   **Reviewing its security**

    ---

    What a request passes through, what each layer guarantees, and what TDB does not
    protect you from.

    [:octicons-arrow-right-24: Architecture & security model](architecture.md) ·
    [Advisories](security/updates.md)

-   **Connecting AI agents**

    ---

    Set up Claude, Cursor or VS Code, scope a key to its tools, and keep results lean.

    [:octicons-arrow-right-24: IDE / AI tool setup](getting-started/ide-setup.md) ·
    [MCP reference](api/mcp.md)

</div>

Your data is already scattered across PostgreSQL, MySQL, SQL Server, Snowflake, and
CSV files — and every team now wants AI agents to query it. TDB makes that
**existing, multi-source** data safely accessible **on your own infrastructure**,
without moving it into a single cloud warehouse or routing it through a SaaS
copilot, and with a tamper-evident audit log you own.

---

## Features

<div class="grid cards" markdown>

-   :material-database-outline:{ .lg .middle } **Connectors**

    ---

    PostgreSQL, MySQL, SQL Server, Snowflake, Parquet files and DuckDB databases. Database-wide or single-table
    registration per source, and multiple simultaneous registered sources.

    [:octicons-arrow-right-24: All connectors](connectors/postgresql.md)

-   :material-key-outline:{ .lg .middle } **Auth & API**

    ---

    Static API keys plus DB-managed keys (create / rotate / revoke), role-based
    access control (read / readwrite / admin), JWT, OAuth 2.1 with PKCE on MCP,
    per-API-key rate limiting and CORS configuration.

    [:octicons-arrow-right-24: Authentication](auth/overview.md)

-   :material-api:{ .lg .middle } **Query & views**

    ---

    A REST query endpoint (SELECT and CTEs only, row ceiling enforced),
    YAML-defined named views with typed parameters, and auto schema detection.

    [:octicons-arrow-right-24: Query API](api/query.md)

-   :material-robot-outline:{ .lg .middle } **MCP for AI tools**

    ---

    Seven MCP tools — `query_source`, `schema_source`, `preview_source`,
    `filter_source`, `aggregate_source`, `list_views`, `run_view` — on one endpoint
    that speaks the `2026-07-28` specification and the earlier handshake-era
    revisions. Tools
    are marked read-only and return structured output, with prompt-injection
    filtering and per-key tool allow-lists. Claude Code connects on the newest
    revision, and a free [VS Code extension](getting-started/ide-setup.md#vs-code)
    sets TDB up in one command.

    [:octicons-arrow-right-24: MCP reference](api/mcp.md)

-   :material-feather:{ .lg .middle } **Lean by design**

    ---

    Every token an agent spends reading a result is a token it can't spend on
    your question. [Lean Mode](api/mcp.md#saving-tokens-lean-mode) cuts a 100-row
    answer from 10,412 to 6,868 tokens and a 30-table schema from 5,235 to 3,025,
    with the same answers on our 21-question check. Agents can also aggregate and
    filter in the database instead of pulling rows, and a scoped key lists only
    its own tools. Set it per call, per key or server-wide.

    [:octicons-arrow-right-24: Saving tokens](api/mcp.md#saving-tokens-lean-mode)

-   :material-elephant:{ .lg .middle } **PostgreSQL wire gateway**

    ---

    Metabase, DBeaver, `psql`, JDBC and other PostgreSQL clients connect with
    only a new host, port and password. Every statement is checked as read-only
    and audited before it runs, as it is over REST and MCP.

    [:octicons-arrow-right-24: Wire gateway](integrations/postgres-wire.md)

-   :material-shield-check-outline:{ .lg .middle } **Audit & compliance**

    ---

    An NDJSON audit log on every query, signed and hash-chained so tampering is
    detectable, naming the AI client behind each MCP call, with integrity
    verification via `GET /v1/audit/verify` and incremental export to Splunk HEC
    or S3. Prometheus metrics and a health check cover the rest of operations.

    [:octicons-arrow-right-24: Audit log](security/audit.md)

-   :material-server-security:{ .lg .middle } **Read-only, enforced**

    ---

    TDB never modifies your data. Every connector enforces read-only at the
    connection or session level — not merely by validating the SQL — and
    refusals are written to the audit log too.

    [:octicons-arrow-right-24: RBAC](security/rbac.md)

</div>

---

## How it works

```
Postgres · MySQL · SQL Server · Snowflake
      │
      │  read-only connection (per connector)
      ▼
 ┌────────────────────────────────────────────┐
 │              The Data-Bridge               │
 │                                            │
 │  POST /v1/sources     ← register source    │
 │  POST /v1/query       ← SQL SELECT         │
 │  POST /v1/mcp         ← MCP tool calls     │
 │  GET  /v1/views       ← YAML-defined views │
 │  :5432 (opt-in)       ← Postgres wire      │
 │  GET  /metrics        ← Prometheus         │
 │                                            │
 │  Every query → hash-chained audit log      │
 │  RBAC enforced per key (read/readwrite/    │
 │  admin); tool allow-lists per MCP key      │
 └────────────────────────────────────────────┘
      │
      │  Authorization: Bearer <token>
      ▼
 Your app · Claude Desktop · Cursor · Metabase · psql
```

---

## Quick links

<div class="grid cards" markdown>

-   **Get running**

    ---

    [:octicons-arrow-right-24: Installation](getting-started/installation.md)<br>
    [:octicons-arrow-right-24: Quickstart — first query in 5 minutes](getting-started/quickstart.md)<br>
    [:octicons-arrow-right-24: Community edition (free, CSV)](getting-started/community.md)<br>
    [:octicons-arrow-right-24: Connect an IDE or AI tool](getting-started/ide-setup.md)

-   **Connect a source**

    ---

    [:octicons-arrow-right-24: PostgreSQL](connectors/postgresql.md)<br>
    [:octicons-arrow-right-24: MySQL](connectors/mysql.md)<br>
    [:octicons-arrow-right-24: SQL Server](connectors/sqlserver.md)<br>
    [:octicons-arrow-right-24: Snowflake](connectors/snowflake.md)<br>
    [:octicons-arrow-right-24: Parquet](connectors/parquet.md)<br>
    [:octicons-arrow-right-24: DuckDB](connectors/duckdb.md)

-   **Govern it**

    ---

    [:octicons-arrow-right-24: Authentication overview](auth/overview.md)<br>
    [:octicons-arrow-right-24: Role-based access control](security/rbac.md)<br>
    [:octicons-arrow-right-24: Audit log & tamper verification](security/audit.md)<br>
    [:octicons-arrow-right-24: Credentials at rest](security/credentials.md)

-   **Operate it**

    ---

    [:octicons-arrow-right-24: YAML named views](api/views.md)<br>
    [:octicons-arrow-right-24: PostgreSQL wire gateway](integrations/postgres-wire.md)<br>
    [:octicons-arrow-right-24: Prometheus metrics](observability/metrics.md)<br>
    [:octicons-arrow-right-24: Splunk HEC integration](integrations/splunk.md)<br>
    [:octicons-arrow-right-24: All environment variables](reference/environment-variables.md)

</div>

---

## Interactive API docs

When TDB is running, the full OpenAPI reference is available at:

- **Swagger UI** — `http://localhost:8000/docs`
- **ReDoc** — `http://localhost:8000/redoc`
- **OpenAPI JSON** — `http://localhost:8000/openapi.json`
