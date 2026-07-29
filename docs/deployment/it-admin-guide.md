# TDB Enterprise — IT Administrator Deployment Guide

!!! info "TDB Enterprise"
    This guide is written for the IT or platform team that will deploy and operate
    the **commercial enterprise edition** on your own infrastructure. For a faster
    first run, start with [Installation](../getting-started/installation.md) and the
    [Quickstart](../getting-started/quickstart.md). For the free, open-source CSV
    edition, see the [Community Edition quickstart](../getting-started/community.md).

---

## 0. What you are deploying

TDB Enterprise is a single, self-contained **Docker image**. It runs one
FastAPI/uvicorn process that exposes:

- a **REST** query API and source registry (`/v1/...`),
- an **MCP** server for AI tools (`/v1/mcp`),
- **OpenAPI/Swagger** docs (`/docs`),
- operational endpoints: `/health`, `/metrics` (Prometheus), `/` (version banner).

It is **read-only middleware** in front of *your* data sources (CSV files and, in
enterprise, PostgreSQL / MySQL / SQL Server / Snowflake). It does not copy or
warehouse your data — it proxies governed `SELECT`-only queries and writes an
audit record for each one. Access is gated by a signed **license** (see the
build/licensing playbook).

There is **one container**; there is no separate database server to run. State
lives in two files on a mounted volume (registry DB + audit log).

---

## 1. Pre-installation checklist

### 1.1 Operating system & runtime

