# OAuth 2.1 with PKCE

TDB Enterprise ships a built-in OAuth 2.1 authorization server. This lets
Claude Desktop, Cursor, ChatGPT, and any other MCP client that supports
OAuth 2.1 authenticate with TDB without manual token management.

**Implemented specifications:**

- [RFC 9728](https://www.rfc-editor.org/rfc/rfc9728) — OAuth 2.0 Protected Resource Metadata
- [RFC 8414](https://www.rfc-editor.org/rfc/rfc8414) — Authorization Server Metadata
- [RFC 7591](https://www.rfc-editor.org/rfc/rfc7591) — Dynamic Client Registration
- PKCE S256 (required — `plain` is rejected)

---

## Required environment variables

```bash
TDB_JWT_SECRET=<64-char hex>         # Required — signs the access tokens
TDB_ADMIN_USER=admin                 # Required — OAuth login credentials
TDB_ADMIN_PASSWORD=<password>        # Required

# Required if TDB sits behind a reverse proxy or non-localhost URL
TDB_SERVER_URL=https://tdb.yourcompany.com
```

If `TDB_SERVER_URL` is not set, TDB derives the base URL from the incoming request.
Set it explicitly when running behind nginx, Caddy, or a load balancer.

---

## Discovery endpoints

MCP clients discover the authorization server automatically. No manual configuration
is needed beyond pointing the client at the TDB base URL.

| Endpoint | Spec | Description |
|---|---|---|
| `GET /.well-known/oauth-protected-resource` | RFC 9728 | Resource server metadata |
| `GET /.well-known/oauth-authorization-server` | RFC 8414 | Authorization server metadata |

```bash
# Verify discovery is working
curl http://localhost:8000/.well-known/oauth-authorization-server | python -m json.tool
```

Expected output:

```json
{
  "issuer": "http://localhost:8000",
  "authorization_endpoint": "http://localhost:8000/oauth/authorize",
  "token_endpoint": "http://localhost:8000/oauth/token",
  "registration_endpoint": "http://localhost:8000/oauth/register",
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code"],
  "code_challenge_methods_supported": ["S256"],
  "token_endpoint_auth_methods_supported": ["none"]
}
```

---

## Connecting Claude Desktop

1. Open **Claude Desktop** → Settings → Developer → Edit Config (`claude_desktop_config.json`)

2. Add a TDB MCP server entry:

```json
{
  "mcpServers": {
    "tdb": {
      "url": "http://localhost:8000/v1/mcp"
    }
  }
}
```

3. Save and restart Claude Desktop.

4. On first use, Claude Desktop will detect the `WWW-Authenticate` header from TDB's
   MCP endpoint, open a browser window to `http://localhost:8000/oauth/authorize`,
   and prompt you to log in with `TDB_ADMIN_USER` / `TDB_ADMIN_PASSWORD`.

5. After authorising, Claude Desktop receives a JWT and stores it. All subsequent MCP
   calls use this token automatically. Tokens expire after `TDB_JWT_EXPIRE_MINUTES`
   (default 60 minutes); Claude Desktop re-authenticates automatically.

---

## Connecting Cursor

1. Open Cursor → Settings → MCP → Add Server

2. Set the server URL to `http://localhost:8000/v1/mcp`

3. Cursor performs the same OAuth 2.1 discovery and PKCE flow as Claude Desktop.

---

## Manual OAuth flow (testing)

You can walk through the full OAuth flow manually with curl:

**Step 1 — Register a client (dynamic registration):**

```bash
curl -X POST http://localhost:8000/oauth/register \
  -H "Content-Type: application/json" \
  -d '{
    "client_name": "My Test Client",
    "redirect_uris": ["http://localhost:9999/callback"]
  }'
```

Response:

```json
{
  "client_id": "abc123...",
  "client_name": "My Test Client",
  "redirect_uris": ["http://localhost:9999/callback"],
  "grant_types": ["authorization_code"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
```

**Step 2 — Generate a PKCE code verifier and challenge:**

```bash
# Generate a random code verifier (43–128 chars, URL-safe)
CODE_VERIFIER=$(python -c "
import secrets, base64
v = secrets.token_urlsafe(48)
print(v)
")

# Compute the S256 code challenge
CODE_CHALLENGE=$(python -c "
import hashlib, base64, sys
v = '$CODE_VERIFIER'
digest = hashlib.sha256(v.encode('ascii')).digest()
print(base64.urlsafe_b64encode(digest).rstrip(b'=').decode())
")

echo "Verifier: $CODE_VERIFIER"
echo "Challenge: $CODE_CHALLENGE"
```

**Step 3 — Open the authorization URL in a browser:**

```
http://localhost:8000/oauth/authorize
  ?response_type=code
  &client_id=abc123...
  &redirect_uri=http://localhost:9999/callback
  &code_challenge=<CODE_CHALLENGE>
  &code_challenge_method=S256
  &state=random-state-value
```

Log in with `TDB_ADMIN_USER` / `TDB_ADMIN_PASSWORD`. You'll be redirected to
`http://localhost:9999/callback?code=<auth_code>&state=random-state-value`.

**Step 4 — Exchange the code for an access token:**

```bash
curl -X POST http://localhost:8000/oauth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code" \
  -d "code=<auth_code>" \
  -d "client_id=abc123..." \
  -d "redirect_uri=http://localhost:9999/callback" \
  -d "code_verifier=$CODE_VERIFIER"
```

Response:

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "expires_in": 3600
}
```

**Step 5 — Use the access token on MCP:**

```bash
curl -X POST http://localhost:8000/v1/mcp \
  -H "Authorization: Bearer eyJhbGci..." \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "query_source",
      "arguments": {"sql": "SELECT COUNT(*) FROM data"}
    }
  }'
```

---

## Redirect URI restrictions

For security, TDB only allows redirect URIs that meet one of these criteria:

- `https://` — any HTTPS URI (required for production)
- `http://localhost` — loopback only (required for native app flows like Claude Desktop)

Plain `http://` to a non-localhost host is rejected. This follows
[OAuth 2.1 BCP](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics).

---

## Security notes

- PKCE is **required** for all authorization code flows — `plain` challenge method is rejected.
- Authorization codes expire after 10 minutes and are single-use.
- Code exchange is atomic (uses `BEGIN IMMEDIATE` transaction) — replay attacks are not possible.
- All login forms use constant-time credential comparison with a 200 ms brute-force throttle.
