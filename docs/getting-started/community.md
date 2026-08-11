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

    docker run -d --rm --name tdb -p 8000:8000 \
      -e TDB_API_KEYS \
      -v "$PWD/data:/data:ro" \
      ghcr.io/tdb-project/tdb-community:latest
    ```

=== "Windows (PowerShell)"

    ```powershell
    $env:TDB_API_KEYS = (python3 -c "import secrets; print(secrets.token_hex(32))")

    docker run -d --rm --name tdb -p 8000:8000 `
      -e TDB_API_KEYS `
      -v "${PWD}\data:/data:ro" `
      ghcr.io/tdb-project/tdb-community:latest
    ```

The `-d` flag runs TDB **detached** (in the background), so this same terminal stays free for
Steps 2–5 — and the `TDB_API_KEYS` you just set stays available to them. Naming the container
`tdb` lets the later commands refer to it by name. Put your CSV in a `data/` folder next to
where you run the command.

!!! tip "Image tags"
    `:latest` always points to the newest **stable release** (not the tip of `main`). For
    production, pin an immutable release tag like `:0.4.2` — or a `@sha256:` digest;
    `:0.4` floats to the newest patch within a minor. Pull `:edge` to try the latest
    unreleased `main` build.

Verify it's up (use `curl.exe` on Windows — see the note below):

```bash
curl http://localhost:8000/health
# {"status": "ok"}
```

Open [http://localhost:8000/docs](http://localhost:8000/docs) for the interactive Swagger UI.
Tail the container logs any time with `docker logs -f tdb`.

!!! warning "Windows: `curl.exe` for the GET, `Invoke-RestMethod` for JSON POSTs"
    In PowerShell, `curl` is an **alias for `Invoke-WebRequest`**, so type `curl.exe` explicitly
    for the Step 1 health check to get the real curl. For the POST requests that send a JSON body
    (Steps 2–3), PowerShell mangles quoted JSON when handing it to `curl.exe` — the spaces in your
    SQL get split into separate arguments — so the Windows tabs use the native `Invoke-RestMethod`
    cmdlet instead. It's the reliable approach across PowerShell 5.1 and 7.x.

!!! note "Stopping TDB"
    When you're done, stop the container from the same terminal:

    ```bash
    docker stop tdb
    ```

    Because it was started with `--rm`, stopping also **removes** it — the ephemeral registry
    and audit log are discarded. For a setup that survives restarts, see the tip at the end.

---

## Step 2 — Register your CSV

=== "macOS / Linux"

    ```bash
    curl -X POST http://localhost:8000/v1/sources \
      -H "Authorization: Bearer $TDB_API_KEYS" \
      -H "Content-Type: application/json" \
      -d '{"name":"mydata","source_type":"csv","connection":{"file_path":"/data/your_file.csv"}}'
    ```

=== "Windows (PowerShell)"

    ```powershell
    $body = @{
      name        = "mydata"
      source_type = "csv"
      connection  = @{ file_path = "/data/your_file.csv" }
    } | ConvertTo-Json

    Invoke-RestMethod -Method Post -Uri http://localhost:8000/v1/sources `
      -Headers @{ Authorization = "Bearer $env:TDB_API_KEYS" } `
      -ContentType "application/json" -Body $body
    ```

Fill in the two placeholders: **`name`** (`mydata`) is any label you choose for the source,
and **`file_path`** (`/data/your_file.csv`) is the path to *your* CSV **inside the container** —
replace `your_file.csv` with your file's name. It must sit in the `data/` folder you mounted in
Step 1, i.e. `/data/<your-file>.csv`. Schema (column names + types) is auto-detected from the
CSV header row, and the response returns the new source's `id` — copy it for Step 3.

!!! note "The table is always named `data`"
    Whatever your file or source is called, TDB exposes its rows as a single table named
    `data`. That fixed name — not your filename — is what goes in the `FROM` clause in Step 3.

---

## Step 3 — Query it (REST)

=== "macOS / Linux"

    ```bash
    curl -X POST http://localhost:8000/v1/query \
      -H "Authorization: Bearer $TDB_API_KEYS" \
      -H "Content-Type: application/json" \
      -d '{"source_id":"<id-from-step-2>","sql":"SELECT * FROM data","limit":10}'
    ```

=== "Windows (PowerShell)"

    ```powershell
    $body = @{
      source_id = "<id-from-step-2>"
      sql       = "SELECT * FROM data"
      limit     = 10
    } | ConvertTo-Json

    Invoke-RestMethod -Method Post -Uri http://localhost:8000/v1/query `
      -Headers @{ Authorization = "Bearer $env:TDB_API_KEYS" } `
      -ContentType "application/json" -Body $body
    ```

`SELECT * FROM data` works on any CSV — swap it for any read-only query you like. The
**column names are exactly your CSV's header row** (run `SELECT * FROM data` once to see them).
The optional `limit` field caps the response: it defaults to **100** and is hard-capped at
**1,000**, so a bare `SELECT *` returns at most those rows regardless of your SQL. Asking for
more than 1,000 is **rejected with `422`**, not silently clamped — so you never receive a
short answer believing it is complete. (Enterprise, where the ceiling is configurable via
`TDB_MAX_ROWS`, rejects an over-ceiling `limit` with `400` and an
`limit_exceeds_max` reason.) When rows *were* dropped, the response carries
`"truncated": true` — reliable from **0.4.7**; earlier versions under-reported it on
queries carrying no `LIMIT` of their own. Read-only is enforced: `INSERT` / `UPDATE` /
`DELETE` / `DROP` are rejected at the API level.

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

Restart Claude Desktop; `query_source` appears in the tools list.

!!! warning "Config format differs by client"
    Don't reuse this exact block for other clients — VS Code's `mcp.json` uses a
    top-level `servers` key (not `mcpServers`), and Windsurf uses `serverUrl`
    (not `url`). See [Connect an IDE / AI Tool →](ide-setup.md) for **Cursor**,
    **VS Code**, **JetBrains**, **Windsurf**, and **Cline**, each with the correct
    JSON and example queries to run once connected.

---

## Step 5 — Check the audit log

Every query — REST or from an AI tool — is appended to a local NDJSON file
(`tdb_audit.jsonl`) inside the container:

=== "macOS / Linux"

    ```bash
    docker exec tdb cat /app/tdb_audit.jsonl | tail -3
    ```

=== "Windows (PowerShell)"

    ```powershell
    docker exec tdb cat /app/tdb_audit.jsonl | Select-Object -Last 3
    ```

Each line records the SQL, the row count, a **truncated** key hint (never the raw key),
and a UTC timestamp:

```json
{"event":"query","source_id":"…","sql":"SELECT …","rows_returned":3,"key_hint":"a1b2c3…","ts":"2026-06-01T03:02:01.160+00:00"}
```

Refused attempts — a bad API key, a non-`SELECT` statement, an unknown source —
are logged too, as `"event":"denied"` with an `action` and a `reason`:

```json
{"event":"denied","action":"query","reason":"sql_validation_failed","source_id":"…","sql":"DROP TABLE data","key_hint":"a1b2c3…","ts":"2026-06-01T03:02:04.771+00:00"}
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
