# TDB Enterprise

**The Data-Bridge** is a self-hosted, auditable API layer that turns your databases into secure REST and MCP endpoints — governed, queryable, and AI-ready.

Register a data source once. Query it from REST, SQL, or any MCP-compatible AI tool. Every query is logged.

---

## What's in this release

TDB Enterprise ships the following features. Everything listed here is live and tested.

**Connectors**

| Feature | Status |
|---|---|
| PostgreSQL connector | ✅ Shipped |
| MySQL connector | ✅ Shipped |
| SQL Server connector | ✅ Shipped |
| Snowflake connector | ✅ Shipped |
| Multiple simultaneous registered sources | ✅ Shipped |

**Auth & API**

| Feature | Status |
|---|---|
| Static API key auth | ✅ Shipped |
| DB-managed API keys (create / rotate / revoke) | ✅ Shipped |
| Role-based access control (read / readwrite / admin) | ✅ Shipped |
| JWT authentication | ✅ Shipped |
| OAuth 2.1 with PKCE on MCP | ✅ Shipped |
| Rate limiting per API key | ✅ Shipped |
| CORS configuration | ✅ Shipped |

**Query & MCP**

| Feature | Status |
|---|---|
| REST query endpoint (SELECT only) | ✅ Shipped |
| MCP `query_source` tool | ✅ Shipped |
| MCP `schema_source`, `preview_source`, `filter_source`, `aggregate_source` tools | ✅ Shipped |
| YAML-defined named views with typed parameters | ✅ Shipped |
| Prompt injection filtering (input + output) | ✅ Shipped |
| MCP tool-level allow-lists per API key | ✅ Shipped |
| Auto schema detection | ✅ Shipped |

**Audit & Compliance**

| Feature | Status |
|---|---|
| Audit log (NDJSON, every query) | ✅ Shipped |
| Signed hash-chained audit log (tamper-evident) | ✅ Shipped |
| Audit log integrity verification (`GET /v1/audit/verify`) | ✅ Shipped |
| Splunk HEC export (`POST /v1/audit/export`) | ✅ Shipped |

**Observability**

| Feature | Status |
|---|---|
| Prometheus metrics (`GET /metrics`) | ✅ Shipped |
| Schema caching with configurable TTL | ✅ Shipped |
| Health check (`GET /health`) | ✅ Shipped |

**Wave 5 (launch):** Public launch in progress — Show HN · GHAS on community repo.

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
- [Pricing →](pricing.md)

---

## Interactive API docs

When TDB is running, the full OpenAPI reference is available at:

- **Swagger UI** — `http://localhost:8000/docs`
- **ReDoc** — `http://localhost:8000/redoc`
- **OpenAPI JSON** — `http://localhost:8000/openapi.json`
