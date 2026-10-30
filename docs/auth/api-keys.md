# API Key Management

## Your initial admin key

When TDB first starts it has no DB-managed keys. To create your first key (or make
any authenticated request), you need the **static env key** — the value you set for
`TDB_API_KEYS` when starting the container:

```bash
Authorization: Bearer <value-of-TDB_API_KEYS>
```

If you started the container without setting `TDB_API_KEYS` (e.g. via Docker Desktop),
the fallback key is `dev-insecure-key-change-me`. Use it to bootstrap, then replace it:

```bash
# Bootstrap: create a real admin key using the dev fallback
curl -X POST http://localhost:8000/v1/auth/keys \
  -H "Authorization: Bearer dev-insecure-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{"name":"admin","role":"admin"}'

# Save the raw_key from the response — that is now your permanent admin key.
# Then restart the container with TDB_API_KEYS set to a strong value:
#   docker run ... -e TDB_API_KEYS=your-strong-key-here ...
```

!!! warning
    `dev-insecure-key-change-me` is a public default. Replace it before exposing
    TDB to any network outside your local machine.

---

## DB-managed keys

DB-managed keys are stored as SHA-256 hashes in the TDB registry. The raw key value
is shown **once** — at creation or rotation time — and cannot be retrieved afterwards.
Keep it safe.

All key management endpoints require **admin** role. See [RBAC →](../security/rbac.md).

---

## Key format

All DB-managed keys are prefixed with `tdbk_` followed by 32 URL-safe random bytes:

```
tdbk_a1b2c3d4e5f6...
```

---

## Create a key

```
POST /v1/auth/keys
```

**Request body:**

```json
{
  "name": "ci-pipeline",
  "expires_in_days": 90,
  "rate_limit": 120,
  "role": "read"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string (1–100 chars) | Yes | Human-readable label |
| `expires_in_days` | integer > 0 | No | Key auto-expires after this many days. Omit for no expiry. |
| `rate_limit` | integer > 0 | No | Requests per minute. Omit to use the server default (`TDB_DEFAULT_RATE_LIMIT`, default 60). |
| `role` | `"read"`, `"readwrite"`, or `"admin"` | No | RBAC role. Default: `"read"` (from 0.11.0; `"admin"` before). Existing keys keep their role. See [RBAC →](../security/rbac.md). |

**Response (HTTP 201):**

```json
{
  "key": {
    "id": "f47ac10b-...",
    "name": "ci-pipeline",
    "key_prefix": "tdbk_a1b2",
    "created_at": "2026-05-22T09:00:00Z",
    "expires_at": "2026-08-20T09:00:00Z",
    "revoked_at": null,
    "last_used_at": null,
    "rate_limit": 120,
    "role": "read",
    "allowed_tools": null
  },
  "raw_key": "tdbk_a1b2c3d4e5f6..."
}
```

!!! danger "Save the raw key now"
    The `raw_key` field is returned exactly once. It cannot be retrieved later.
    If you lose it, rotate the key to get a new one.

---

## List keys

Returns metadata for all keys (active, expired, and revoked). The raw key value
is never included.

```
GET /v1/auth/keys
```

**Response:**

```json
{
  "keys": [
    {
      "id": "f47ac10b-...",
      "name": "ci-pipeline",
      "key_prefix": "tdbk_a1b2",
      "created_at": "2026-05-22T09:00:00Z",
      "expires_at": "2026-08-20T09:00:00Z",
      "revoked_at": null,
      "last_used_at": "2026-05-22T10:30:00Z",
      "rate_limit": 120
    }
  ]
}
```

The `key_prefix` (first 12 characters) lets you identify which key is which
without exposing the full value.

---

## Revoke a key

Immediately invalidates the key. Revocation is permanent — use rotation instead
if you want a replacement.

```
DELETE /v1/auth/keys/<key_id>
```

Returns HTTP 204 on success. Returns HTTP 404 if the key ID is not found or is
already revoked.

---

## Rotate a key

Atomically revokes the existing key and issues a replacement with the same name,
expiry, rate limit, role, allowed tools, Lean Mode setting and attributes. The new raw
key is returned once. A [policy](../security/policies.md) rule that names the key by name
keeps applying to the new key. (Before enterprise 0.21.2 the allowed tools and Lean Mode setting were
not carried over — see [Security & updates](../security/updates.md).)

```
POST /v1/auth/keys/<key_id>/rotate
```

**Response:**

```json
{
  "key": {
    "id": "new-uuid-...",
    "name": "ci-pipeline",
    "key_prefix": "tdbk_z9y8",
    ...
  },
  "raw_key": "tdbk_z9y8x7w6..."
}
```

The old key stops working immediately. Update the key value in your integration
before the old one is rotated out.

---

## Set key role

Change the RBAC role of a key at any time. Role changes take effect immediately on the next request.

```
PUT /v1/auth/keys/<key_id>/role
```

**Request body:**

```json
{"role": "read"}
```

Valid values: `"read"`, `"readwrite"`, `"admin"`.

**Response:** the updated key record.

See [Role-based access control →](../security/rbac.md) for the full access matrix.

---

## Set a key's Lean Mode

**From 0.21.0.** Give a key a default [Lean Mode](../api/mcp.md#saving-tokens-lean-mode)
for its MCP calls, so an agent gets smaller results without changing its client. The
key's setting overrides `TDB_LEAN_MODE`; a call's own arguments still win.

```
PUT /v1/auth/keys/<key_id>/lean-mode
```

```bash
curl -X PUT http://localhost:8000/v1/auth/keys/<key_id>/lean-mode \
  -H "Authorization: Bearer <YOUR_ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"lean": true, "max_cell_chars": 500}'
