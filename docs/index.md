---
hide:
  # With navigation.tabs, "Home" is already a tab. The left sidebar on this page
  # then renders a single entry that says "Home" too, so the landing page of the
  # docs showed the same link twice, side by side. Hiding the sidebar here drops
  # the duplicate and gives the feature cards the full content width.
  - navigation
---

# TDB Enterprise

**The Data-Bridge** is a self-hosted, auditable API layer that turns your databases
into secure REST and MCP endpoints — governed, queryable, and AI-ready.

Register a data source once. Query it from REST, SQL, or any MCP-compatible AI
tool. Every query is logged.

[Install it :material-arrow-right:](getting-started/installation.md){ .md-button .md-button--primary }
[First query in 5 minutes](getting-started/quickstart.md){ .md-button }
[Why TDB](why-tdb.md){ .md-button }

!!! info "Evaluating rather than deploying?"
    The **free community edition** runs one CSV source in a single Docker command,
    with the same read-only enforcement and the same audit log. It is the fastest
    way to answer "does this work for us?" — start at
    [Community Edition](getting-started/community.md), or compare the two at
    [Editions](pricing.md).

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

    PostgreSQL, MySQL, SQL Server and Snowflake. Database-wide or single-table
    registration per source, and multiple simultaneous registered sources.

    [:octicons-arrow-right-24: All connectors](connectors/postgresql.md)

-   :material-key-outline:{ .lg .middle } **Auth & API**

    ---

    Static API keys plus DB-managed keys (create / rotate / revoke), role-based
    access control (read / readwrite / admin), JWT, OAuth 2.1 with PKCE on MCP,
    per-API-key rate limiting and CORS configuration.

    [:octicons-arrow-right-24: Authentication](auth/overview.md)

-   :material-api:{ .lg .middle } **Query & MCP**

    ---

    A REST query endpoint (SELECT only) and seven MCP tools — `query_source`,
    `schema_source`, `preview_source`, `filter_source`, `aggregate_source`,
    `list_views`, `run_view`. YAML-defined named views with typed parameters,
    prompt-injection filtering on input and output, per-key tool allow-lists,
    and auto schema detection.

    [:octicons-arrow-right-24: Query API](api/query.md)

-   :material-shield-check-outline:{ .lg .middle } **Audit & compliance**

    ---

    An NDJSON audit log on every query, signed and hash-chained so tampering is
    detectable, with integrity verification via `GET /v1/audit/verify` and
    incremental export to Splunk HEC or S3.

    [:octicons-arrow-right-24: Audit log](security/audit.md)

-   :material-chart-line:{ .lg .middle } **Observability**

    ---

    Prometheus metrics at `GET /metrics`, schema caching with a configurable
    TTL, and a health check at `GET /health`.

    [:octicons-arrow-right-24: Metrics](observability/metrics.md)

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
 │              TDB Enterprise                │
 │                                            │
 │  POST /v1/sources     ← register source    │
 │  POST /v1/query       ← SQL SELECT         │
 │  POST /v1/mcp         ← MCP tool calls     │
 │  GET  /v1/views       ← YAML-defined views │
 │  GET  /metrics        ← Prometheus         │
 │                                            │
 │  Every query → hash-chained audit log      │
 │  RBAC enforced per key (read/readwrite/    │
 │  admin); tool allow-lists per MCP key      │
 └────────────────────────────────────────────┘
      │
      │  Authorization: Bearer <token>
      ▼
 Your app / Claude Desktop / Cursor
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
    [:octicons-arrow-right-24: Snowflake](connectors/snowflake.md)

-   **Govern it**

    ---

    [:octicons-arrow-right-24: Authentication overview](auth/overview.md)<br>
    [:octicons-arrow-right-24: Role-based access control](security/rbac.md)<br>
    [:octicons-arrow-right-24: Audit log & tamper verification](security/audit.md)<br>
    [:octicons-arrow-right-24: Credentials at rest](security/credentials.md)

-   **Operate it**

    ---

    [:octicons-arrow-right-24: YAML named views](api/views.md)<br>
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
