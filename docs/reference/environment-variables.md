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

## Credential Encryption at Rest

*Enterprise only.*

| Variable | Default | Description |
|---|---|---|
| `TDB_ENCRYPTION_KEY` | *(unset — a key file is generated)* | Urlsafe-base64 32-byte key used to encrypt source `connection` blobs (including database passwords) in the registry database. Generate with `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`. When unset, TDB generates a key on first start and stores it beside the registry database as `.tdb_secret_key` (mode `0600`). |

!!! warning "The generated key defends against a stolen database, not a stolen host"
    Credentials are always encrypted at rest — there is no plaintext mode. But if
    you let TDB generate the key, that key sits in the same directory as the
    database it protects, so anything that copies the whole data directory
    (a volume snapshot, a backup tarball, `docker cp`) takes both.

    That default is still worth having: it covers the common case of a database
    file leaking on its own. For production, set `TDB_ENCRYPTION_KEY` from your
    secrets manager so the key never touches the data volume.

!!! danger "Losing the key means re-registering every source"
    There is no recovery path. If the registry database and its key are
    separated, TDB fails loudly on the affected sources rather than returning
    garbage — you must restore the matching key or re-register the sources.
    Back the key up **separately from the database**; a backup containing both
    provides no protection.

---

## CSV Source Security

| Variable | Default | Description |
|---|---|---|
| `TDB_ALLOWED_DATA_DIR` | *(unset — no restriction)* | Confines registered CSV `file_path` values to this directory. When set, a `file_path` that resolves (symlinks and `..` are expanded first) to a location **outside** this directory is rejected with `403` at register, schema, and query time. When unset, paths are accepted as-is. Applies to the **CSV connector in both editions**; database connectors take a table name (not a path) and are unaffected. **The Docker images set this to `/data` by default**, so the bundled deployment is confined out of the box. For a bare `tdb serve`, set it to the directory holding your CSVs. |

!!! warning "Set this for any non-Docker CSV deployment exposed beyond localhost"
    Without `TDB_ALLOWED_DATA_DIR`, the CSV connector reads any path the server
    process can access — a client with the API key could register
    `file_path: /etc/passwd` and read it back. The Docker images are confined to
    `/data`; a source-installed `tdb serve` is **not** until you set this.

---

## License

TDB Enterprise requires a valid, signed license to serve data. Trial images have the
license baked in, so you normally set nothing here. See [Licensing & expiry](../getting-started/installation.md#licensing-expiry).

| Variable | Default | Description |
|---|---|---|
| `TDB_LICENSE` | *(empty)* | A signed license token. Takes precedence over the baked file — use it to drop in a renewed license without rebuilding the image. |
| `TDB_LICENSE_FILE` | `/app/license.jwt` | Path to the license token baked into the image. Read only when `TDB_LICENSE` is unset. |

When the license is missing, invalid, or expired, the server keeps running and
`/health` stays `200`, but all data/API requests return `403 license_expired`.

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
| `TDB_MAX_ROWS` | `1000` | Maximum rows any single query response may contain (**enterprise only** — community is fixed at 1,000). A request whose `limit` exceeds this is rejected with `400`; a result larger than `limit` is cut and flagged `truncated: true`, whatever the SQL's own `LIMIT` says. Every returned row is held in memory, so raise this deliberately and size RAM to match. An unparseable or non-positive value falls back to `1000` — a typo never means "unlimited". |
| `TDB_QUERY_TIMEOUT` | `30` | Seconds a single query may run at the source before it is cancelled (**enterprise only**). Enforced by the database itself — `statement_timeout` (PostgreSQL), `max_execution_time` (MySQL), the ODBC query timeout (SQL Server), `STATEMENT_TIMEOUT_IN_SECONDS` (Snowflake) — so the engine stops doing the work, rather than TDB merely stopping waiting for it. A cancelled query returns `504` and is recorded in the audit log as a `query_timeout` denial. Set `0` to disable, which is the only way to get pre-0.3.0 behaviour. **CSV/DuckDB has no server to ask and is not interruptible.** An unparseable or negative value falls back to `30` — as with `TDB_MAX_ROWS`, a typo never removes a limit. |
| `TDB_AUDIT_ROTATE_MAX_BYTES` | `0` | Seal the audit log into a timestamped segment once the live file exceeds this many bytes (**enterprise only**). `0` disables rotation, which is pre-0.4.0 behaviour: the log grows without bound. TDB never *deletes* an entry — rotation decides when the live log is sealed, not when anything is dropped, and each sealed segment stays independently verifiable. Pruning old segments is the operator's decision. An unparseable or negative value falls back to `0`. See [audit retention](../security/audit.md#retention-sealed-segments). |
| `TDB_AUDIT_ROTATE_MAX_DAYS` | `0` | Seal the audit log once its oldest entry is older than this many days (**enterprise only**). Useful where retention is a calendar policy rather than a size budget. May be combined with `TDB_AUDIT_ROTATE_MAX_BYTES`; whichever threshold is crossed first seals the log. `0` disables. |

---

## Splunk HEC Export

| Variable | Default | Description |
|---|---|---|
| `TDB_SPLUNK_HEC_URL` | *(empty — disabled)* | Full URL of the Splunk HTTP Event Collector endpoint. Example: `https://splunk.corp.com:8088/services/collector/event`. If not set, `POST /v1/audit/export` returns `{"disabled": true}`. |
| `TDB_SPLUNK_HEC_TOKEN` | *(required if URL set)* | Splunk HEC authentication token. Generate one in the Splunk UI under Settings → Data Inputs → HTTP Event Collector. |
| `TDB_SPLUNK_INDEX` | *(HEC default)* | Splunk index to write events to. Omit to use the index configured on the HEC token. |
| `TDB_SPLUNK_SOURCETYPE` | `tdb:audit` | Splunk sourcetype assigned to exported events. |
| `TDB_SPLUNK_VERIFY_TLS` | `true` | Set to `false` to disable TLS certificate verification. Only use in development with self-signed certs. |
| `TDB_S3_BUCKET` | — | Bucket for the [S3 audit archive](../integrations/s3.md) (**enterprise only**). Unset disables the exporter, which is the default. |
| `TDB_S3_PREFIX` | `tdb-audit` | Key prefix within the bucket. |
| `TDB_S3_REGION` | SDK default | AWS region. Credentials come from the standard AWS chain — TDB takes no access keys of its own. |
| `TDB_S3_ENDPOINT_URL` | — | Override for S3-compatible storage (MinIO, Ceph, R2). |
| `TDB_S3_SSE` | — | Server-side encryption: `AES256` or `aws:kms`. |
| `TDB_S3_SSE_KMS_KEY_ID` | — | KMS key ID, used when `TDB_S3_SSE` is `aws:kms`. |

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
- [ ] For CSV sources on a bare `tdb serve` (non-Docker), `TDB_ALLOWED_DATA_DIR` is set to the directory holding your CSVs, so `file_path` can't escape to arbitrary files
- [ ] DB-managed API keys are created with the minimum required role (`read` for read-only integrations)
