# Splunk HEC Integration

TDB can push its audit log to Splunk via the HTTP Event Collector (HEC). This lets your security team search, alert on, and archive every query in Splunk alongside other infrastructure events.

---

## How it works

`POST /v1/audit/export` reads the TDB audit log file and sends each entry to your Splunk HEC endpoint as a structured event. Entries are batched (100 per request) and timestamped using the `ts` field from each audit entry.

The export is **on-demand** — TDB does not stream to Splunk in real time. Schedule the export endpoint as a cron job, or call it from your SIEM pipeline on a fixed interval.

!!! info "Incremental from 0.4.0"
    Each run ships only what Splunk has not already received. TDB records how far
    each destination has consumed and resumes from there.

    **Before 0.4.0 every run re-sent the whole log from the first entry**, so a
    scheduled export duplicated all of history on every run. If you have been
    running the export on a cron against an earlier version, expect existing
    duplicates in your index — TDB cannot retract what it already sent. The first
    0.4.0 run starts from the beginning once (there is no cursor yet), then goes
    incremental.

    Pass `since_seq=0` to deliberately re-send everything.

---

## Prerequisites

1. A running Splunk instance (cloud or self-hosted).
2. An HTTP Event Collector (HEC) token. Create one in Splunk under **Settings → Data Inputs → HTTP Event Collector → New Token**.
3. Note the HEC endpoint URL and token.

---

## Configuration

Set these environment variables before starting TDB:

| Variable | Required | Description |
|---|---|---|
| `TDB_SPLUNK_HEC_URL` | Yes | Full HEC endpoint URL. Example: `https://splunk.corp.com:8088/services/collector/event` |
| `TDB_SPLUNK_HEC_TOKEN` | Yes | Splunk HEC authentication token |
| `TDB_SPLUNK_INDEX` | No | Target Splunk index. Omit to use the index configured on the HEC token. |
| `TDB_SPLUNK_SOURCETYPE` | No | sourcetype for exported events. Default: `tdb:audit` |
| `TDB_SPLUNK_VERIFY_TLS` | No | Set to `false` to skip TLS verification. Default: `true`. Only use in development. |

```bash
# In your .env
TDB_SPLUNK_HEC_URL=https://splunk.corp.com:8088/services/collector/event
TDB_SPLUNK_HEC_TOKEN=your-hec-token
TDB_SPLUNK_INDEX=tdb_audit
```

If `TDB_SPLUNK_HEC_URL` is not set, `POST /v1/audit/export` returns `{"disabled": true}` — safe default for deployments without Splunk.

---

## Running an export

```
POST /v1/audit/export
```

Requires **admin** role.

```bash
curl -X POST http://localhost:8000/v1/audit/export \
  -H "Authorization: Bearer <ADMIN_KEY>"
```

**Response — successful export:**

```json
{
  "exported": 2847,
  "skipped": 0,
  "errors": []
}
```

**Response — disabled (no HEC URL configured):**

```json
{"disabled": true}
```

!!! tip "Long exports"

    Add `&wait=false` to return `202` immediately and run the export in
    the background, then read the result from
    `GET /v1/audit/export/status`. A destination runs one export at a
    time — an overlapping call gets `409`. See
    [Exporting to a SIEM](../security/audit.md#synchronous-or-backgrounded).

**Response — partial failure:**

```json
{
  "exported": 2800,
  "skipped": 47,
  "errors": ["HEC returned HTTP 503: ..."]
}
```

---

## Splunk event format

Each audit entry is wrapped in a Splunk HEC event envelope:

```json
{
  "time": 1716374483.456,
  "sourcetype": "tdb:audit",
  "source": "tdb-audit",
  "index": "tdb_audit",
  "event": {
    "ts": "2026-05-22T09:01:23.456789Z",
    "source_id": "f47ac10b-...",
    "sql": "SELECT COUNT(*) FROM orders",
    "rows_returned": 1,
    "api_key": "tdbk_a1b2...",
    "seq": 42,
    "prev_hash": "a3f1...",
    "hash": "9c82..."
  }
}
```

The `time` field uses the Unix timestamp derived from the entry's `ts` field, so events appear at the correct position in Splunk's timeline even if the export runs hours after the query.

---

## Splunk search examples

**All queries in the last 24 hours:**

```
index=tdb_audit sourcetype=tdb:audit earliest=-24h
```

**Queries against a specific source:**

```
index=tdb_audit sourcetype=tdb:audit source_id="f47ac10b-..."
```

**Top 10 most active API keys:**

```
index=tdb_audit sourcetype=tdb:audit
| stats count by api_key
| sort -count
| head 10
```

**Queries returning more than 500 rows (potential data exfiltration):**

```
index=tdb_audit sourcetype=tdb:audit rows_returned>500
```

---

## Scheduling exports

Run the export on a schedule using cron or your orchestration tool:

```bash
# Export every 15 minutes via cron
*/15 * * * * curl -s -X POST http://localhost:8000/v1/audit/export \
  -H "Authorization: Bearer $TDB_ADMIN_KEY" >> /var/log/tdb-splunk-export.log 2>&1
```

Or via a systemd timer, Kubernetes CronJob, or your existing SIEM pipeline.

---

## Troubleshooting

**`{"disabled": true}`** — `TDB_SPLUNK_HEC_URL` is not set. Add it to your `.env` and restart TDB.

**`HEC returned HTTP 403`** — Invalid or revoked HEC token. Regenerate the token in Splunk and update `TDB_SPLUNK_HEC_TOKEN`.

**`HEC returned HTTP 400`** — The index specified in `TDB_SPLUNK_INDEX` does not exist in Splunk, or the HEC token is not permitted to write to it. Either create the index or remove `TDB_SPLUNK_INDEX` to use the token's default index.

**TLS errors** — If using a self-signed certificate on an internal Splunk instance, set `TDB_SPLUNK_VERIFY_TLS=false`. Do not use this in production with a public CA certificate.
