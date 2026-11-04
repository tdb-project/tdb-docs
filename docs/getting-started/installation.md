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
    docker pull registry.tdb.jiracorp.co.in/tdb-customers/acme/tdb-enterprise:trial-20260701-9a54018
    ```

    Your welcome e-mail contains your exact repository path and tag.

=== "Tarball (air-gapped)"

    If your environment cannot reach our registry, we provide the same image as a
    compressed tarball instead. Load it:

    ```bash
    docker load < trial-acme-20260701-9a54018.tar.gz
    # Loaded image: tdb-enterprise:trial-acme-20260701-9a54018
    ```

---

## Run the server

=== "Terminal (recommended)"

    ```bash
    mkdir -p data   # BEFORE docker run — see the warning below
    export TDB_API_KEYS=$(python3 -c "import secrets; print(secrets.token_hex(32))")
    echo "$TDB_API_KEYS"   # keep this: it is your admin key
    docker run -d --name tdb \
      -p 8000:8000 \
      -e TDB_API_KEYS \
      -e TDB_JWT_SECRET=$(python3 -c "import secrets; print(secrets.token_hex(32))") \
      -e TDB_ADMIN_USER=admin \
      -e TDB_ADMIN_PASSWORD=your-strong-password \
      -v "$(pwd)/data:/app/data" \
      tdb-enterprise:trial-acme-20260701-9a54018
    ```

!!! warning "Linux hosts: create the `data` directory before `docker run`"

    If `./data` does not exist, Docker creates it **owned by root** — and TDB
    runs as a non-root user (uid 1000) which then cannot open its registry
    database inside it. On **0.7.0 and earlier** the container exits at startup
    with SQLite's unhelpful `unable to open database file`; from **0.7.1** the
    error names the path and the fix. Either `mkdir -p data` first (as above) or
    `chown 1000` an existing directory. Docker Desktop on macOS/Windows is not
    affected, which is why this bites on exactly the Linux servers most
    production deployments use.

!!! note "Using CSV sources? Mount your files at `/data`"

    The image confines registered CSV paths to `/data`
    ([`TDB_ALLOWED_DATA_DIR`](../reference/environment-variables.md)) — a
    `file_path` anywhere else is refused with 403. Add a second mount for your
    CSV files:

    ```bash
      -v "$(pwd)/csvs:/data" \
    ```

    Database connectors are unaffected — they take a table name, not a path.

    Your Bearer token for all API requests is the value you set for `TDB_API_KEYS`.
    Use it as:

    ```
    Authorization: Bearer <value-of-TDB_API_KEYS>
    ```

=== "Docker Desktop"

    Set `TDB_API_KEYS` under **Optional settings → Environment variables** when you start the
    container: TDB will not start without it. Generate a value with
    `python3 -c "import secrets; print(secrets.token_hex(32))"`, then use it as:

    ```
    Authorization: Bearer <value-of-TDB_API_KEYS>
    ```

Check the logs — you should see the license confirmed and the server start:

```
INFO:     license_ok customer=ACME Corp edition=trial tier=midmarket expires=2026-07-01T00:00:00+00:00 days_left=30
INFO:     tdb_startup version=0.30.0 build_sha=b0671ce audit_sql=full
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

Verify TDB is running:

```bash
curl http://localhost:8000/health
# {"status":"ok"}
```

!!! warning "TDB needs a real key to start"
    From community 0.14.0 and enterprise 0.25.0 there is no built-in key. TDB will not start
    with an example value, or, unless another login is set up, without `TDB_API_KEYS`.

---

## Configuration

TDB is configured entirely through environment variables, passed with `-e` (or an
`--env-file`). The essentials:

```bash title="tdb.env  (use with: docker run --env-file tdb.env ...)"
# --- Required ---
TDB_API_KEYS=<output of: python3 -c "import secrets; print(secrets.token_hex(32))">  # Comma-separated

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
