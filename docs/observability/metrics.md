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

### Source queries

| Metric | Type | Labels | Description |
|---|---|---|---|
| `tdb_source_query_duration_seconds` | Histogram | `source_type` | Time spent executing a query against a data source |

`source_type` is the registered source's type — `csv`, `postgres`, `mysql`,
`sqlserver` or `snowflake` — so a slow source can be identified without
correlating against your database's own monitoring. Queries arriving over REST,
MCP and named views are all counted.

This measures the source, not the request: it excludes auth, SQL validation and
response serialisation, all of which `tdb_request_latency_seconds` includes. A
large gap between the two is TDB overhead; a large value here is the source.

**Buckets:** 10ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s, 10s, 30s, 60s.

Queries that fail or time out **are** recorded, at the duration they ran for. A
query cancelled at `TDB_QUERY_TIMEOUT` appears as a sample near your timeout
setting, not as a missing one.

Queries rejected *before* reaching a source — a blocked write, an unknown source,
a `limit` above `TDB_MAX_ROWS` — are not recorded here, because no source was
queried. Those appear in the audit log as denials.

### Audit log

| Metric | Type | Description |
|---|---|---|
| `tdb_audit_append_duration_seconds` | Histogram | Time to append one signed entry to the audit chain |

Every query and every denial writes one entry, so this is on the hot path of
every request. It is measured including the wait for the log's internal lock, so
concurrent load shows up here rather than being averaged away.

**Buckets:** 100µs, 250µs, 500µs, 1ms, 5ms, 10ms, 50ms, 100ms, 500ms, 1s, 5s.

**What healthy looks like:** sub-millisecond, and *flat as the log grows*. A p95
that climbs with log size is the signal to enable rotation
(`TDB_AUDIT_ROTATE_MAX_BYTES` / `TDB_AUDIT_ROTATE_MAX_DAYS`) — see
[audit retention](../security/audit.md).

### Connection pool

Only populated when `TDB_POOL_SIZE` is set (PostgreSQL sources — see
[the IT admin guide](../deployment/it-admin-guide.md)). With pooling off there is
no checkout to measure and nothing is recorded, so these describe pooled
deployments only.

| Metric | Type | Labels | Description |
|---|---|---|---|
| `tdb_pool_checkout_duration_seconds` | Histogram | — | Time waiting to check a connection out of the pool |
| `tdb_pool_checkout_failures_total` | Counter | `reason` | Checkouts that never returned a connection |

**Buckets:** 100µs, 1ms, 5ms, 10ms, 50ms, 100ms, 500ms, 1s, 5s, 10s.

A healthy pool checks out in microseconds. Rising waits mean the pool is
saturated — queries are queueing for a connection before they even reach
PostgreSQL. `tdb_pool_checkout_failures_total{reason="pool_timeout"}` is the same
condition after it has become an error, and a checkout that waited and then
failed is recorded in **both** metrics. The wait is the early warning; the
counter is the incident.

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

**p95 source query time, per source type:**

```promql
histogram_quantile(
  0.95,
  sum by (source_type, le) (rate(tdb_source_query_duration_seconds_bucket[5m]))
)
```

**Is TDB the overhead, or the source?** Compare the two p95s — if they track each
other, the time is in your data source, not in TDB.

```promql
histogram_quantile(0.95, sum by (le) (rate(tdb_source_query_duration_seconds_bucket[5m])))
/
histogram_quantile(0.95, sum by (le) (rate(tdb_request_latency_seconds_bucket[5m])))
```

**Audit append p95 — alert if this stops being flat:**

```promql
histogram_quantile(0.95, rate(tdb_audit_append_duration_seconds_bucket[5m]))
```

**Pool saturation (pooled deployments):**

```promql
histogram_quantile(0.95, rate(tdb_pool_checkout_duration_seconds_bucket[5m]))

rate(tdb_pool_checkout_failures_total[5m])
```

---

## Schema cache configuration

The schema cache avoids repeated introspection calls to the underlying database. Schemas are cached in-process with a configurable TTL.

| Variable | Default | Description |
|---|---|---|
| `TDB_SCHEMA_CACHE_TTL` | `300` | Cache TTL in seconds. Set to `0` to disable. |

The cache is keyed by source ID and invalidated automatically when a source is deleted via `DELETE /v1/sources/{id}`.

---

## Service endpoints

Three endpoints that are not about your data.

**Only `/health`, `/` and `/metrics` keep answering when the licence has
expired.** Everything else, including `/v1/version`, returns
`403 license_expired`. That is what makes `/` the endpoint to reach for when you
need a version out of a container whose licence has lapsed.

### Health check

```
GET /health
```

Returns HTTP 200 with `{"status": "ok"}` when TDB is running. No authentication required. Use this for load balancer health probes.

```bash
curl http://localhost:8000/health
```

**It stays green on an expired licence** — deliberately, so an orchestrator does
not kill a container that only needs its licence renewed. Data and API routes
return `403 license_expired` instead. So `/health` alone will not tell you the
licence has lapsed: watch the startup log, or treat a `403 license_expired` from
any data route as the signal.

### Version

```
GET /v1/version
```

Requires any valid API key, **and a valid licence** — on an expired licence this
returns `403 license_expired` rather than a version. Reports the exact patch
level **and the source commit** the image was built from:

```bash
curl -s -H "Authorization: Bearer <KEY>" http://localhost:8000/v1/version
```

```json
{"version": "0.8.0", "build_sha": "e10d31d"}
```

This is the authoritative answer to "what am I running" — worth quoting in a
support request, and worth checking against
[Security & Updates](../security/updates.md) when a fix is announced. The same
values appear in the container's startup log as `tdb_startup version=… build_sha=…`.

### Version banner

```
GET /
```

No authentication, and **not licence-gated** — this still answers when
`/v1/version` is returning `403 license_expired`. It omits `build_sha`, so prefer
`/v1/version` when identifying an exact build, and fall back to this when the
licence has lapsed.

```json
{"product": "The Data-Bridge", "version": "0.8.0", "status": "running", "docs": "/docs"}
```