| Requirement | Supported / recommended |
|---|---|
| Host OS | Any 64-bit Linux that runs a current container engine (Ubuntu 22.04/24.04 LTS, RHEL/Rocky 8–9, Debian 12 verified in test). macOS/Windows via Docker Desktop work for evaluation. |
| CPU arch | **Production / commercial image: `linux/amd64` (x86-64).** Trial and evaluation images are multi-arch (`linux/amd64` + `linux/arm64`), so they run natively on Apple Silicon Macs — `docker pull` selects your architecture automatically. If you need arm64 for a *production* deployment (e.g. AWS Graviton), tell us: it is a build-pipeline change, not a code change. |
| Container engine | Docker Engine **≥ 20.10** (BuildKit era) or any OCI-compliant runtime (Podman ≥ 4 works). |
| Compose (optional) | Docker Compose **v2** if you deploy via compose file. |
| Kubernetes (optional) | Any conformant cluster **≥ 1.25**. We ship **no Helm chart or manifests** today — a reference Deployment is in [§4.3](#43-kubernetes-reference-only). |

The image is based on `python:3.12-slim`. The only system library baked in is
`unixodbc` (so the SQL Server connector can load). To actually *connect* to SQL
Server you additionally need the **Microsoft ODBC Driver 18**, which is **not**
in the image — see [§5.4](#54-sql-server-driver).

### 1.2 Hardware sizing (starting point — not an SLA)

> **Starting points, not an SLA.** These are conservative recommendations derived from
> the deployment footprint — they are **not** benchmarked figures, and we do not publish
> throughput numbers we have not measured. Size against your own workload:
> [§8](#8-performance-capacity) gives a repeatable method.

| Profile | vCPU | RAM | Disk |
|---|---|---|---|
| Evaluation / single user | 1 | 1 GB | 5 GB |
| Small team (default) | 2 | 2–4 GB | 20 GB |
| Heavier CSV / concurrent | 4 | 8 GB | 50 GB+ |

Notes that drive sizing:
- The image is **~520 MB on disk / ~130 MB compressed**.
- **CSV** queries run in-process via **DuckDB**, which is memory- and CPU-hungry on
  large files (it reads the file per query — see [§6](#6-security-model-what-an-admin-must-know)). RAM should
  comfortably exceed your largest CSV working set. Database connectors stream from the
  remote engine and are far lighter on the TDB host.
- Disk grows mainly with the **audit log** (one NDJSON line per successful query,
  append-only). Plan rotation/retention — see [§5.3](#53-logging-audit).

### 1.3 Network & ports

- The container listens on **TCP 8000** (HTTP) inside the container. Publish/remap it
  with `-p` (e.g. `-p 8000:8000`). **The server speaks plain HTTP** — terminate TLS in
  front of it (see [§7](#7-tls-ssl)).
- **Egress** the host must allow: to each registered data source (e.g. Postgres 5432,
  MySQL 3306, SQL Server 1433, Snowflake 443); to your private registry if pulling the
  image; and to your SIEM endpoint if you enable the Splunk exporter (HEC over 443/8088).
- **No inbound** connections are required other than to port 8000 from your API/MCP
  clients (and the reverse proxy).

### 1.4 Getting the image

Two channels — see [Installation](../getting-started/installation.md) for the
step-by-step version:

**A. Customer-scoped private registry (default).** You receive a pull token scoped to
your own repo path:

```bash
docker login registry.tdb.jiracorp.co.in        # use the token we provide
docker pull registry.tdb.jiracorp.co.in/tdb-customers/<your-slug>/tdb-enterprise:<edition>-<expiry>
```

**B. Air-gapped tarball.** For locked-down environments we deliver a `.tar.gz`:

```bash
docker load < tdb-enterprise-<edition>-<expiry>.tar.gz
docker images tdb-enterprise        # confirm the tag is now present
```

> **License:** trial images have a signed, expiring license **baked in** — nothing to
> configure. Commercial deployments may instead receive a token-less image and supply
> the license at runtime via `-e TDB_LICENSE=...`. See [§5.5](#55-license).

---

## 2. Quick deploy (single container)

The minimum viable run. **Generate real secrets — do not copy the placeholders.**

```bash
docker run -d --name tdb -p 8000:8000 \
  -e TDB_API_KEYS="$(python3 -c 'import secrets;print(secrets.token_urlsafe(32))')" \
  -e TDB_JWT_SECRET="$(python3 -c 'import secrets;print(secrets.token_hex(32))')" \
  -e TDB_ADMIN_USER=admin \
  -e TDB_ADMIN_PASSWORD='<choose-a-strong-password>' \
  -e TDB_LOG_FILE=/app/logs/tdb_audit.jsonl \
  -v "$(pwd)/tdb-data:/app/data" \
  -v "$(pwd)/tdb-logs:/app/logs" \
  -v "$(pwd)/csv:/data:ro" \
  <image>
```

What each mount is for:
- `/app/data` — **persistent state** (SQLite registry DB + API-key store). **Must be
  writable.** Do **not** mount it read-only: the registry DB is created here at
  startup, and a read-only mount makes the container exit at migration
  (`unable to open database file`).
- `/app/logs` — the audit log (if you point `TDB_LOG_FILE` here). Persist it.
- `/data` (read-only) — where you place CSV inputs. The image confines registered CSV
  paths to this directory by default (`TDB_ALLOWED_DATA_DIR=/data`). Mounting it `:ro`
  is correct and recommended — it's a *separate* path from `/app/data`.

Confirm it came up:

```bash
docker logs tdb | grep license_ok      # INFO: license_ok customer=... edition=... days_left=...
curl -fsS http://localhost:8000/health  # {"status":"ok",...}
```

The container runs as a **non-root** user (`tdb`). Keep it that way; do not add
`--user root`.

---

## 3. Configuration reference (environment variables)

All configuration is via environment variables — there is no config file to edit
inside the image. Defaults in parentheses.

### 3.1 Core / required

| Variable | Default | Purpose |
|---|---|---|
| `TDB_API_KEYS` | `dev-insecure-key-change-me` | Comma-separated static API keys (`Authorization: Bearer <key>`). **Always override.** |
| `TDB_JWT_SECRET` | *(empty)* | HMAC secret for JWT auth. **Required** for JWT/OAuth login flows. |
| `TDB_ADMIN_USER` / `TDB_ADMIN_PASSWORD` | *(empty)* | Admin credentials for the login/OAuth consent flow. |
| `TDB_LICENSE` | *(empty)* | Runtime license token. **Takes precedence** over the baked file (commercial/renewals). |
| `TDB_LICENSE_FILE` | `/app/license.jwt` | Path to the baked license token. |

### 3.2 Data, storage & logging

| Variable | Default | Purpose |
|---|---|---|
| `TDB_REGISTRY_DB` | `data/tdb_registry.db` | SQLite path for the source registry + API keys. |
| `TDB_ALLOWED_DATA_DIR` | `/data` *(set in image)* | Confines registered **CSV** file paths to this dir (symlinks/`..` resolved). Empty = no restriction. DB connectors unaffected. |
| `TDB_LOG_FILE` | `tdb_audit.jsonl` | Audit log path (NDJSON). Point at a persisted volume, e.g. `/app/logs/...`. |
| `TDB_LOG_LEVEL` | `INFO` | `DEBUG`/`INFO`/`WARNING`/`ERROR`. |
| `TDB_VIEWS_DIR` | *(unset)* | Directory of YAML named-view definitions. Unset = views disabled. |
| `TDB_SCHEMA_CACHE_TTL` | `300` | Schema cache TTL (seconds). `0` disables caching. |

### 3.3 Access, auth & API behaviour

| Variable | Default | Purpose |
|---|---|---|
| `TDB_JWT_EXPIRE_MINUTES` | `60` | JWT lifetime. |
| `TDB_DEFAULT_RATE_LIMIT` | `60` | Default requests/min per API key (overridable per key). |
| `TDB_CORS_ORIGINS` | *(empty = off)* | Comma-separated allowed origins. `*` = all (dev only). |
| `TDB_CORS_ALLOW_CREDENTIALS` | `false` | `Access-Control-Allow-Credentials`. Don't combine with `*`. |
| `TDB_SERVER_URL` | *(derived)* | Public base URL TDB advertises as the OAuth **issuer**. Set to your external HTTPS URL behind a proxy. |

### 3.4 SIEM export (optional — Splunk)

| Variable | Default | Purpose |
|---|---|---|
| `TDB_SPLUNK_HEC_URL` | *(unset = exporter off)* | Splunk HEC endpoint, e.g. `https://splunk.corp/services/collector/event`. |
| `TDB_SPLUNK_HEC_TOKEN` | *(empty)* | HEC token (a **secret** — see [§6.3](#63-where-runtime-secrets-live)). |
| `TDB_SPLUNK_INDEX` | *(unset)* | Target index. |
| `TDB_SPLUNK_SOURCETYPE` | `tdb:audit` | Event sourcetype. |
| `TDB_SPLUNK_VERIFY_TLS` | `true` | Set `false` only for self-signed lab Splunk. |

### 3.5 Advanced

| Variable | Default | Purpose |
|---|---|---|
| `TDB_LICENSE_PUBKEY_FILE` | *(baked key)* | Override the license verification public key path. Normally leave unset. |

---

## 4. Deployment topologies

### 4.1 Docker Compose (recommended for a single host)

```yaml
# docker-compose.yml
services:
  tdb:
    image: registry.tdb.jiracorp.co.in/tdb-customers/<slug>/tdb-enterprise:<edition>-<expiry>
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      TDB_API_KEYS: ${TDB_API_KEYS}
      TDB_JWT_SECRET: ${TDB_JWT_SECRET}
      TDB_ADMIN_USER: ${TDB_ADMIN_USER}
      TDB_ADMIN_PASSWORD: ${TDB_ADMIN_PASSWORD}
      TDB_LOG_FILE: /app/logs/tdb_audit.jsonl
      TDB_SERVER_URL: https://tdb.yourcompany.com
    volumes:
      - tdb-data:/app/data
      - tdb-logs:/app/logs
      - ./csv:/data:ro
volumes:
  tdb-data:
  tdb-logs:
```

Keep secrets in a `.env` file (referenced as `${...}`) with `chmod 600`, or use your
platform's secret store — see [§6.3](#63-where-runtime-secrets-live).

### 4.2 Single `docker run`

See [§2](#2-quick-deploy-single-container). Add `--restart unless-stopped` for
production. The image already declares a `HEALTHCHECK` against `/health`, so
`docker ps` / orchestrators see container health automatically.

### 4.3 Kubernetes (reference only)

> **Reference only — not a supported artifact.** No official Helm chart or manifests
> ship today. The manifest below is a minimal starting point to adapt to your cluster
> conventions; we do not support it as a deployment artifact. The supported single-host
> path is Docker Compose ([§4.1](#41-docker-compose-recommended-for-a-single-host)).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: tdb-enterprise }
spec:
  replicas: 1                       # see scaling caveat below
  selector: { matchLabels: { app: tdb } }
  template:
    metadata: { labels: { app: tdb } }
    spec:
      securityContext: { runAsNonRoot: true }
      containers:
        - name: tdb
          image: <image>
          ports: [ { containerPort: 8000 } ]
          envFrom: [ { secretRef: { name: tdb-secrets } } ]
          volumeMounts:
            - { name: data, mountPath: /app/data }
            - { name: logs, mountPath: /app/logs }
          readinessProbe: { httpGet: { path: /health, port: 8000 } }
          livenessProbe:  { httpGet: { path: /health, port: 8000 } }
      volumes:
        - { name: data, persistentVolumeClaim: { claimName: tdb-data } }
        - { name: logs, persistentVolumeClaim: { claimName: tdb-logs } }
```

> **Scaling caveat:** state (registry DB + audit log) is **local SQLite + a local
> file**. Do **not** run multiple replicas against the same volume — SQLite and the
> append-only hash-chained log assume a single writer. Run **one replica** with a
> `Recreate` strategy and an RWO volume. Horizontal scale-out is **not supported
> today** — this is a property of the state layer, not of the packaging, so no chart
> or manifest change would lift it.

---

## 5. Custom configuration tasks

### 5.1 Registering a data source

Sources are registered at runtime over the REST API (admin key), not in a config file.
Example (PostgreSQL):

```bash
curl -fsS -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer $ADMIN_KEY" -H 'Content-Type: application/json' \
  -d '{
        "name": "orders_pg",
        "source_type": "postgres",
        "connection": {
          "host": "db.internal", "port": 5432, "dbname": "sales",
          "user": "tdb_readonly", "password": "***",
          "schema": "public", "table": "orders"
        }
      }'
```

- One registered source maps to **one table** (matches the CSV model). Need several
  tables? Register several sources, or use named views.
- **Read-only is enforced at the engine level**, not just by SQL parsing: Postgres sets
  `connection.read_only = True`; MySQL uses a read-only transaction; SQL Server always
  rolls back (autocommit off) and is fronted by the SELECT-only validator.
- **Best practice: give TDB a least-privilege, read-only DB login** (e.g. Postgres role
  with `SELECT` only; SQL Server `db_datareader`). This is your real guarantee — TDB's
  enforcement is defence-in-depth on top of it.

> **Security:** the `connection` block (including the password) is **persisted**,
> encrypted at rest — read [§6.2](#62-where-source-credentials-live) for the key
> model before registering anything against production.

### 5.2 Named views (optional)

Set `TDB_VIEWS_DIR=/app/views` and mount a directory of YAML view definitions to expose
curated, reusable queries. Leave unset to disable the feature.

### 5.3 Logging & audit

- The audit log is **append-only NDJSON** at `TDB_LOG_FILE`, **hash-chained** (each entry
  carries `seq` + `prev_hash` + SHA-256 `hash`) so tampering is detectable.
- **Both outcomes are recorded.** A successful query writes `event: "query"`; a refusal
  writes `event: "denied"` with the attempted `action` and a machine-readable `reason`
  (e.g. `invalid_api_key`, `sql_validation_failed`). Rejected access attempts are
  therefore first-class audit evidence, not just application-log noise — and because
  denials are signed into the *same* chain, deleting an inconvenient one breaks
  `/v1/audit/verify`.
- Verify integrity any time (admin role):
  ```bash
  curl -fsS -H "Authorization: Bearer $ADMIN_KEY" http://localhost:8000/v1/audit/verify
  # {"valid": true, "entries": N}
  ```
- **Rotation/retention is your responsibility.** Rotating the file starts a fresh chain;
  archive rotated segments if you need continuous verifiability. Optionally stream to a
  SIEM via the Splunk exporter ([§3.4](#34-siem-export-optional-splunk)).
> Denial auditing applies to **both editions**. Entry shapes and the verification
> endpoint are documented in full at [Audit Log](../security/audit.md).

### 5.4 SQL Server driver

The image ships `unixodbc` but **not** the Microsoft ODBC Driver 18. If you register a
SQL Server source you must provide the driver. The cleanest path is to install it in a
thin layer on top of our image:

```dockerfile
FROM <tdb-enterprise-image>
USER root
RUN curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /usr/share/keyrings/microsoft.gpg \
 && curl -fsSL https://packages.microsoft.com/config/debian/12/prod.list > /etc/apt/sources.list.d/mssql.list \
 && apt-get update && ACCEPT_EULA=Y apt-get install -y --no-install-recommends msodbcsql18 \
 && rm -rf /var/lib/apt/lists/*
USER tdb
```

The connector defaults to `Encrypt=yes`. For self-signed SQL Server certs set
`"trust_server_certificate": true` in the source's connection block (lab only).

### 5.5 License

- **Trial:** baked in and expiring. On expiry the container keeps `/health`, `/`, and
  `/metrics` green but returns **HTTP 403** on every data route. Renew by re-pulling a
  new image.
- **Commercial:** supply/rotate the token at runtime with `-e TDB_LICENSE=<token>` and
  restart — no rebuild. See
  [Installation → License](../getting-started/installation.md) and the
  `TDB_LICENSE` / `TDB_LICENSE_FILE` entries in the
  [environment variable reference](../reference/environment-variables.md).
- To extend a trial or renew a commercial license, contact
  [hello@tdb.jiracorp.co.in](mailto:hello@tdb.jiracorp.co.in).

---

## 6. Security model — what an admin must know

### 6.1 What TDB stores, and where

| Data | Location | At rest |
|---|---|---|
| Source registry (incl. **DB connection + password**) | SQLite `TDB_REGISTRY_DB` (`/app/data/tdb_registry.db`) | **Encrypted** (Fernet, `enc:v1:` prefix) — whole connection blob, no plaintext mode. Key from `TDB_ENCRYPTION_KEY` or an auto-generated `0600` file beside the DB — see [§6.2](#62-where-source-credentials-live) |
| API keys | same SQLite DB (`api_keys` table) | **SHA-256 hashed** (only hash + 6-char prefix); raw key shown once at creation |
| OAuth clients / authorization codes | same SQLite DB | codes are short-lived |
| Audit log (query text, source id, row count, key hint, timestamp) | `TDB_LOG_FILE` NDJSON | plaintext, hash-chained |
| License token | `/app/license.jwt` (or `TDB_LICENSE` env) | signed JWT; baked tokens are recoverable via `docker history` |

### 6.2 Where source credentials live

When you register a database source, the **entire connection block — including the
password — is encrypted before it is written** to the `sources.connection` column of
the SQLite registry DB (Fernet: AES-128-CBC + HMAC-SHA256, values prefixed `enc:v1:`).
The whole blob is encrypted, not just password-like fields, so host and username are
covered too. **There is no plaintext mode** — you cannot turn this off, and a database
from an older version is re-encrypted in place at first startup.

The encryption key comes from one of two places:

| Key source | How | What it protects against |
|---|---|---|
| `TDB_ENCRYPTION_KEY` (**recommended**) | You supply it from your secrets manager | Theft of the database file *and* theft of the whole data volume — the key never lands on disk beside the data |
| Auto-generated (default) | Created as `.tdb_secret_key` (mode `0600`) beside the registry DB on first start | Theft of the **database file alone** — e.g. a stray backup or a copied `.db`. **Not** theft of the whole data directory, since the key travels with it |

Be precise about that distinction in any security questionnaire: with the default
key, "credentials are encrypted at rest" is true and the key sits next to the data.
Setting `TDB_ENCRYPTION_KEY` is what makes the claim unconditional.

Still applies regardless:

- **Protect the `/app/data` volume**: restrictive filesystem permissions, and put it on
  an **encrypted volume / encrypted-at-rest storage class**.
- **Use a dedicated least-privilege, read-only DB account per source** so the blast
  radius of a leaked credential is "read this one table," not "own the database."
- Restrict who holds an **admin** API key — registering/reading sources is an admin
  operation.
- **Back the key up separately from the database.** Losing it means re-registering
  every source; there is no offline recovery and no online re-key command today.

Registration responses and `GET /v1/sources` mask secret fields (`password`,
`token`, …) as `***`, so credentials are not echoed back over the API either.

> Full detail: [Credentials at Rest](../security/credentials.md) and the
> `TDB_ENCRYPTION_KEY` entry in the
> [environment variable reference](../reference/environment-variables.md).

### 6.3 Where runtime secrets live

`TDB_JWT_SECRET`, `TDB_ADMIN_PASSWORD`, `TDB_SPLUNK_HEC_TOKEN`, and a runtime
`TDB_LICENSE` are read from **environment variables**. They are **not** written to the
registry DB, but standard env-secret caveats apply: they are visible via
`docker inspect`, the container's `/proc/1/environ`, and orchestrator manifests. Inject
them from your secret manager (Docker/K8s secrets, Vault) rather than committing them.
A **baked** license token is additionally recoverable from image metadata
(`docker history`) — acceptable because it only verifies against our public key and
grants nothing beyond what was issued.

### 6.4 DuckDB / query engine access

- CSV sources are queried **in-process by DuckDB**; there is no DuckDB server and no
  persisted DuckDB database file. DuckDB reads the CSV file directly per query.
- CSV file access is **confined to `TDB_ALLOWED_DATA_DIR`** (default `/data`); symlinks
  and `..` are resolved and paths outside are rejected. Keep that mount read-only.
- Queries are **`SELECT`-only**, enforced by the SQL validator before execution; the
  CSV path is passed through DuckDB's `read_csv` Python API (never string-interpolated),
  which prevents path/SQL injection.

### 6.5 Authentication & directory integration

TDB authenticates **callers** via static API keys, JWT (HMAC, `TDB_JWT_SECRET`), and
**OAuth 2.1 + PKCE** (for Claude/ChatGPT/Cursor MCP), plus a single
`TDB_ADMIN_USER`/`TDB_ADMIN_PASSWORD` for the consent/login flow. Authorization is
**RBAC** (`read` / `readwrite` / `admin`) per API key, with optional MCP tool-level
scoping.

> **Directory / AD integration is not supported today:**
> - **No SSO / SAML / OIDC-IdP / LDAP / Active Directory login** to TDB itself (this is
>   on the post-launch roadmap, not in the current edition).
> - **No Kerberos / keytab / Windows-integrated auth to data sources.** All DB
>   connectors use **SQL username/password** auth only (SQL Server uses `UID/PWD`, not
>   `Trusted_Connection`). If your security policy mandates AD/Kerberos service accounts
>   for database access, that requirement is **not met today** — the supported pattern is
>   a dedicated, least-privilege SQL login per source. If AD-backed database auth is a
>   hard requirement in your environment, raise it with us before you evaluate.

---

## 7. TLS / SSL

**TDB serves plain HTTP on port 8000 and does not terminate TLS itself.** For any
non-localhost deployment, put it behind a TLS-terminating **reverse proxy** (nginx,
Caddy, Traefik) or an ingress controller / cloud load balancer.

Minimal nginx example:

```nginx
server {
    listen 443 ssl;
    server_name tdb.yourcompany.com;
    ssl_certificate     /etc/ssl/certs/tdb.crt;
    ssl_certificate_key /etc/ssl/private/tdb.key;

    location / {
        proxy_pass         http://127.0.0.1:8000;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

Then set **`TDB_SERVER_URL=https://tdb.yourcompany.com`** so TDB advertises the correct
external HTTPS URL as its OAuth issuer. Note the **OAuth flow requires `https://`**
redirect URIs (only `http://localhost` is exempted) — OAuth will not work over plain
HTTP to a non-local host, which is by design. Restrict the container's port 8000 to the
proxy (bind to `127.0.0.1` or a private network), not the public interface.

---

## 8. Performance & capacity

> **No published benchmark figures.** We deliberately do not quote throughput or latency
> numbers, because we have not run a benchmark we would be willing to hold ourselves to.
> What follows is a repeatable method for measuring TDB on your own hardware and data,
> plus the qualitative behaviour that drives the result.

How to benchmark in your environment:
1. Register a representative source (your real CSV size / DB table).
2. Drive load with a tool like `k6`/`hey`/`wrk` against `POST /v1/query` with a
   representative `SELECT`, ramping concurrency.
3. Watch container CPU/RAM (`docker stats`) and the `/metrics` endpoint
   (`request_count`, `request_latency`, `schema_cache_hits`).

What to expect qualitatively:
- **CSV/DuckDB** is CPU/RAM-bound and re-reads the file per query — large CSVs benefit
  most from more RAM and from the **schema cache** (`TDB_SCHEMA_CACHE_TTL`).
- **DB connectors** push work to the remote engine; TDB mostly marshals rows, so the TDB
  host stays light and the source DB is the bottleneck.
- **Every response is capped at 1,000 rows, in both editions** — `limit` defaults to 100
  and cannot exceed 1,000, and there is **no pagination or cursor today**, so a result set
  larger than 1,000 rows must be narrowed in SQL (filter, aggregate, or `LIMIT`/`OFFSET`
  in the query itself against a database source). This bounds per-request memory, and it
  is the main thing to check before planning a bulk-extract workload.
- Single process/single writer — see the scaling caveat in [§4.3](#43-kubernetes-reference-only).

---

## 9. Post-setup verification

Run these after deploying.

1. **Liveness:** `curl -fsS http://localhost:8000/health` → `{"status":"ok"}`. The
   version banner is `GET /`.
2. **License accepted:** `docker logs tdb | grep license_ok` shows
   `customer=... edition=... days_left=N`. (If you see 403s on data routes with
   `reason: missing|expired`, the license isn't being picked up.)
3. **Auth enforced:** a query **without** a bearer key → `401`; with a valid key → `200`.
4. **Register + query a source** ([§5.1](#51-registering-a-data-source)) → rows returned.
5. **Read-only enforced:** send a non-SELECT (e.g. `UPDATE ...`) → rejected (`400`), and
   nothing is written to the source.
6. **Audit working:** confirm a new line appended to `TDB_LOG_FILE` after a successful
   query, then `GET /v1/audit/verify` → `{"valid": true, ...}`.
7. **Metrics:** `curl -fsS http://localhost:8000/metrics` returns Prometheus text;
   wire it into your scraper.
8. **MCP (if used):** point your AI client at `https://<host>/v1/mcp` and confirm the
   `initialize` handshake + a `query_source` call succeed.

---

## 10. Day-2 operations

- **Backups:** snapshot the `/app/data` volume (registry DB + API keys) and archive the
  audit log. These two volumes are the entire stateful surface.
- **Upgrades:** pull the new image tag, stop the old container, start the new one against
  the **same** `/app/data` volume. Migrations are idempotent and run on startup.
- **Key rotation:** add/rotate API keys via the `/v1/apikeys` admin API (raw key shown
  once); rotate `TDB_JWT_SECRET` (invalidates outstanding JWTs) on a schedule.
- **License renewal:** see [§5.5](#55-license).
- **Monitoring:** scrape `/metrics`; alert on `/health` and on `audit/verify` returning
  `valid:false`.

---

## 11. Related documentation

- [Installation](../getting-started/installation.md) — image channels, license, first run
- [Quickstart](../getting-started/quickstart.md) — register a source and run a query
- [Connect an IDE / AI Tool (MCP)](../getting-started/ide-setup.md)
- [Environment variables](../reference/environment-variables.md) — every setting in §3
- [RBAC](../security/rbac.md) · [Audit Log](../security/audit.md) ·
  [Credentials at Rest](../security/credentials.md) · [Security & Updates](../security/updates.md)
- [Prometheus metrics](../observability/metrics.md) · [Splunk HEC](../integrations/splunk.md)
- [Editions](../pricing.md) — what the community and paid editions each include

Questions about a deployment we have not covered here?
[hello@tdb.jiracorp.co.in](mailto:hello@tdb.jiracorp.co.in)
