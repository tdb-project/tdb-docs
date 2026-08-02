# Editions

TDB is an open-core product. The community edition is **free, forever** under AGPLv3. The paid editions add the connectors, auth, and compliance features you need to run TDB in production for a real team.

This page shows exactly what each edition includes. **For pricing and a free 30-day evaluation, [email us](mailto:hello@tdb.jiracorp.co.in)** — we'll tailor a quote to your deployment and get you set up.

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

    [Get started →](getting-started/community.md){ .md-button }

=== "SMB"

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

    - All 7 MCP tools: `query_source`, `schema_source`, `preview_source`, `filter_source`, `aggregate_source`, `list_views`, `run_view`
    - Schema annotations — describe what a cryptic column holds, so an AI assistant reads it correctly instead of guessing
    - Prompt injection filtering (input + output)
    - Per-key MCP tool allow-lists

    **Access control**

    - RBAC: `read` / `readwrite` / `admin` roles per API key

    **Audit**

    - Signed hash-chained audit log (SHA-256, tamper-evident)
    - `GET /v1/audit/verify` — walk the chain, detect tampering
    - Configurable retention — seal the log into verifiable segments by size or age; TDB never deletes an entry

    **Observability**

    - Prometheus metrics (`GET /metrics`)
    - Schema caching with configurable TTL
    - Health check endpoint

    [Contact us →](mailto:hello@tdb.jiracorp.co.in){ .md-button .md-button--primary }

=== "Mid-Market"

    **Commercial license. Self-hosted.**

    For regulated industries — healthcare-tech, fintech, and any team that needs SOC 2-ready audit export and enterprise identity.

    **Everything in SMB, plus:**

    **SIEM integration**

    - Splunk HEC audit export (`POST /v1/audit/export`)
    - *Additional destinations (S3, Datadog, OpenTelemetry) are built on request — tell us which one you need*

    **Enterprise identity**

    - SSO / SAML / SCIM *(on the roadmap, not yet built)*

    **Fine-grained access control**

    - Column-level access control *(on the roadmap, not yet built)*

    **Support**

    - SLA-backed support — next business day response
    - Onboarding call and deployment review

    [Contact us →](mailto:hello@tdb.jiracorp.co.in){ .md-button .md-button--primary }

---

## Discounts

Annual subscriptions and larger / multi-seat deployments qualify for discounts. [Email us](mailto:hello@tdb.jiracorp.co.in) to discuss terms for your team.

---

## Edition comparison

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
| SSO / SAML / SCIM | ❌ | ❌ | On roadmap |
| Per-key rate limiting | ❌ | ✅ | ✅ |
| CORS configuration | ❌ | ✅ | ✅ |
| **Query & MCP** | | | |
| REST query endpoint (SELECT) | ✅ | ✅ | ✅ |
| Row limit per response † | 1,000 | 1,000 | 1,000 |
| MCP tools | `query_source` | All 7 | All 7 |
| MCP tool allow-lists per key | ❌ | ✅ | ✅ |
| Prompt injection filtering | ❌ | ✅ | ✅ |
| YAML named views | ❌ | ✅ | ✅ |
| Schema annotations for AI clients | ❌ | ✅ | ✅ |
| **Access control** | | | |
| RBAC (read / readwrite / admin) | ❌ | ✅ | ✅ |
| Column-level access control | ❌ | ❌ | On roadmap |
| **Audit** | | | |
| Local NDJSON audit log | ✅ | ✅ | ✅ |
| Hash-chained tamper-evident log | ❌ | ✅ | ✅ |
| Log integrity verification | ❌ | ✅ | ✅ |
| Audit log retention (sealed segments) | ❌ | ✅ | ✅ |
| Splunk HEC export | ❌ | ❌ | ✅ |
| Additional SIEM exporters | ❌ | ❌ | On request |
| **Observability** | | | |
| Prometheus metrics | ❌ | ✅ | ✅ |
| Schema caching | ❌ | ✅ | ✅ |
| Health check | ✅ | ✅ | ✅ |
| **Support** | | | |
| Community (GitHub Issues) | ✅ | ✅ | ✅ |
| SLA-backed (next business day) | ❌ | ❌ | ✅ |
| Onboarding call | ❌ | ❌ | ✅ |

† All editions currently cap responses at 1,000 rows — a deliberate safety default that keeps an AI agent from pulling an entire table. Push work down with aggregates and named views; pagination beyond 1,000 rows is on the roadmap.

---

## The open-source guarantee

Features in the community edition are free forever under AGPLv3. We will never move a free feature behind a paywall. If you evaluate TDB today with CSV, those capabilities are yours permanently — no license change, no feature flag.

---

## FAQ

**Can I run the enterprise tier on-premises?**
Yes. TDB is self-hosted by design. The commercial license covers your on-premises deployment. There is no SaaS-only restriction.

**How do evaluations work, and what happens when mine ends?**
[Email us](mailto:hello@tdb.jiracorp.co.in) and we'll issue a **30-day evaluation licence** at no charge — a dedicated, self-hosted image with the licence baked in, running entirely on your own infrastructure (no data reaches us). When it expires, TDB keeps running and `/health` stays healthy, but data and API requests return a clear `license_expired` response until you renew or convert. Need longer to evaluate? Just ask.

**Do you offer annual or volume discounts?**
Yes — annual subscriptions and larger / multi-seat deployments qualify for discounts. [Email us](mailto:hello@tdb.jiracorp.co.in) to discuss terms.

**How much does it cost?**
Paid editions are a flat self-hosted license (no per-query or per-row metering). We share pricing by email so we can match the right edition to your deployment and include a free 30-day evaluation — [get in touch](mailto:hello@tdb.jiracorp.co.in).

**Is the source code for the enterprise tier available?**
No. The enterprise tier is a closed-source commercial product. The community edition is fully open under AGPLv3.

**I need BigQuery / Datadog / SSO now. When do those ship?**
They are on the roadmap and not yet built. We deliberately don't publish dates we haven't committed engineering to — a date we miss is worse than no date, particularly in a product you're evaluating for compliance. If one of these is blocking your evaluation, [email us](mailto:hello@tdb.jiracorp.co.in): that's the signal that moves it up, and design partners get built first.
