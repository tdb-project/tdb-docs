# Sources API

The Sources API manages the registry of data sources. All endpoints require authentication
(`Authorization: Bearer <token>`).

Base path: `/v1/sources`

!!! tip "Use names, not UUIDs"
    Every path parameter that accepts `<source_id>` also accepts the source's
    **registered name** (e.g. `customers`). Names are case-insensitive. UUIDs still
    work — they are useful in scripts or audit-trail lookups — but the name is easier
    to type for day-to-day operations.

---

## Register a source

```
POST /v1/sources
```

TDB supports two registration modes:

- **Database-wide** (recommended): omit `table` to register an entire database. One
  registration covers all tables. Users query any table by name in SQL.
- **Single-table**: include `table` to scope the source to a specific table. The
  schema endpoint returns only that table's columns.

=== "Database-wide (Bash)"

    ```bash
    curl -X POST http://localhost:8000/v1/sources \
      -H "Authorization: Bearer <YOUR_KEY>" \
      -H "Content-Type: application/json" \
      -d '{
        "name": "production_db",
        "source_type": "postgres",
        "connection": {
          "host": "host.docker.internal",
          "port": 5432,
          "dbname": "production",
          "user": "tdb_reader",
          "password": "s3cret",
          "schema": "public"
        },
        "description": "Production Postgres — all tables",
        "tags": ["production"]
      }'
    ```

=== "Single-table (Bash)"

    ```bash
    curl -X POST http://localhost:8000/v1/sources \
      -H "Authorization: Bearer <YOUR_KEY>" \
      -H "Content-Type: application/json" \
      -d '{
        "name": "orders",
        "source_type": "postgres",
        "connection": {
          "host": "host.docker.internal",
          "port": 5432,
          "dbname": "production",
          "user": "tdb_reader",
          "password": "s3cret",
          "table": "orders",
          "schema": "public"
        },
        "description": "Order records only",
        "tags": ["production", "finance"]
      }'
    ```

=== "Database-wide (PowerShell)"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/sources" `
      -Method POST `
      -ContentType "application/json" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" } `
      -Body '{
        "name": "production_db",
        "source_type": "postgres",
        "connection": {
          "host": "host.docker.internal",
          "port": 5432,
          "dbname": "production",
          "user": "tdb_reader",
          "password": "s3cret",
          "schema": "public"
        },
        "description": "Production Postgres — all tables",
        "tags": ["production"]
      }'
    ```

=== "Single-table (PowerShell)"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/sources" `
      -Method POST `
      -ContentType "application/json" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" } `
      -Body '{
        "name": "orders",
        "source_type": "postgres",
        "connection": {
          "host": "host.docker.internal",
          "port": 5432,
          "dbname": "production",
          "user": "tdb_reader",
          "password": "s3cret",
          "table": "orders",
          "schema": "public"
        },
        "description": "Order records only",
        "tags": ["production", "finance"]
      }'
    ```

**Request body fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string (1–100) | Yes | Unique human-readable identifier — used in all subsequent commands |
| `source_type` | string | Yes | Connector type: `"postgres"`, `"mysql"`, `"sqlserver"`, `"snowflake"`, `"csv"` |
| `connection` | object | Yes | Connector-specific config — see [PostgreSQL →](../connectors/postgresql.md) |
| `description` | string (max 500) | No | Human-readable description |
| `tags` | array of strings | No | Arbitrary labels for organisation |

**Responses:**

| Status | Meaning |
|---|---|
| 201 | Source registered. Returns full `SourceRecord`. |
| 400 | Invalid `source_type` or connection details. |
| 409 | A source with this name already exists. |

**Response body (201):**

```json
{
  "id": "a1b2c3d4-e5f6-...",
  "name": "production_db",
  "source_type": "postgres",
  "connection": { "host": "host.docker.internal", "port": 5432, "dbname": "production", "schema": "public" },
  "description": "Production Postgres — all tables",
  "tags": ["production"],
  "registered_by": "tdbk_...",
  "registered_at": "2026-05-22T09:00:00Z",
  "status": "active"
}
```

---

## List sources

**Why you'd use this:** Verify what sources are registered and retrieve their names
before running queries. Always run this after registering a new source to confirm it
was accepted.

```
GET /v1/sources
```

=== "Bash"

    ```bash
    curl http://localhost:8000/v1/sources \
      -H "Authorization: Bearer <YOUR_KEY>"
    ```

=== "PowerShell"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/sources" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" }
    ```

Returns a summary list (no `connection` details, no passwords).

**Response:**

```json
[
  {
    "id": "a1b2c3d4-...",
    "name": "orders",
    "source_type": "postgres",
    "description": "Order records",
    "tags": ["production", "finance"],
    "registered_at": "2026-05-22T09:00:00Z"
  },
  {
    "id": "b2c3d4e5-...",
    "name": "products",
    "source_type": "postgres",
    "description": "",
    "tags": [],
    "registered_at": "2026-05-22T09:05:00Z"
  }
]
```

