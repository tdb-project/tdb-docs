# JWT Authentication

TDB issues short-lived JWT access tokens via a username/password login endpoint.
Tokens are signed HS256 using `TDB_JWT_SECRET`.

---

## Required environment variables

```bash
TDB_JWT_SECRET=<64-char hex>        # Required — server will return 503 without it
TDB_ADMIN_USER=admin                # Required — login username
TDB_ADMIN_PASSWORD=<password>       # Required — login password
TDB_JWT_EXPIRE_MINUTES=60           # Optional — token lifetime (default: 60 minutes)
```

Generate a secure secret:

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

!!! warning "Single admin user"
    The current release supports one admin identity (`TDB_ADMIN_USER` / `TDB_ADMIN_PASSWORD`).
    Multi-user support with individual logins is planned for a future release. For
    role-scoped access today, use [DB-managed API keys with RBAC](../security/rbac.md).

---

## Issue a token

```
POST /v1/auth/token
```

**Request body:**

```json
{
  "username": "admin",
  "password": "your-strong-password"
}
```

**Response:**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "expires_in": 3600
}
```

| Field | Description |
|---|---|
| `access_token` | JWT to use as `Authorization: Bearer <token>` |
| `token_type` | Always `"bearer"` |
| `expires_in` | Token lifetime in seconds |

---

## Use the token

Include the JWT in the `Authorization` header on every subsequent request:

```bash
curl http://localhost:8000/v1/sources \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

---

## Full login + query example

```bash
# 1. Log in
TOKEN=$(curl -s -X POST http://localhost:8000/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"your-password"}' \
  | python -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# 2. List sources
curl http://localhost:8000/v1/sources \
  -H "Authorization: Bearer $TOKEN"

# 3. Query
curl -X POST http://localhost:8000/v1/query \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "<SOURCE_ID>",
    "sql": "SELECT * FROM data LIMIT 10"
  }'
```

!!! note "Table name in the query above"
    The `FROM data` here is illustrative. The table name depends on the connector —
    CSV sources use `data`, database sources use the registered table name. See
    [Query API → Table name in queries](../api/query.md#table-name-in-queries).

---

## Token contents

The JWT payload contains:

```json
{
  "sub": "admin",
  "role": "admin",
  "exp": 1716375600,
  "iat": 1716372000
}
```

JWT tokens currently carry the `"admin"` role. For fine-grained, role-scoped access,
use [DB-managed API keys with RBAC](../security/rbac.md) (read / readwrite / admin);
per-user JWT roles are planned for a future release.

---

## Token expiry and refresh

TDB does not yet implement token refresh. When a token expires, re-authenticate via
`POST /v1/auth/token`. The `TDB_JWT_EXPIRE_MINUTES` default is 60 minutes — increase
it for longer-lived sessions or automate re-login in your integration.

---

## Error responses

| Scenario | HTTP | Body |
|---|---|---|
| Wrong username or password | 401 | `{"detail":"Invalid credentials"}` |
| `TDB_JWT_SECRET` not set | 503 | `{"detail":"JWT authentication is not configured. Set TDB_JWT_SECRET, TDB_ADMIN_USER, and TDB_ADMIN_PASSWORD."}` |
| Expired token used | 401 | `{"detail":"Not authenticated"}` |
