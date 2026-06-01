# Environment Variables

All TDB configuration is done through environment variables. Set them in a `.env`
file in the project directory, or pass them directly to the process.

---

## Core

| Variable | Default | Description |
|---|---|---|
| `TDB_API_KEYS` | `dev-insecure-key-change-me` | Comma-separated list of static API keys. Evaluated by constant-time comparison on every request. **Change this before exposing TDB to any network.** |
| `TDB_LOG_LEVEL` | `INFO` | Log verbosity. One of: `DEBUG`, `INFO`, `WARNING`, `ERROR`. |
| `TDB_LOG_FILE` | `tdb_audit.jsonl` | Path to the NDJSON audit log. Relative paths are resolved from the working directory. |
| `TDB_REGISTRY_DB` | `data/tdb_registry.db` | Path to the SQLite registry database. Created automatically on first startup. |

---

## JWT Authentication

| Variable | Default | Description |
|---|---|---|
| `TDB_JWT_SECRET` | *(required)* | HMAC secret for signing JWT tokens. Generate with: `python -c "import secrets; print(secrets.token_hex(32))"`. TDB returns HTTP 503 if this is not set and JWT/OAuth is used. |
| `TDB_JWT_EXPIRE_MINUTES` | `60` | JWT token lifetime in minutes. Increase for longer-lived sessions. |
| `TDB_ADMIN_USER` | *(required)* | Admin login username. Used by `POST /v1/auth/token` and the OAuth authorize form. |
| `TDB_ADMIN_PASSWORD` | *(required)* | Admin login password. |

---

## Rate Limiting

| Variable | Default | Description |
|---|---|---|
| `TDB_DEFAULT_RATE_LIMIT` | `60` | Default requests per minute for DB-managed API keys. Per-key overrides take precedence. Static env keys and JWTs are not rate-limited. |

---

## CORS

| Variable | Default | Description |
|---|---|---|
| `TDB_CORS_ORIGINS` | *(empty — disabled)* | Comma-separated list of allowed origins. Empty means CORS middleware is not added (safe default for self-hosted deployments). Use `*` to allow all origins (dev only). |
| `TDB_CORS_ALLOW_CREDENTIALS` | `false` | Set to `true` to include `Access-Control-Allow-Credentials: true`. Do not combine with `TDB_CORS_ORIGINS=*` — browsers reject this combination. |

---

## OAuth / Reverse Proxy

| Variable | Default | Description |
|---|---|---|
| `TDB_SERVER_URL` | *(derived from request)* | Public base URL of the TDB server. Required when running behind a reverse proxy so OAuth discovery endpoints return correct URLs. Example: `https://tdb.yourcompany.com`. |

---

## Views (YAML-defined queries)

| Variable | Default | Description |
|---|---|---|
| `TDB_VIEWS_DIR` | *(empty — disabled)* | Path to a directory containing YAML view definition files. If not set, the views feature is disabled (safe default). See [YAML views guide](../api/views.md). |

---

## Schema Cache

| Variable | Default | Description |
|---|---|---|
| `TDB_SCHEMA_CACHE_TTL` | `300` | Time-to-live for the in-process schema cache, in seconds. Set to `0` to disable caching. The cache is keyed by source ID and invalidated automatically when a source is deleted. |

---

## Splunk HEC Export

| Variable | Default | Description |
|---|---|---|
| `TDB_SPLUNK_HEC_URL` | *(empty — disabled)* | Full URL of the Splunk HTTP Event Collector endpoint. Example: `https://splunk.corp.com:8088/services/collector/event`. If not set, `POST /v1/audit/export` returns `{"disabled": true}`. |
| `TDB_SPLUNK_HEC_TOKEN` | *(required if URL set)* | Splunk HEC authentication token. Generate one in the Splunk UI under Settings → Data Inputs → HTTP Event Collector. |
| `TDB_SPLUNK_INDEX` | *(HEC default)* | Splunk index to write events to. Omit to use the index configured on the HEC token. |
| `TDB_SPLUNK_SOURCETYPE` | `tdb:audit` | Splunk sourcetype assigned to exported events. |
| `TDB_SPLUNK_VERIFY_TLS` | `true` | Set to `false` to disable TLS certificate verification. Only use in development with self-signed certs. |

---

## Minimal production `.env`

```bash
# Core
TDB_API_KEYS=your-strong-bootstrap-key

# JWT + OAuth (required for Claude Desktop / Cursor)
TDB_JWT_SECRET=<output of: python -c "import secrets; print(secrets.token_hex(32))">
TDB_ADMIN_USER=admin
TDB_ADMIN_PASSWORD=your-strong-admin-password

# Optional tuning
TDB_LOG_LEVEL=INFO
TDB_DEFAULT_RATE_LIMIT=60
TDB_SCHEMA_CACHE_TTL=300

# If behind a reverse proxy
TDB_SERVER_URL=https://tdb.yourcompany.com

# If your frontend needs CORS
TDB_CORS_ORIGINS=https://app.yourcompany.com

# Splunk audit export (optional)
# TDB_SPLUNK_HEC_URL=https://splunk.corp.com:8088/services/collector/event
# TDB_SPLUNK_HEC_TOKEN=your-hec-token
# TDB_SPLUNK_INDEX=tdb_audit

# YAML views (optional)
# TDB_VIEWS_DIR=/etc/tdb/views
```

---

## Security checklist

- [ ] `TDB_API_KEYS` is set to a secret value (not the default `dev-insecure-key-change-me`)
- [ ] `TDB_JWT_SECRET` is at least 32 random bytes (64 hex chars)
- [ ] `TDB_ADMIN_PASSWORD` is a strong password (not guessable)
- [ ] `TDB_CORS_ORIGINS` is set to specific origins, not `*`, if credentials are involved
- [ ] `TDB_LOG_FILE` path is writable and backed up (it's your tamper-evident audit trail)
- [ ] `TDB_REGISTRY_DB` path is on persistent storage (source registrations are stored here)
- [ ] DB-managed API keys are created with the minimum required role (`read` for read-only integrations)
