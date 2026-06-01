# Sources API

The Sources API manages the registry of data sources. All endpoints require authentication
(`Authorization: Bearer <token>`).

Base path: `/v1/sources`

---

## Register a source

```
POST /v1/sources
```

**Request body:**

```json
{
  "name": "orders",
  "source_type": "postgres",
  "connection": {
    "host": "db.internal",
    "port": 5432,
    "dbname": "production",
    "user": "tdb_reader",
    "password": "s3cret",
    "table": "orders",
    "schema": "public"
  },
  "description": "Order records",
  "tags": ["production", "finance"]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string (1–100) | Yes | Unique human-readable identifier |
| `source_type` | string | Yes | Connector type. Currently: `"postgres"` or `"csv"` |
| `connection` | object | Yes | Connector-specific config — see [PostgreSQL →](../connectors/postgresql.md) |
| `description` | string (max 500) | No | Human-readable description |
| `tags` | array of strings | No | Arbitrary labels for organisation |

**Responses:**

| Status | Meaning |
|---|---|
| 201 | Source registered. Returns full `SourceRecord`. |
| 400 | Invalid `source_type` — type not registered in this build. |
| 409 | A source with this name already exists. |

**Response body (201):**

```json
{
  "id": "a1b2c3d4-e5f6-...",
  "name": "orders",
  "source_type": "postgres",
  "connection": { ... },
  "description": "Order records",
  "tags": ["production", "finance"],
  "registered_by": "tdbk_...",
  "registered_at": "2026-05-22T09:00:00Z",
  "status": "active"
}
```

---

## List sources

```
GET /v1/sources
```

Returns a summary list (no `connection` details).

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

```
GET /v1/sources/<source_id>
```

Returns the full source record including `connection` details.

**Response (200):**

```json
{
  "id": "a1b2c3d4-...",
  "name": "orders",
  "source_type": "postgres",
  "connection": {
    "host": "db.internal",
    "port": 5432,
    "dbname": "production",
    "user": "tdb_reader",
    "password": "s3cret",
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

Returns 404 if the source ID is not found.

---

## Get schema

```
GET /v1/sources/<source_id>/schema
```

Introspects the live table schema. For PostgreSQL, queries `information_schema.columns`.
Validates the connection is reachable before returning.

**Response (200):**

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
  "inspected_at": "2026-05-22T09:10:00Z"
}
```

**Error responses:**

| Status | Meaning |
|---|---|
| 404 | Source ID not found |
| 503 | Source is registered but the backend is unreachable |

---

## Delete a source

```
DELETE /v1/sources/<source_id>
```

Removes the source from the registry. Does not affect the underlying database.
Subsequent queries against this `source_id` will return 404.

**Responses:**

| Status | Meaning |
|---|---|
| 204 | Source deleted |
| 404 | Source ID not found |