```

Fields: `lean` (true/false), `format` (`objects` | `table`), `detail` (`full` |
`types` | `names` | `tables`), `max_cell_chars` (minimum 50, `0` = off). Send `{}` to
clear the key's setting. An invalid value is `400`. The setting is shown as
`lean_mode` on `GET /v1/auth/keys`. Requires `admin`.

## Set a key's attributes

**Enterprise, from 0.22.0.** Values that [column and row policies](../security/policies.md)
read as `:name` in a row filter: `rows: "region = :region"` gives each key only its own
region's rows.

```
PUT /v1/auth/keys/<key_id>/attributes
```

```bash
curl -X PUT http://localhost:8000/v1/auth/keys/<key_id>/attributes \
  -H "Authorization: Bearer <YOUR_ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"attributes": {"region": "EMEA"}}'
```

Up to 20 attributes; names start with a letter or `_` and hold letters, digits and `_` (at
most 64 characters); values are strings of at most 200 characters. Send `{"attributes": {}}` to clear them. Shown as `attributes` on
`GET /v1/auth/keys`. Requires `admin`. A key whose rule needs an attribute it does not
have is refused (`policy_missing_attribute`), never served unfiltered.

## Restrict MCP tools

Limit which MCP tools a key is allowed to call. From 0.19.0 the key's `tools/list`
shows only these tools. Pass `null` to restore unrestricted access (the default).

```
PUT /v1/auth/keys/<key_id>/tools
```

**Request body — restrict to specific tools:**

```json
{"tools": ["query_source", "schema_source"]}
```

**Request body — allow all tools:**

```json
{"tools": null}
```

Tool names must be exact. From 0.19.0 an unknown name (for example `query-source`)
is rejected with 400, naming it and the valid tools, and nothing is changed.

When a key with a tool allow-list calls a disallowed tool, TDB returns a tool-error response (HTTP 200 with `isError: true`) rather than a protocol-level error, so MCP clients handle it gracefully.

**Response:** the updated key record.

---

## Set per-key rate limit

Override the rate limit for a specific key. Pass `null` to reset to the server
default (`TDB_DEFAULT_RATE_LIMIT`).

```
PUT /v1/auth/keys/<key_id>/rate-limit
```

**Request body:**

```json
{"requests_per_minute": 30}
```

To reset to the server default:

```json
{"requests_per_minute": null}
```

**Response:** the updated key record.

---

## Rate limiting behaviour

Rate limiting uses a **fixed window** aligned to calendar minutes (e.g., 09:00:00–09:00:59).

When a rate limit is exceeded, TDB returns HTTP 429 with headers:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1716371460
Retry-After: 45
```

Rate limiting applies to DB-managed keys only. Static env keys and JWT tokens are
not rate-limited.

---

## Key lifecycle example

```bash
# 1. Create a read-only key for a data analyst
curl -X POST http://localhost:8000/v1/auth/keys \
  -H "Authorization: Bearer <YOUR_ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"name":"analyst-team","expires_in_days":90,"role":"read"}'

# Save the raw_key from the response

# 2. Create a readwrite key for a data pipeline that registers sources
curl -X POST http://localhost:8000/v1/auth/keys \
  -H "Authorization: Bearer <YOUR_ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"name":"etl-pipeline","role":"readwrite"}'

# 3. Restrict an MCP key to query and schema tools only
curl -X PUT http://localhost:8000/v1/auth/keys/<key_id>/tools \
  -H "Authorization: Bearer <YOUR_ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"tools":["query_source","schema_source"]}'

# 4. Use a read key to run a query
curl -X POST http://localhost:8000/v1/query \
  -H "Authorization: Bearer tdbk_..." \
  -H "Content-Type: application/json" \
  -d '{"source_id":"...","sql":"SELECT COUNT(*) FROM data"}'

# 5. Rotate after 90 days (inherits same role)
curl -X POST http://localhost:8000/v1/auth/keys/<key_id>/rotate \
  -H "Authorization: Bearer <YOUR_ADMIN_KEY>"

# 6. Revoke a key that's no longer needed
curl -X DELETE http://localhost:8000/v1/auth/keys/<key_id> \
  -H "Authorization: Bearer <YOUR_ADMIN_KEY>"
```

!!! note "Table name in the query above"
    The `FROM data` in step 4 is illustrative. The table name depends on the connector —
    CSV sources use `data`, database sources use the registered table name. See
    [Query API → Table name in queries](../api/query.md#table-name-in-queries).
