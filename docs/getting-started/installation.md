# Installation

## Requirements

| Requirement | Version |
|---|---|
| Python | 3.12 or later |
| [uv](https://docs.astral.sh/uv/) | latest |
| PostgreSQL | 12 or later (your existing database — TDB connects to it) |

TDB Enterprise does not require Docker to run. `uv` manages the Python environment and dependencies.

---

## Install uv

=== "macOS / Linux"

    ```bash
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```

=== "Windows"

    ```powershell
    powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
    ```

---

## Install TDB Enterprise

You will receive a copy of the `tdb-enterprise` source repository as part of your license. Clone or unzip it to a local directory, then install:

```bash
cd tdb-enterprise
uv sync
```

This installs all dependencies into `.venv` inside the project directory. The `uv sync` command is idempotent — safe to run on updates.

Verify the install:

```bash
uv run python -c "import tdb_enterprise; print('ok')"
```

---

## Configure environment variables

TDB is configured entirely through environment variables. Create a `.env` file in the `tdb-enterprise` directory:

```bash title=".env"
# --- Required ---
TDB_API_KEYS=your-secret-key-here         # Comma-separated; used for the initial setup

# --- Required for JWT + OAuth ---
TDB_JWT_SECRET=<64-char hex>              # Generate with: python -c "import secrets; print(secrets.token_hex(32))"
TDB_ADMIN_USER=admin
TDB_ADMIN_PASSWORD=<strong-password>

# --- Optional ---
TDB_LOG_LEVEL=INFO                        # DEBUG | INFO | WARNING | ERROR
TDB_LOG_FILE=tdb_audit.jsonl              # Audit log output path
TDB_REGISTRY_DB=data/tdb_registry.db     # Source registry SQLite path
TDB_DEFAULT_RATE_LIMIT=60                 # Default requests per minute per API key
```

!!! warning "Never use the default dev key in production"
    The default `TDB_API_KEYS` value is `dev-insecure-key-change-me`. TDB prints a
    warning on startup if it detects this key. Always set a strong key before exposing
    TDB to any network.

Generate a secure JWT secret:

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

---

## Start the server

```bash
uv run uvicorn tdb.main:app --host 0.0.0.0 --port 8000
```

You should see output similar to:

```
INFO:     Started server process [12345]
INFO:     Waiting for application startup.
INFO:     tdb_startup version=0.4.0 dev_mode=False
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

Verify TDB is running:

```bash
curl http://localhost:8000/health
# {"status":"ok"}
```

---

## Running behind a reverse proxy

If you put TDB behind nginx, Caddy, or a load balancer, set `TDB_SERVER_URL` to the
public base URL. This is required for OAuth 2.1 discovery endpoints to return correct URLs:

```bash
TDB_SERVER_URL=https://tdb.yourcompany.com
```

---

## Data directories

TDB writes two files on first startup (created automatically):

| Path | Contents |
|---|---|
| `data/tdb_registry.db` | SQLite registry of registered sources |
| `tdb_audit.jsonl` | NDJSON audit log — one entry per query |

Both paths are configurable via environment variables. Make sure the process has write
access to these locations.

---

## Next step

[Quickstart — register your first source and run a query →](quickstart.md)