---

## Get a source

**Why you'd use this:** Retrieve full connection details for a source (e.g. to verify
which database and table it points to). Accepts the source **name** or UUID.

```
GET /v1/sources/<source_id_or_name>
```

=== "Bash (by name)"

    ```bash
    curl http://localhost:8000/v1/sources/orders \
      -H "Authorization: Bearer <YOUR_KEY>"
    ```

=== "Bash (by UUID)"

    ```bash
    curl http://localhost:8000/v1/sources/a1b2c3d4-e5f6-... \
      -H "Authorization: Bearer <YOUR_KEY>"
    ```

=== "PowerShell (by name)"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/sources/orders" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" }
    ```

**Response (200):**

```json
{
  "id": "a1b2c3d4-...",
  "name": "orders",
  "source_type": "postgres",
  "connection": {
    "host": "host.docker.internal",
    "port": 5432,
    "dbname": "production",
    "user": "tdb_reader",
    "password": "***",
    "table": "orders",
    "schema": "public"
  },
  "description": "Order records",
  "tags": ["production", "finance"],
  "registered_by": "tdbk_...",
  "registered_at": "2026-05-22T09:00:00Z",
  "status": "active"
}
```

!!! note "Passwords are masked"
    Sensitive connection fields (`password`, `secret`, `token`) are returned as `"***"` 
    in API responses. The real values are stored securely in the registry and used
    only at query time.

Returns 404 if no source matches the name or UUID.

---

## Get schema

**Why you'd use this:** Inspect column names and types before writing a query.
This avoids trial-and-error SQL errors and helps AI agents understand the shape of
your data. Accepts the source **name** or UUID.

```
GET /v1/sources/<source_id_or_name>/schema
```

=== "Bash (by name)"

    ```bash
    curl http://localhost:8000/v1/sources/orders/schema \
      -H "Authorization: Bearer <YOUR_KEY>"
    ```

=== "Bash (by UUID)"

    ```bash
    curl http://localhost:8000/v1/sources/a1b2c3d4-e5f6-.../schema \
      -H "Authorization: Bearer <YOUR_KEY>"
    ```

=== "PowerShell (by name)"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/sources/orders/schema" `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" }
    ```

Introspects the live table schema. For PostgreSQL, queries `information_schema.columns`.
Validates that the connection is reachable before returning — returns 503 if the
backend is down.

The response shape depends on how the source was registered:

**Response (200) — single-table source:**

```json
{
  "source_id": "a1b2c3d4-...",
  "source_name": "orders",
  "columns": [
    {"name": "id", "type": "integer"},
    {"name": "customer_id", "type": "integer"},
    {"name": "total", "type": "numeric"},
    {"name": "status", "type": "character varying"},
    {"name": "created_at", "type": "timestamp without time zone"}
  ],
  "tables": null,
  "inspected_at": "2026-05-22T09:10:00Z"
}
```

**Response (200) — database-wide source:**

```json
{
  "source_id": "b2c3d4e5-...",
  "source_name": "production_db",
  "columns": [],
  "tables": [
    {
      "name": "customers",
      "columns": [
        {"name": "id", "type": "integer"},
        {"name": "email", "type": "text"},
        {"name": "created_at", "type": "timestamp without time zone"}
      ]
    },
    {
      "name": "orders",
      "columns": [
        {"name": "id", "type": "integer"},
        {"name": "customer_id", "type": "integer"},
        {"name": "total", "type": "numeric"},
        {"name": "status", "type": "character varying"}
      ]
    }
  ],
  "inspected_at": "2026-05-22T09:10:00Z"
}
```

### Descriptions in the schema response

!!! info "From 0.7.0"

If you have set [schema annotations](#schema-annotations), they are included
here as well as in MCP:

```json
{
  "source_id": "a1b2c3d4-...",
  "source_name": "customers",
  "description": "One row per retail customer account.",
  "source_description": "Prod replica, read-only",
  "columns": [
    {"name": "cust_stat_cd_01", "type": "VARCHAR",
     "description": "Account status. A=active, C=closed, S=suspended."},
    {"name": "amt", "type": "BIGINT", "description": null}
  ],
  "tables": null,
  "inspected_at": "2026-08-11T09:10:00Z"
}
```

| Field | Where it comes from |
|---|---|
| `columns[].description` | Column annotation |
| `tables[].description` | Table annotation (database-wide sources) |
| `description` | Table annotation (single-table sources) |
| `source_description` | The `description` given when the source was registered |

Each is `null` when unset, so a deployment with no annotations sees exactly the
response shown above this section.

**Error responses:**

| Status | Meaning |
|---|---|
| 404 | Source name or UUID not found |
| 503 | Source is registered but the backend database is unreachable |

!!! warning "Fixed in 0.7.0: this endpoint returned 500 for CSV sources"

    On **0.2.0 through 0.6.0**, requesting the schema of a CSV source returned
    `500`. Database connectors and the MCP `schema_source` tool were unaffected,
    so a deployment could hit this only through REST and only on CSV. Upgrade, or
    read a CSV source's schema through MCP in the meantime.

---

## Schema annotations

!!! info "Enterprise, from 0.4.0"

**Why you'd use this:** Column names like `cust_stat_cd_01` mean nothing to an AI
assistant, which will guess — and a plausible guess yields a query that runs and
returns the wrong number. Annotations attach a human description to a table or
column; the MCP [`schema_source`](mcp.md#annotations-telling-the-model-what-a-column-means)
tool passes them to the model along with the names and types.

All three endpoints accept the source **name** or UUID.

### Set annotations

```http
PUT /v1/sources/{ref}/annotations
```

Requires the `readwrite` or `admin` [role](../security/rbac.md).

```bash
curl -X PUT http://localhost:8000/v1/sources/customers/annotations \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "annotations": [
      {"description": "One row per retail customer account."},
      {"column": "cust_stat_cd_01",
       "description": "Account status. A=active, C=closed, S=suspended."},
      {"table": "orders", "column": "amt_usd_cts",
       "description": "Order total in US cents."}
    ]
  }'
