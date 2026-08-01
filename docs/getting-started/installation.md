# Installation

!!! info "TDB Enterprise"
    This guide covers the **commercial enterprise edition** (licensed Docker image,
    SQL database connectors). For the free, open-source CSV edition, see the
    [Community Edition quickstart](community.md).

TDB Enterprise ships as a **Docker image**. You do not build it from source —
you receive a ready-to-run image as part of your license (a trial image, or a
pull from our private registry for commercial customers).

## Requirements

| Requirement | Version |
|---|---|
| Docker | 20.10 or later (or any OCI runtime) |
| CPU architecture | **Trial / evaluation images are multi-arch** — `linux/amd64` and `linux/arm64`, so they run natively on Apple Silicon Macs as well as x86 servers. Commercial production images are `linux/amd64`; [tell us](mailto:hello@tdb.jiracorp.co.in) if you need arm64 in production (e.g. AWS Graviton) |
| PostgreSQL / MySQL / SQL Server / Snowflake | Your existing database — TDB connects to it |

That's it. Python, `uv`, and the source tree are **not** required to run TDB —
everything is inside the image.

---

## Get the image

Your image — trial or commercial — is a dedicated build with your license already
baked in. The license travels inside the image; there is nothing else to configure
for it.

=== "Private registry (recommended)"

    We deliver via a **customer-scoped private registry**: you receive a pull token
    that is scoped to your own repository path and can see only your images. Log in
    with that token and pull:

    ```bash
    docker login registry.tdb.jiracorp.co.in        # use your issued pull token
    docker pull registry.tdb.jiracorp.co.in/tdb-customers/acme/tdb-enterprise:trial-20260701
    ```

    Your welcome e-mail contains your exact repository path and tag.

=== "Tarball (air-gapped)"

    If your environment cannot reach our registry, we provide the same image as a
    compressed tarball instead. Load it:

    ```bash
    docker load < trial-acme-20260701.tar.gz
    # Loaded image: tdb-enterprise:trial-acme-20260701
    ```

---

## Run the server

=== "Terminal (recommended)"

    ```bash
    docker run -d --name tdb \
      -p 8000:8000 \
      -e TDB_API_KEYS=your-secret-key-here \
      -e TDB_JWT_SECRET=$(python3 -c "import secrets; print(secrets.token_hex(32))") \
      -e TDB_ADMIN_USER=admin \
      -e TDB_ADMIN_PASSWORD=your-strong-password \
      -v "$(pwd)/data:/app/data" \
      tdb-enterprise:trial-acme-20260701
    ```

    Your Bearer token for all API requests is the value you set for `TDB_API_KEYS`
    (here: `your-secret-key-here`). Use it as:

    ```
    Authorization: Bearer your-secret-key-here
    ```

=== "Docker Desktop"

    If you started the container from Docker Desktop without setting environment
    variables, TDB falls back to a built-in development key:

    ```
    Authorization: Bearer dev-insecure-key-change-me
    ```

    Use this key to complete the Quickstart. Before exposing TDB to any network,
    stop the container and re-run it with `-e TDB_API_KEYS=<your-own-key>` (or set
    it in Docker Desktop's **Optional settings → Environment variables**).

Check the logs — you should see the license confirmed and the server start:

```
INFO:     license_ok customer=ACME Corp edition=trial expires=2026-07-01T00:00:00+00:00 days_left=30
INFO:     tdb_startup version=0.2.0 build_sha=fd18b38 dev_mode=False
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

Verify TDB is running:

```bash
curl http://localhost:8000/health
# {"status":"ok"}
```

!!! warning "Never use the default dev key in production"
    Always set a strong `TDB_API_KEYS` value before exposing TDB to any network.
    The fallback key (`dev-insecure-key-change-me`) is for local evaluation only.

---

## Configuration

TDB is configured entirely through environment variables, passed with `-e` (or an
`--env-file`). The essentials:

```bash title="tdb.env  (use with: docker run --env-file tdb.env ...)"
# --- Required ---
TDB_API_KEYS=your-secret-key-here         # Comma-separated; used for initial setup

# --- Required for JWT + OAuth ---
TDB_JWT_SECRET=<64-char hex>              # python -c "import secrets; print(secrets.token_hex(32))"
TDB_ADMIN_USER=admin
TDB_ADMIN_PASSWORD=<strong-password>

# --- Optional ---
TDB_LOG_LEVEL=INFO                        # DEBUG | INFO | WARNING | ERROR
TDB_DEFAULT_RATE_LIMIT=60                 # Default requests per minute per API key
```

See the [Environment Variables reference](../reference/environment-variables.md) for
the full list, including the **License** variables.

---

## Licensing & expiry

TDB Enterprise requires a valid, signed license to serve data.

- **Trial images** carry the license inside the image (at `/app/license.jwt`) — no
  setup needed.
- **Commercial deployments** can instead supply the token at runtime with
  `-e TDB_LICENSE=<token>`, which takes precedence over any baked file. This lets
  you drop in a renewed license without pulling a new image.

**What happens at expiry:** TDB keeps running and `/health` stays green (so your
orchestrator does not kill the container), but every data and API request returns:

```json
HTTP 403
{ "error": "license_expired", "reason": "expired", "expired_at": "..." }
```

To extend a trial or renew, [contact us](mailto:hello@tdb.jiracorp.co.in) for a new
image or token.

---

## Running behind a reverse proxy

If you put TDB behind nginx, Caddy, or a load balancer, set `TDB_SERVER_URL` to the
public base URL. This is required for OAuth 2.1 discovery endpoints to return correct URLs:

```bash
-e TDB_SERVER_URL=https://tdb.yourcompany.com
```

---

## Data directories

TDB writes two files (created automatically). Mount a volume so they persist across
container restarts:

| Path (in container) | Contents |
|---|---|
| `/app/data/tdb_registry.db` | SQLite registry of registered sources |
| `/app/tdb_audit.jsonl` | NDJSON audit log — one entry per query |

The `-v "$(pwd)/data:/app/data"` flag in the run command above persists the registry.
Both paths are configurable (`TDB_REGISTRY_DB`, `TDB_LOG_FILE`).

---

## Next step

[Quickstart — register your first source and run a query →](quickstart.md)
