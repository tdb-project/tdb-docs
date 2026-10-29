# Give an AI agent governed access to PostgreSQL

*Enterprise. About 15 minutes. Checked end to end against enterprise 0.21.0 and
PostgreSQL 16.*

By the end, an agent such as Claude Code can query your PostgreSQL database through TDB
with a key that can only read, can only use two tools, gets compact (Lean) results, and
leaves an audit entry for every call — including the ones TDB refuses.

## 1. Create a read-only database role

TDB reads with the credentials you register, so give it a role that can do nothing else.
In `psql`, as the database owner:

```sql
CREATE ROLE tdb_reader LOGIN PASSWORD 'choose-a-strong-password';
GRANT USAGE ON SCHEMA public TO tdb_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO tdb_reader;
```

TDB refuses roles that can reach the server's files or programs (a superuser, for
example). More: [least-privileged role](../connectors/postgresql.md#use-a-least-privileged-role).

## 2. Register the database

With an admin key (`TDB_API_KEYS` holds one):

```bash
export TDB=http://localhost:8000
export ADMIN_KEY=...   # a key from TDB_API_KEYS

curl -s -X POST $TDB/v1/sources \
  -H "Authorization: Bearer $ADMIN_KEY" -H "Content-Type: application/json" \
  -d '{"name":"shop","source_type":"postgres",
       "connection":{"host":"db.internal","port":5432,"dbname":"shop",
                     "user":"tdb_reader","password":"choose-a-strong-password","schema":"public"}}'
```

Leaving out `table` registers the whole database: the agent can see and join every table
the role can read.

## 3. Create a key for the agent

```bash
curl -s -X POST $TDB/v1/auth/keys \
  -H "Authorization: Bearer $ADMIN_KEY" -H "Content-Type: application/json" \
  -d '{"name":"claude-code-agent","role":"read","expires_in_days":90}'
```

The response holds the key's details under `key` (keep `key.id`) and the key itself as
`raw_key`, shown once. Save both:

```bash
export AGENT_ID=...    # key.id
export AGENT_KEY=...   # raw_key, starts with tdbk_
```

## 4. Limit it to two tools, and turn on Lean Mode

```bash
curl -s -X PUT $TDB/v1/auth/keys/$AGENT_ID/tools \
  -H "Authorization: Bearer $ADMIN_KEY" -H "Content-Type: application/json" \
  -d '{"tools":["schema_source","query_source"]}'

curl -s -X PUT $TDB/v1/auth/keys/$AGENT_ID/lean-mode \
  -H "Authorization: Bearer $ADMIN_KEY" -H "Content-Type: application/json" \
  -d '{"lean": true}'
```

The agent is now offered only `schema_source` and `query_source`, and every result comes
back in the compact table format unless a call asks otherwise.
See [Saving tokens](../api/mcp.md#saving-tokens-lean-mode).

## 5. Check what the agent will see

```bash
curl -s -X POST $TDB/v1/mcp \
  -H "Authorization: Bearer $AGENT_KEY" -H "Content-Type: application/json" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"query_source",
       "arguments":{"source_name":"shop","sql":"SELECT region, COUNT(*) AS orders, SUM(amount) AS revenue FROM orders GROUP BY region ORDER BY revenue DESC"}}}'
```

The result text, in the Lean format:

```json
{"source":"shop","columns":["region","orders","revenue"],
 "rows":[["APAC",100,"55617.83"],["AMER",100,"54536.79"],["EMEA",100,"53028.48"]],
 "rows_returned":3,"truncated":false,"lean":{"format":"table"}}
```

Two refusals worth seeing once:

| The agent tries | It gets | The audit log records |
|---|---|---|
| `preview_source` | `Tool 'preview_source' is not permitted for this API key.` | `denied`, `tool_not_permitted_preview_source` |
| `UPDATE orders SET amount = 0` | `SQL validation error: Blocked keyword: UPDATE` | `denied`, `sql_validation_failed` |

## 6. Connect the agent

Give the agent `$TDB` and `$AGENT_KEY`. For Claude Code, the TDB plugin asks for both;
for VS Code, Cursor and others, see [Connect an IDE / AI tool](../getting-started/ide-setup.md).

## Next

- Serve TDB over HTTPS before anyone outside your machine uses it:
  [HTTPS with Caddy](https-caddy.md).
- Rotate the agent's key before it expires: `POST /v1/auth/keys/{id}/rotate`.
- Run through the [production checklist](production-checklist.md).
