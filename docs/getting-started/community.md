# Community Edition — 5-minute quickstart

The free, open-source ([AGPLv3](https://github.com/tdb-project/tdb-community/blob/main/LICENSE))
edition turns a **CSV file** into a governed REST + MCP API, with an audit log on every
query. No database, no build step — just Docker.

!!! info "Community vs Enterprise"
    Community is CSV-only, one source at a time, single static API key, 1,000-row cap,
    one MCP tool (`query_source`), and a local NDJSON audit log. Need SQL databases,
    multiple sources, OAuth, RBAC, or tamper-evident audit export? See [Editions](../pricing.md).

---

## Step 1 — Run it (no clone needed)

The image is published to GHCR and is multi-arch (Intel + Apple Silicon). You only need
[Docker Desktop](https://www.docker.com/products/docker-desktop/).

=== "macOS / Linux"

    ```bash
    export TDB_API_KEYS=$(python3 -c "import secrets; print(secrets.token_hex(32))")

    docker run --rm -p 8000:8000 \
      -e TDB_API_KEYS \
      -v "$PWD/data:/data:ro" \
      ghcr.io/tdb-project/tdb-community:latest
    ```

=== "Windows (PowerShell)"

    ```powershell
    $env:TDB_API_KEYS = (python3 -c "import secrets; print(secrets.token_hex(32))")

    docker run --rm -p 8000:8000 `
      -e TDB_API_KEYS `
      -v "${PWD}\data:/data:ro" `
      ghcr.io/tdb-project/tdb-community:latest
    ```

Put your CSV in a `data/` folder next to where you run the command. To pin a version, use
`:0.4.1` instead of `:latest`.

Verify it's up:

```bash
curl http://localhost:8000/health
# {"status": "ok"}
```

Open [http://localhost:8000/docs](http://localhost:8000/docs) for the interactive Swagger UI.

---

## Step 2 — Register your CSV

```bash
curl -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer $TDB_API_KEYS" \
  -H "Content-Type: application/json" \
  -d '{"name":"sales","source_type":"csv","connection":{"file_path":"/data/sales.csv"}}'
```

Schema (column names + types) is auto-detected. The table is always queryable as `data`.

---

## Step 3 — Query it (REST)

```bash
curl -X POST http://localhost:8000/v1/query \
  -H "Authorization: Bearer $TDB_API_KEYS" \
  -H "Content-Type: application/json" \
  -d '{"source_id":"<id-from-step-2>","sql":"SELECT country, SUM(units) AS total FROM data GROUP BY country ORDER BY total DESC","limit":100}'
```

Read-only is enforced: `INSERT` / `UPDATE` / `DELETE` / `DROP` are rejected at the API
level. Responses are capped at **1,000 rows** — even if your SQL has a larger `LIMIT`.

---

## Step 4 — Connect an AI tool (MCP)

TDB exposes a standard [MCP](https://modelcontextprotocol.io) endpoint at `/v1/mcp`
(Streamable HTTP). The Community Edition ships one tool: `query_source`.

**Claude Desktop** — edit `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "tdb": {
      "type": "http",
      "url": "http://localhost:8000/v1/mcp",
      "headers": { "Authorization": "Bearer YOUR_API_KEY" }
    }
  }
}
```

Restart Claude Desktop; `query_source` appears in the tools list. The same `type: "http"`
block works for **Cursor** (`.cursor/mcp.json`) and **VS Code** (`mcp.json`).

---

## Step 5 — Check the audit log

Every query — REST or from an AI tool — is appended to a local NDJSON file
(`tdb_audit.jsonl`) inside the container:

```bash
docker exec <container> cat /app/tdb_audit.jsonl | tail -3
```

Each line records the SQL, the row count, a **truncated** key hint (never the raw key),
and a UTC timestamp:

```json
{"event":"query","source_id":"…","sql":"SELECT …","rows_returned":3,"key_hint":"a1b2c3…","ts":"2026-06-01T03:02:01.160+00:00"}
```

!!! tip "Persisting data and logs"
    The `--rm` run above is ephemeral. For a persistent setup, use the
    [`docker-compose.yml`](https://github.com/tdb-project/tdb-community/blob/main/docker-compose.yml)
    from the repo — it mounts named volumes for the registry and audit log.

---

## Next steps

- Full README & source: [github.com/tdb-project/tdb-community](https://github.com/tdb-project/tdb-community)
- Outgrown one CSV? [Compare editions](../pricing.md) — Enterprise adds Postgres/MySQL/SQL
  Server/Snowflake, multiple sources, OAuth, RBAC, and tamper-evident audit export.
