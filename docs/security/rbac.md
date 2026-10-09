# Role-Based Access Control

TDB Enterprise enforces a three-tier role system on every DB-managed API key. Roles are checked on every request; a denied request returns HTTP 403 with a clear message.

---

## Roles

| Role | Can query | Can register/delete sources | Can manage API keys | Can verify audit log |
|---|---|---|---|---|
| `read` | ✅ | ❌ | ❌ | ❌ |
| `readwrite` | ✅ | ✅ | ❌ | ❌ |
| `admin` | ✅ | ✅ | ✅ | ✅ |

**Role sources:**

| Key type | Role |
|---|---|
| Static env keys (`TDB_API_KEYS`) | Always `admin` |
| JWT tokens | `role` claim in the JWT payload (default: `read` from 0.11.0; `admin` before) |
| DB-managed keys | `role` column in the registry database (a key created without a `role` is `read` from 0.11.0; `admin` before) |

---

## Endpoint access matrix

| Endpoint | Minimum role |
|---|---|
| `GET /` | None (public — version banner) |
| `GET /health` | None (public) |
| `GET /metrics` | None (public — restrict via network policy in production) |
| `GET /v1/version` | any valid key |
| `GET /v1/sources` | `read` |
| `GET /v1/sources/{id}` | `read` |
| `GET /v1/sources/{id}/schema` | `read` |
| `GET /v1/sources/{ref}/annotations` | `read` |
| `POST /v1/query` | `read` |
| `POST /v1/mcp` | `read` |
| `GET /v1/views` | `read` |
| `GET /v1/views/{name}` | `read` |
| `POST /v1/views/{name}/run` | `read` |
| `POST /v1/sources` | `readwrite` |
| `DELETE /v1/sources/{id}` | `readwrite` |
| `PUT /v1/sources/{ref}/annotations` | `readwrite` |
| `DELETE /v1/sources/{ref}/annotations` | `readwrite` |
| `POST /v1/auth/keys` | `admin` |
| `GET /v1/auth/keys` | `admin` |
| `DELETE /v1/auth/keys/{id}` | `admin` |
| `POST /v1/auth/keys/{id}/rotate` | `admin` |
| `PUT /v1/auth/keys/{id}/rate-limit` | `admin` |
| `PUT /v1/auth/keys/{id}/role` | `admin` |
| `PUT /v1/auth/keys/{id}/tools` | `admin` |
| `GET /v1/audit/verify` | `admin` |
| `GET /v1/audit/history` | `admin` |
| `POST /v1/audit/rotate` | `admin` |
| `POST /v1/audit/export` | `admin` |
| `GET /v1/audit/export/status` | `admin` |

Grouped by minimum role rather than by path, since the question this table
answers is "what can a key with role X do".

!!! note "Rate limiting is checked before the role"

    A key that is over its rate limit gets **429** on any endpoint, including one
    its role would not have allowed anyway. A 429 therefore does not mean the
    role was sufficient.

---

## Creating keys with a specific role

```bash
# Read-only key for a dashboard or BI tool
curl -X POST http://localhost:8000/v1/auth/keys \
  -H "Authorization: Bearer <ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"name": "dashboard", "role": "read"}'

# Readwrite key for a pipeline that registers sources
curl -X POST http://localhost:8000/v1/auth/keys \
  -H "Authorization: Bearer <ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"name": "etl-pipeline", "role": "readwrite"}'
```

---

## Changing a key's role

Role changes take effect immediately.

```bash
curl -X PUT http://localhost:8000/v1/auth/keys/<KEY_ID>/role \
  -H "Authorization: Bearer <ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"role": "readwrite"}'
```

Valid values: `"read"`, `"readwrite"`, `"admin"`. Invalid values return HTTP 422.

---

## Restricting tool access

Beyond the role system, you can restrict which MCP tools a key can call. This is useful when you want a key that can query data but cannot introspect schema, or cannot run aggregate queries — or a key for an agent that may only run the [views](../api/views.md) you defined.

```bash
# Allow only query_source and schema_source
curl -X PUT http://localhost:8000/v1/auth/keys/<KEY_ID>/tools \
  -H "Authorization: Bearer <ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"tools": ["query_source", "schema_source"]}'

# Remove restriction (allow all tools)
curl -X PUT http://localhost:8000/v1/auth/keys/<KEY_ID>/tools \
  -H "Authorization: Bearer <ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"tools": null}'
```

Available MCP tools: `query_source`, `schema_source`, `preview_source`, `filter_source`, `aggregate_source`, `list_views`, `run_view`.

**The scope also governs the REST routes that do the same thing, from enterprise
0.12.0.** Earlier releases checked it only on MCP, so a key scoped to
`["run_view"]` could still send any SQL to `POST /v1/query`.

| REST route | Requires tool |
|---|---|
| `POST /v1/query` | `query_source` |
| `GET /v1/sources/{ref}/schema` | `schema_source` |
| `GET /v1/views`, `GET /v1/views/{name}` | `list_views` |
| `POST /v1/views/{name}/run` | `run_view` |

A REST request outside the scope returns **403** and is audited as
`tool_not_permitted_<tool>`. Routes with no MCP equivalent (source and key
management, audit) are governed by the role alone.

!!! note
    Tool-level restrictions only apply to DB-managed keys. Static env keys and JWT tokens are never tool-restricted.

---

## Error responses

| Scenario | HTTP status | Detail |
|---|---|---|
| No credentials | 401 | `Invalid or missing credentials` |
| Invalid token | 401 | `Invalid or expired token` |
| Valid key, insufficient role | 403 | `Insufficient privileges. Required: 'readwrite', have: 'read'.` |
| Valid key, tool not allowed (MCP) | 200 (tool-error) | `Tool 'schema_source' is not permitted for this API key.` |
| Valid key, tool not allowed (REST, from 0.12.0) | 403 | `This API key's allowed_tools does not include 'query_source', which this endpoint requires.` |

MCP tool-access denials return HTTP 200 with `isError: true` in the JSON-RPC result, not a protocol-level error, so MCP clients handle them as a failed tool call rather than a connection failure.

---

## Recommended setup for production

```bash
# 1. Bootstrap key — used only to create other keys, then locked away
TDB_API_KEYS=your-strong-bootstrap-key

# 2. Create a read-only key for each integration
curl -X POST .../v1/auth/keys -d '{"name":"grafana","role":"read","expires_in_days":365}'
curl -X POST .../v1/auth/keys -d '{"name":"claude-desktop","role":"read"}'

# 3. Create a readwrite key for the team that registers sources
curl -X POST .../v1/auth/keys -d '{"name":"data-platform-team","role":"readwrite"}'

# 4. Restrict the Claude Desktop key to safe MCP tools only
curl -X PUT .../v1/auth/keys/<claude_key_id>/tools \
  -d '{"tools":["query_source","schema_source","preview_source"]}'
```
