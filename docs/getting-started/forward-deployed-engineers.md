# For forward-deployed engineers

You go into a customer's environment and make an AI agent work on *their* data. At every
customer you meet different databases, a new security review and a network you cannot
change, and you build a data API for them that someone has to maintain after you leave.

TDB is that data layer, already built. Deploy it inside the customer's network, register
their sources, and give both the agent and the customer's security team one governed,
audited endpoint. Next customer, same setup.

!!! note "Commercial edition"
    Most of this page needs a commercial edition: the free community edition serves one CSV
    file. For a 30-day evaluation, [email us](mailto:hello@tdb.asvanlabs.com).

## What changes at each engagement

| At each customer you need… | What TDB gives you |
|---|---|
| Access to whatever they run | One registration call per source: [PostgreSQL](../connectors/postgresql.md), [MySQL](../connectors/mysql.md), [SQL Server](../connectors/sqlserver.md), [Snowflake](../connectors/snowflake.md), [ClickHouse](../connectors/clickhouse.md), [DuckDB](../connectors/duckdb.md), [Parquet](../connectors/parquet.md) and CSV. The same REST, SQL and MCP interface whatever the engine |
| An agent that can use it | Built-in [MCP tools](../api/mcp.md): query, schema, preview, filter, aggregate and named views. Tested as a Claude custom connector on claude.ai and Claude Desktop; plugins for [Claude Code and Claude Desktop](ide-setup.md) and an extension for VS Code |
| Analysts and BI tools too | The [PostgreSQL wire gateway](../integrations/postgres-wire.md): psql, Metabase and DBeaver connect unchanged, governed and audited |
| A yes from their security team | Read-only enforced, credentials encrypted at rest, privileged database roles refused, every query in a [signed, hash-chained audit log](../security/audit.md) with [Splunk](../integrations/splunk.md) and [S3](../integrations/s3.md) export. Hand the reviewers the [architecture and security model](../architecture.md) |
| To keep personal data from the agent | [Column and row policies](../security/policies.md): withhold, mask or filter per key or SSO group. `tdb policy suggest` drafts one from column names and types |
| A person to approve sensitive reads | [Query approvals](../security/approvals.md) |
| Their own sign-in | [Single sign-on (OIDC) and SCIM](../auth/sso.md) |
| One deployment for several customers | [Tenants](../security/tenants.md): each with its own sources, keys, policies and audit log |
| Lower token cost | [Lean Mode](../api/mcp.md#saving-tokens-lean-mode): a 100-row answer went from 10,412 to 6,868 tokens and a 30-table schema from 5,235 to 3,025, with the same answers on our 21-question check ([measurements](../deployment/performance.md)) |
| To stay inside their network | One Docker image or [Helm chart](../deployment/kubernetes.md). Claude Desktop with the TDB extension and Claude Code need no internet access at TDB ([Claude in a private network](../deployment/private-network.md)) |
| The same queries every time | [Named views](../api/views.md) with typed parameters: write a query once, the agent calls it by name |
| To debug on site | Every query is audited, failures included, with the reason; [Prometheus metrics](../observability/metrics.md) |

## An engagement, step by step

1. **Install** inside the customer's network: [installation](installation.md), or
   [Kubernetes](../deployment/kubernetes.md).
2. **Register their sources** with a read-only database user. Each database connector page says
   which privileges to grant.
3. **Write a policy** for the agent's key: what it may see, what is masked, which rows.
   See [two keys, one table](../recipes/governed-access.md).
4. **Connect the agent**: [IDE and AI tool setup](ide-setup.md), or
   [PostgreSQL for an AI agent](../recipes/postgres-agent.md) from start to finish.
5. **Hand over the security material**: the [architecture page](../architecture.md), the
   [audit log](../security/audit.md) and the [security updates](../security/updates.md).
6. **Before go-live**, run the [production checklist](../recipes/production-checklist.md).

Script steps 2 to 4 once with the REST API and the CLI, and you can repeat them at the next
customer.

## What TDB does not do

- **Write.** TDB is read-only by design. An agent that writes back to the customer's
  database needs another path for that.
- **Admin web UI.** Setup is the REST API and the CLI.
- **Other sources.** BigQuery, MongoDB and JSON files are
  [built on request](../pricing.md).

## Next

[Email us](mailto:hello@tdb.asvanlabs.com) for a 30-day evaluation on a real engagement,
or try the idea first with one CSV in the free [community edition](community.md).