```

| Field | Required | Meaning |
|---|---|---|
| `description` | Yes | 1–2000 characters |
| `column` | No | Column this describes. Omit to describe the table itself. |
| `table` | No | Table this applies to. Omit for a single-table source; use it to scope an annotation on a **database-wide** source. |

Annotations are upserted: repeating a `table`/`column` pair replaces its
description, and pairs you do not mention are left alone.

```json
{"source_id": "3f1c...", "written": 3, "validated": true}
```

#### Targets are checked against the live schema

!!! info "From 0.7.0"

An annotation on a misspelled column matches nothing, so it simply never appears
— a failure you would discover much later, if at all. Targets are therefore
checked when you write them:

```json
{
  "detail": {
    "error": "unknown_annotation_target",
    "message": "Some annotations name a table or column this source does not have. Nothing was written. Re-send with ?validate=false to store them anyway.",
    "problems": ["column 'cust_stat_cd_1' does not exist (available: ['amt', 'cust_stat_cd_01'])"]
  }
}
```

The request is rejected with **400** and **nothing is written** — there is no
partial state to reconcile against what you sent.

Two cases deliberately still succeed:

- **`?validate=false`** stores the annotation without checking, for a column that
  has not shipped yet.
- **An unreachable source.** If the schema cannot be read, the write succeeds and
  the response reports `"validated": false`. A database being down is not
  evidence that your annotation is wrong, and refusing would make documenting a
  table depend on that table being up.

`validated` tells you which happened: `true` means every target was confirmed
against the live schema.

### List annotations

```http
GET /v1/sources/{ref}/annotations
```

```json
{
  "source_id": "3f1c...",
  "annotations": [
    {"table": "", "column": "", "description": "One row per retail customer account."},
    {"table": "", "column": "cust_stat_cd_01", "description": "Account status. A=active, C=closed, S=suspended."}
  ]
}
```

An empty `table` or `column` means "not scoped to one".

### Delete annotations

```http
DELETE /v1/sources/{ref}/annotations?column=cust_stat_cd_01
DELETE /v1/sources/{ref}/annotations?table=orders&column=amt_usd_cts
DELETE /v1/sources/{ref}/annotations?all=true
```

Requires `readwrite` or `admin`. Deleting a specific annotation that does not
exist returns 404; `all=true` removes every annotation for the source and reports
how many went.

| Status | Meaning |
|---|---|
| 200 | Applied |
| 403 | Caller lacks the `readwrite` role (write and delete only) |
| 404 | Source not found, or no such annotation to delete |
| 422 | Missing/empty `description`, or over 2000 characters |

---

## Delete a source

**Why you'd use this:** Remove a source that is no longer needed or was registered
incorrectly. Does not affect the underlying database. Accepts the source **name** or
UUID.

```
DELETE /v1/sources/<source_id_or_name>
```

=== "Bash (by name)"

    ```bash
    curl -X DELETE http://localhost:8000/v1/sources/orders \
      -H "Authorization: Bearer <YOUR_KEY>"
    ```

=== "Bash (by UUID)"

    ```bash
    curl -X DELETE http://localhost:8000/v1/sources/a1b2c3d4-e5f6-... \
      -H "Authorization: Bearer <YOUR_KEY>"
    ```

=== "PowerShell (by name)"

    ```powershell
    Invoke-RestMethod -Uri "http://localhost:8000/v1/sources/orders" `
      -Method DELETE `
      -Headers @{ Authorization = "Bearer <YOUR_KEY>" }
    ```

Expected response: **HTTP 204 No Content** (empty body — success).

**Responses:**

| Status | Meaning |
|---|---|
| 204 | Source deleted |
| 404 | Source name or UUID not found |
