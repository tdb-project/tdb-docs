# Prometheus Metrics

TDB exposes a Prometheus-compatible `/metrics` endpoint out of the box. Scrape it with any Prometheus instance to track request throughput, latency, MCP tool usage, and schema cache efficiency.

---

## Endpoint

```
GET /metrics
```

No authentication required. Restrict access via network policy in production — the endpoint is intentionally public so Prometheus can scrape it without credentials.

```bash
curl http://localhost:8000/metrics
```

The response is in standard Prometheus text exposition format.

---

## Available metrics

### HTTP requests

| Metric | Type | Labels | Description |
|---|---|---|---|
| `tdb_requests_total` | Counter | `method`, `endpoint`, `status_code` | Total HTTP requests, by method, path, and response status |
| `tdb_request_latency_seconds` | Histogram | `method`, `endpoint` | Request latency in seconds |

**Latency buckets:** 5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s.

### MCP tools

| Metric | Type | Labels | Description |
|---|---|---|---|
| `tdb_mcp_tool_calls_total` | Counter | `tool_name`, `status` | MCP tool call count by tool name and outcome (`ok` or `error`) |

### Schema cache

| Metric | Type | Description |
|---|---|---|
| `tdb_schema_cache_hits_total` | Counter | Schema lookups served from the in-process cache |
| `tdb_schema_cache_misses_total` | Counter | Schema lookups that required a live DB introspection |

Cache hit rate = `tdb_schema_cache_hits_total / (tdb_schema_cache_hits_total + tdb_schema_cache_misses_total)`.

---

## Prometheus scrape config

Add TDB to your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: tdb
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: /metrics
    scrape_interval: 15s
```

---

## Grafana dashboard (starter queries)

**Request rate (req/s):**

```promql
rate(tdb_requests_total[5m])
```

**p95 request latency:**

```promql
histogram_quantile(0.95, rate(tdb_request_latency_seconds_bucket[5m]))
```

**MCP tool error rate:**

```promql
rate(tdb_mcp_tool_calls_total{status="error"}[5m])
  /
rate(tdb_mcp_tool_calls_total[5m])
```

**Schema cache hit rate:**

```promql
rate(tdb_schema_cache_hits_total[5m])
  /
(rate(tdb_schema_cache_hits_total[5m]) + rate(tdb_schema_cache_misses_total[5m]))
```

**4xx / 5xx error rate:**

```promql
rate(tdb_requests_total{status_code=~"[45].."}[5m])
```

---

## Schema cache configuration

The schema cache avoids repeated introspection calls to the underlying database. Schemas are cached in-process with a configurable TTL.

| Variable | Default | Description |
|---|---|---|
| `TDB_SCHEMA_CACHE_TTL` | `300` | Cache TTL in seconds. Set to `0` to disable. |

The cache is keyed by source ID and invalidated automatically when a source is deleted via `DELETE /v1/sources/{id}`.

---

## Health check

```
GET /health
```

Returns HTTP 200 with `{"status": "ok"}` when TDB is running. No authentication required. Use this for load balancer health probes.

```bash
curl http://localhost:8000/health
```
