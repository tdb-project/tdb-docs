# Authentication Overview

TDB Enterprise supports three authentication mechanisms. They are evaluated in
waterfall order on every authenticated request:

```
Incoming request
      │
      ▼
1. JWT token?       ──yes──> verify signature → allow / 401
      │ no
      ▼
2. DB-managed key?  ──yes──> hash lookup → check expiry + revoke → allow / 401
      │ no                                       │
      │                                          ▼
      │                                   rate-limit check → 429 if exceeded
      ▼
3. Static env key?  ──yes──> constant-time compare → allow / 401
      │ no
      ▼
      401 Unauthorized
```

All three mechanisms use the same header format:

```
Authorization: Bearer <token>
```

---

## Mechanism 1 — Static API keys (env var)

The simplest option. Set `TDB_API_KEYS` to a comma-separated list of keys:

```bash
TDB_API_KEYS=key-one,key-two,key-three
```

These keys are compared using constant-time comparison (`hmac.compare_digest`) on
every request. They are not stored in the database and do not appear in the key
management endpoints. They are **not** rate-limited — the rate limiter applies only
to DB-managed keys.

Use static keys for:
- Initial setup and testing
- Machine-to-machine integrations where key rotation isn't required
- The bootstrap admin key before you issue DB-managed keys

---

## Mechanism 2 — DB-managed API keys

Keys created via `POST /v1/auth/keys`. Stored as SHA-256 hashes in the SQLite
registry. The raw key is shown **once** at creation time and cannot be retrieved.

DB-managed keys support:
- Per-key rate limiting (requests per minute)
- Expiry (`expires_in_days`)
- Rotation (revoke + re-issue atomically, same name)
- Revocation
- `last_used_at` tracking

See [API Keys →](api-keys.md)

---

## Mechanism 3 — JWT tokens

Short-lived tokens issued by `POST /v1/auth/token` (username/password login).
Signed with HS256 using `TDB_JWT_SECRET`. Default expiry is 60 minutes
(configurable via `TDB_JWT_EXPIRE_MINUTES`).

JWTs are suitable for:
- Interactive sessions
- Tools that support OAuth 2.1 / token refresh flows

See [JWT →](jwt.md)

---

## OAuth 2.1 with PKCE (for MCP clients)

Claude Desktop, Cursor, and other MCP clients that support OAuth 2.1 can authenticate
against TDB's built-in authorization server. TDB implements:

- RFC 9728 — OAuth 2.0 Protected Resource Metadata
- RFC 8414 — Authorization Server Metadata
- RFC 7591 — Dynamic Client Registration
- PKCE (S256 only) — required for all authorization code flows

The result is a JWT access token used on subsequent MCP requests.

See [OAuth 2.1 →](oauth.md)

---

## Which mechanism should I use?

| Use case | Recommended |
|---|---|
| First-time setup, local testing | Static env key |
| Production REST API integrations | DB-managed key with expiry + rate limit |
| Multi-user team access | JWT (one login per user) |
| Claude Desktop / Cursor / ChatGPT | OAuth 2.1 PKCE |
| CI/CD pipelines | DB-managed key (rotate on schedule) |

---

## Unauthenticated endpoints

The following endpoints do **not** require authentication:

| Endpoint | Purpose |
|---|---|
| `GET /` | Root info |
| `GET /health` | Liveness probe |
| `GET /docs` | Swagger UI |
| `GET /redoc` | ReDoc UI |
| `POST /v1/mcp` (`initialize` method only) | MCP handshake |
| `GET /.well-known/oauth-protected-resource` | OAuth resource metadata |
| `GET /.well-known/oauth-authorization-server` | OAuth AS metadata |
| `GET /oauth/authorize` | OAuth login form |
| `POST /oauth/register` | OAuth dynamic client registration |
