# Audit Log & Tamper Verification

TDB writes a tamper-evident, hash-chained audit log for every query. Every request — REST or MCP — produces one entry. The log is the primary compliance artifact for SOC 2, HIPAA, and EU AI Act readiness.

---

## Log format

The log is NDJSON (newline-delimited JSON), one JSON object per line. Default path: `tdb_audit.jsonl`. Override with `TDB_LOG_FILE`.

**Entry schema:**

```json
{
  "ts": "2026-05-22T09:01:23.456789+00:00",
  "source_id": "f47ac10b-...",
  "sql": "SELECT COUNT(*) FROM orders",
  "rows_returned": 1,
  "api_key": "tdbk_a1b2...",
  "seq": 42,
  "prev_hash": "a3f1...",
  "hash": "9c82..."
}
```

| Field | Description |
|---|---|
| `ts` | ISO 8601 timestamp (UTC) of when the query executed |
| `source_id` | UUID of the registered source that was queried |
| `sql` | The SQL statement executed (or `<view:name>` for named views) |
| `rows_returned` | Number of rows included in the response |
| `api_key` | First 12 characters of the key that made the request (prefix only) |
| `seq` | Monotonically increasing sequence number, starting from 1 |
| `prev_hash` | SHA-256 hash of the previous entry (genesis entry uses `000...0`) |
| `hash` | SHA-256 hash of this entry (computed over all fields except `hash` itself) |

---

## Hash chain

Each entry's `hash` is a SHA-256 digest of the entry's canonical JSON (all fields sorted, no whitespace, `hash` field excluded). The next entry includes the previous `hash` as `prev_hash`.

This creates a chain: to forge or delete any entry, an attacker would need to recompute every subsequent hash — and TDB detects the break during verification.

```
Entry 1: seq=1, prev_hash=000...0, hash=H1
Entry 2: seq=2, prev_hash=H1,      hash=H2
Entry 3: seq=3, prev_hash=H2,      hash=H3
```

Deleting entry 2 would make entry 3's `prev_hash` point to a non-existent hash. Adding a forged entry would produce the wrong `hash` value. Both are detected by `GET /v1/audit/verify`.

---

## Verifying the log

```
GET /v1/audit/verify
```

Requires **admin** role. Reads the entire log file and walks the hash chain.

**Response — valid log:**

```json
{
  "valid": true,
  "entries": 2847
}
```

**Response — tampered log:**

```json
{
  "valid": false,
  "error": "hash mismatch at seq 42: expected 9c82..., got a3f1...",
  "at_seq": 42
}
```

```bash
curl -H "Authorization: Bearer <ADMIN_KEY>" http://localhost:8000/v1/audit/verify
```

Run this check as part of your daily compliance job, or trigger it after any log rotation.

---

## Exporting to a SIEM

Use `POST /v1/audit/export` to push the log to Splunk. See [Splunk HEC integration →](../integrations/splunk.md).

---

## Configuration

| Variable | Default | Description |
|---|---|---|
| `TDB_LOG_FILE` | `tdb_audit.jsonl` | Path to the NDJSON audit log |
| `TDB_LOG_LEVEL` | `INFO` | Log verbosity (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |

---

## Backup and rotation

The audit log is append-only. TDB never truncates it. You are responsible for rotation and backup:

- Back up `tdb_audit.jsonl` to immutable object storage (S3, GCS) daily.
- Run `GET /v1/audit/verify` before rotation to confirm integrity.
- Keep old log files; the hash chain spans the entire history of a deployment.

!!! warning "Do not edit the log file"
    Any modification — including deleting lines or reordering entries — will break the hash chain and fail verification. Treat the log as write-once.

---

## Legacy entries

Entries written by TDB versions before hash-chaining was introduced do not contain `seq` or `hash` fields. The verifier skips these entries and starts chain validation from the first signed entry it encounters. Legacy and signed entries can coexist in the same file.
