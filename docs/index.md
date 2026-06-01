# TDB Enterprise

**The Data-Bridge** is a self-hosted, auditable API layer that turns your databases into secure REST and MCP endpoints — governed, queryable, and AI-ready.

Register a data source once. Query it from REST, SQL, or any MCP-compatible AI tool. Every query is logged.

Your data is already scattered across PostgreSQL, MySQL, SQL Server, Snowflake, and CSV files — and every team now wants AI agents to query it. TDB makes that **existing, multi-source** data safely accessible **on your own infrastructure**, without moving it into a single cloud warehouse or routing it through a SaaS copilot, and with a tamper-evident audit log you own. → [Why TDB](why-tdb.md)

---

## Features

**Connectors**

- PostgreSQL, MySQL, SQL Server, and Snowflake connectors
- Multiple simultaneous registered sources

**Auth & API**

- Static API keys, plus DB-managed keys (create / rotate / revoke)
- Role-based access control (read / readwrite / admin)
- JWT authentication
- OAuth 2.1 with PKCE on MCP
- Per-API-key rate limiting
- CORS configuration

**Query & MCP**

- REST query endpoint (SELECT only)
- MCP tools: `query_source`, `schema_source`, `preview_source`, `filter_source`, `aggregate_source`
- YAML-defined named views with typed parameters
- Prompt-injection filtering (input + output)
- MCP tool-level allow-lists per API key
- Auto schema detection

**Audit & Compliance**

- Audit log (NDJSON) on every query
- Signed, hash-chained audit log (tamper-evident), with integrity verification via `GET /v1/audit/verify`
- Splunk HEC export via `POST /v1/audit/export`

**Observability**

- Prometheus metrics (`GET /metrics`)
- Schema caching with configurable TTL
- Health check (`GET /health`)

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

TDB never modifies your data. Every connector enforces read-only access at the connection or session level — not just SQL-validated.

---

## Quick links

- [Installation →](getting-started/installation.md)
- [Quickstart — first query in 5 minutes →](getting-started/quickstart.md)
- [PostgreSQL →](connectors/postgresql.md) · [MySQL →](connectors/mysql.md) · [SQL Server →](connectors/sqlserver.md) · [Snowflake →](connectors/snowflake.md)
- [Authentication overview →](auth/overview.md)
- [Role-based access control →](security/rbac.md)
- [Audit log & tamper verification →](security/audit.md)
- [YAML named views →](api/views.md)
- [Prometheus metrics →](observability/metrics.md)
- [Splunk HEC integration →](integrations/splunk.md)
- [All environment variables →](reference/environment-variables.md)
- [Editions & feature comparison →](pricing.md)

---

## Interactive API docs

When TDB is running, the full OpenAPI reference is available at:

- **Swagger UI** — `http://localhost:8000/docs`
- **ReDoc** — `http://localhost:8000/redoc`
- **OpenAPI JSON** — `http://localhost:8000/openapi.json`
