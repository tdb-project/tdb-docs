# Pricing

TDB is an open-core product. The community edition is free, forever. Paid tiers add the connectors, auth, and compliance features you need to run TDB in production for a real team.

**Pricing is published transparently. No "contact sales for pricing."**

---

## Tiers

=== "Community — Free"

    **AGPLv3. Self-hosted. No credit card.**

    Built for developers who want to evaluate the concept with a single data source.

    **Data sources**

    - CSV connector
    - 1 registered source at a time

    **Auth**

    - Single static API key

    **Query & API**

    - REST query endpoint (SELECT only, 1,000-row cap)
    - Auto schema detection
    - OpenAPI / Swagger UI

    **MCP**

    - `query_source` tool only

    **Audit**

    - Local NDJSON audit log (append-only, one entry per query)

    **Install**

    - Docker Compose one-liner
    - CLI (`tdb serve`, `tdb register`, `tdb query`)

    [Get started →](getting-started/installation.md){ .md-button }

=== "SMB — $499 / month"

    **Commercial license. Self-hosted.**

    Everything you need to run TDB in production — multiple databases, governed MCP access, and a tamper-evident audit trail.

    **Data sources**

    - PostgreSQL, MySQL, SQL Server, Snowflake
    - Unlimited simultaneous registered sources

    **Auth**

    - JWT authentication
    - API key rotation and management
    - OAuth 2.1 with PKCE — Claude Desktop and Cursor connect natively
    - Per-key rate limiting
    - CORS configuration

    **Query & API**

    - All Community features, plus:
    - Named views (YAML-defined, typed parameters)

    **MCP**

    - All 5 MCP tools: `query_source`, `schema_source`, `preview_source`, `filter_source`, `aggregate_source`
    - Prompt injection filtering (input + output)
    - Per-key MCP tool allow-lists

    **Access control**

    - RBAC: `read` / `readwrite` / `admin` roles per API key

    **Audit**

    - Signed hash-chained audit log (SHA-256, tamper-evident)
    - `GET /v1/audit/verify` — walk the chain, detect tampering

    **Observability**

    - Prometheus metrics (`GET /metrics`)
    - Schema caching with configurable TTL
    - Health check endpoint

    [Contact us →](mailto:hello@tdb.jiracorp.co.in){ .md-button .md-button--primary }

=== "Mid-Market — $1,999 / month"

    **Commercial license. Self-hosted.**

    For regulated industries — healthcare-tech, fintech, and any team that needs SOC 2-ready audit export and enterprise identity.

    **Everything in SMB, plus:**

    **SIEM integration**

    - Splunk HEC audit export (`POST /v1/audit/export`)
    - *(Datadog + OpenTelemetry exporters — coming Q3 2026)*

    **Enterprise identity**

    - SSO / SAML / SCIM *(coming Q3 2026)*

    **Fine-grained access control**

    - Column-level access control *(coming Q3 2026)*

    **Support**

    - SLA-backed support — next business day response
    - Onboarding call and deployment review

    [Contact us →](mailto:hello@tdb.jiracorp.co.in){ .md-button .md-button--primary }

---

## Annual pricing

Save 15% on any paid tier with an annual subscription.

| Tier | Monthly | Annual (15% off) |
|---|---|---|
| Community | Free | Free |
| SMB | $499 / month | $5,091 / year |
| Mid-Market | $1,999 / month | $20,389 / year |

---

## Tier comparison

| Feature | Community | SMB | Mid-Market |
|---|---|---|---|
| **Data sources** | | | |
| CSV connector | ✅ | ✅ | ✅ |
| PostgreSQL | ❌ | ✅ | ✅ |
| MySQL | ❌ | ✅ | ✅ |
| SQL Server | ❌ | ✅ | ✅ |
| Snowflake | ❌ | ✅ | ✅ |
| Simultaneous sources | 1 | Unlimited | Unlimited |
| **Auth** | | | |
| Static API key | ✅ | ✅ | ✅ |
| API key rotation & management | ❌ | ✅ | ✅ |
| JWT authentication | ❌ | ✅ | ✅ |
| OAuth 2.1 with PKCE | ❌ | ✅ | ✅ |
| SSO / SAML / SCIM | ❌ | ❌ | Coming Q3 2026 |
| Per-key rate limiting | ❌ | ✅ | ✅ |
| CORS configuration | ❌ | ✅ | ✅ |
| **Query & MCP** | | | |
| REST query endpoint (SELECT) | ✅ | ✅ | ✅ |
| Row limit | 1,000 | 1,000 | 1,000 |
| MCP tools | `query_source` | All 5 | All 5 |
| MCP tool allow-lists per key | ❌ | ✅ | ✅ |
| Prompt injection filtering | ❌ | ✅ | ✅ |
| YAML named views | ❌ | ✅ | ✅ |
| **Access control** | | | |
| RBAC (read / readwrite / admin) | ❌ | ✅ | ✅ |
| Column-level access control | ❌ | ❌ | Coming Q3 2026 |
| **Audit** | | | |
| Local NDJSON audit log | ✅ | ✅ | ✅ |
| Hash-chained tamper-evident log | ❌ | ✅ | ✅ |
| Log integrity verification | ❌ | ✅ | ✅ |
| Splunk HEC export | ❌ | ❌ | ✅ |
| Additional SIEM exporters | ❌ | ❌ | Coming Q3 2026 |
| **Observability** | | | |
| Prometheus metrics | ❌ | ✅ | ✅ |
| Schema caching | ❌ | ✅ | ✅ |
| Health check | ✅ | ✅ | ✅ |
| **Support** | | | |
| Community (GitHub Issues) | ✅ | ✅ | ✅ |
| SLA-backed (next business day) | ❌ | ❌ | ✅ |
| Onboarding call | ❌ | ❌ | ✅ |

---

## The open-source guarantee

Features in the community edition are free forever under AGPLv3. We will never move a free feature behind a paywall. If you evaluate TDB today with CSV, those capabilities are yours permanently — no license change, no feature flag.

---

## FAQ

**Can I run the enterprise tier on-premises?**
Yes. TDB is self-hosted by design. The commercial license covers your on-premises deployment. There is no SaaS-only restriction.

**What happens when my trial ends?**
We don't run time-limited trials. [Email us](mailto:hello@tdb.jiracorp.co.in) and we'll arrange a 30-day evaluation licence at no charge.

**Do you offer volume discounts beyond the 15% annual?**
Yes, for deployments with more than 10 seats or custom support requirements. [Get in touch](mailto:hello@tdb.jiracorp.co.in).

**Is the source code for the enterprise tier available?**
No. The enterprise tier is a closed-source commercial product. The community edition is fully open under AGPLv3.

**I need BigQuery / Datadog / SSO now. When do those ship?**
Target Q3 2026. If these are blocking your evaluation, [email us](mailto:hello@tdb.jiracorp.co.in) — design partner conversations accelerate the roadmap.
