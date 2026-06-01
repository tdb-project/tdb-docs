# API Key Management

DB-managed API keys are stored as SHA-256 hashes in the TDB registry. The raw key
value is shown **once** — at creation or rotation time — and cannot be retrieved
afterwards. Keep it safe.

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
  "rate_limit": 120
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string (1–100 chars) | Yes | Human-readable label |
| `expires_in_days` | integer > 0 | No | Key auto-expires after this many days. Omit for no expiry. |
| `rate_limit` | integer > 0 | No | Requests per minute. Omit to use the server default (`TDB_DEFAULT_RATE_LIMIT`, default 60). |
| `role` | `"read"`, `"readwrite"`, or `"admin"` | No | RBAC role. Default: `"admin"`. See [RBAC →](../security/rbac.md). |

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
expiry, and rate limit. The new raw key is returned once.

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

## Restrict MCP tools

Limit which MCP tools a key is allowed to call. Pass `null` to restore unrestricted access (the default).

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
