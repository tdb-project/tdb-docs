# YAML Views

Views are pre-approved, named SQL queries defined in YAML files. They let you publish a curated set of data access patterns — with typed parameters and descriptions — without exposing raw SQL to API consumers.

Use views to:

- Give AI tools and dashboards clean, stable query interfaces
- Enforce read patterns (consumers call a view name, not arbitrary SQL)
- Document data access for compliance reviewers

---

## Enabling views

Set `TDB_VIEWS_DIR` to a directory containing your view YAML files. Views are loaded at startup. If the variable is not set, the views feature is disabled.

```bash
TDB_VIEWS_DIR=/etc/tdb/views
```

---

## View definition format

Each view is a separate `.yml` or `.yaml` file. One file = one view.

**Minimal view (no parameters):**

```yaml
name: daily_signups
description: "New user signups grouped by day"
source: analytics_db
sql: |
  SELECT DATE(created_at) AS day, COUNT(*) AS signups
  FROM users
  GROUP BY day
  ORDER BY day DESC
```

**Parameterised view:**

```yaml
name: orders_by_status
description: "Orders filtered by status and capped at a row limit"
source: orders_db
sql: "SELECT * FROM orders WHERE status = :status LIMIT :limit"
parameters:
  - name: status
    type: string
    description: "Order status (pending, shipped, delivered)"
    required: true
  - name: limit
    type: integer
    description: "Maximum number of rows to return"
    required: false
    default: 50
```

### Fields

| Field | Required | Description |
|---|---|---|
| `name` | Yes | View name (1–128 chars). Must be unique across all view files. |
| `description` | No | Human-readable description shown in API responses |
| `source` | Yes | Registered source **name** (not ID) to execute the query against |
| `sql` | Yes | SQL query template. Use `:param_name` placeholders for parameters. |
| `parameters` | No | List of parameter definitions (see below) |

### Parameter fields

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Parameter name (1–64 chars). Must match a `:placeholder` in the SQL. |
| `type` | No | One of `string`, `integer`, `float`, `boolean`. Default: `string`. |
| `description` | No | Human-readable description |
| `required` | No | If `true`, callers must supply a value. Default: `false`. |
| `default` | No | Default value used when the parameter is omitted. Ignored if `required: true`. |

---

## Parameter types and SQL safety

Parameter values are type-checked and safely embedded before execution. TDB does not use string interpolation — each type produces a validated SQL literal:

| Type | Input | SQL output |
|---|---|---|
| `string` | `"O'Brien"` | `'O''Brien'` (single quotes escaped) |
| `integer` | `"42"` or `42` | `42` |
| `float` | `"3.14"` or `3.14` | `3.14` |
| `boolean` | `"true"`, `"1"`, `"yes"` | `1` |
| `boolean` | `"false"`, `"0"`, `"no"` | `0` |

Supplying a value that cannot be coerced to the declared type returns HTTP 400.

---

## REST endpoints

All view endpoints require Bearer auth (any valid API key with `read` role or above).

### List views

```
GET /v1/views
```

Returns all view definitions loaded from `TDB_VIEWS_DIR`.

```bash
curl -H "Authorization: Bearer <KEY>" http://localhost:8000/v1/views
```

**Response:**

```json
[
  {
    "name": "daily_signups",
    "description": "New user signups grouped by day",
    "source": "analytics_db",
    "parameters": []
  },
  {
    "name": "orders_by_status",
    "description": "Orders filtered by status and capped at a row limit",
    "source": "orders_db",
    "parameters": [
      {"name": "status", "type": "string", "description": "Order status (pending, shipped, delivered)", "required": true, "default": null},
      {"name": "limit", "type": "integer", "description": "Maximum number of rows to return", "required": false, "default": 50}
    ]
  }
]
```

### Get a view

```
GET /v1/views/{name}
```

Returns the definition for a single view. Returns HTTP 404 if not found.

### Run a view

```
POST /v1/views/{name}/run
```

**Request body:**

```json
{
  "parameters": {"status": "pending"},
  "limit": 100
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `parameters` | object | No | Key-value pairs for named parameters. |
| `limit` | integer (1–1000) | No | Max rows to return. Default: 1000. |

**Response:**

```json
{
  "view": "orders_by_status",
  "source": "orders_db",
  "columns": ["id", "customer_id", "status", "amount"],
  "rows": [
    {"id": 1, "customer_id": 42, "status": "pending", "amount": 99.99}
  ],
  "rows_returned": 1,
  "executed_at": "2026-05-22T09:01:23.456789Z"
}
```

```bash
curl -X POST http://localhost:8000/v1/views/orders_by_status/run \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{"parameters": {"status": "pending"}, "limit": 100}'
```

---

## MCP tool

Views are also callable via the MCP `list_views` and `run_view` tools when using Claude Desktop, Cursor, or any MCP client.

```
list_views()               → returns all view names and descriptions
run_view(name, params)     → executes the named view
```

The MCP tools respect the same `allowed_tools` restrictions as REST endpoints. See [RBAC → Restricting MCP tool access](../security/rbac.md#restricting-mcp-tool-access).

---

## Error responses

| Scenario | HTTP status | Detail |
|---|---|---|
| View not found | 404 | `View 'name' not found.` |
| Missing required parameter | 400 | `Missing required parameters: ['status']` |
| Unknown parameter supplied | 400 | `Unknown parameters: ['typo']` |
| Type coercion failure | 400 | `Parameter 'limit' expects integer, got 'abc'` |
| Source not registered | 400 | `View 'name' references source 'x' which is not registered.` |

---

## Example: full setup

```bash
# 1. Create view files
mkdir /etc/tdb/views

cat > /etc/tdb/views/revenue_by_month.yml << 'EOF'
name: revenue_by_month
description: "Monthly revenue totals, optionally filtered by year"
source: orders_db
sql: |
  SELECT strftime('%Y-%m', order_date) AS month, SUM(amount) AS total
  FROM orders
  WHERE (:year = 0 OR strftime('%Y', order_date) = :year)
  GROUP BY month
  ORDER BY month DESC
parameters:
  - name: year
    type: string
    description: "4-digit year to filter by, or '0' for all years"
    required: false
    default: "0"
EOF

# 2. Start TDB with the views directory
TDB_VIEWS_DIR=/etc/tdb/views tdb serve

# 3. List views
curl -H "Authorization: Bearer <KEY>" http://localhost:8000/v1/views

# 4. Run a view
curl -X POST http://localhost:8000/v1/views/revenue_by_month/run \
  -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{"parameters": {"year": "2026"}}'
```

---

## Reload behavior

Views are loaded once at startup. To add, remove, or modify view files, restart TDB. There is no hot-reload — a restart ensures the audit log records which view definition was active at query time.
