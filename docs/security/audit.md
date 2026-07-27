# Audit Log & Tamper Verification

TDB writes a tamper-evident, hash-chained audit log for every query **and every refused attempt**. Every request — REST or MCP — produces one entry. The log is the primary compliance artifact for SOC 2, HIPAA, and EU AI Act readiness.

---

## Log format

The log is NDJSON (newline-delimited JSON), one JSON object per line. Default path: `tdb_audit.jsonl`. Override with `TDB_LOG_FILE`.

There are two entry types, distinguished by `event`: `query` for a query that ran, and `denied` for an attempt that was refused. Both are signed into the same chain.

**`event: "query"` — a query that executed:**

```json
{
  "event": "query",
  "ts": "2026-05-22T09:01:23.456789+00:00",
  "source_id": "f47ac10b-...",
  "sql": "SELECT COUNT(*) FROM orders",
  "rows_returned": 1,
  "key_hint": "tdbk_a...",
  "seq": 42,
  "prev_hash": "a3f1...",
  "hash": "9c82..."
}
```

| Field | Description |
|---|---|
| `event` | `"query"` for an executed query, `"denied"` for a refused attempt |
| `ts` | ISO 8601 timestamp (UTC) of when the query executed |
| `source_id` | UUID of the registered source that was queried |
| `sql` | The SQL statement executed (or `<view:name>` for named views) |
| `rows_returned` | Number of rows included in the response (`query` entries only) |
| `key_hint` | First 6 characters of the key that made the request, then `...`. The raw key is never written to the log |
| `seq` | Monotonically increasing sequence number, starting from 1 |
| `prev_hash` | SHA-256 hash of the previous entry (genesis entry uses `000...0`) |
| `hash` | SHA-256 hash of this entry (computed over all fields except `hash` itself) |

---

## Denied attempts

A trail that records only what succeeded cannot answer the question an auditor
actually asks: *who tried what, and was turned away?* TDB records refusals as
first-class audit events, signed into the same chain, so a deleted denial breaks
verification exactly as a deleted query does.

**`event: "denied"` — an attempt that was refused:**

```json
{
  "event": "denied",
  "action": "query",
  "reason": "sql_validation_failed",
  "ts": "2026-05-22T09:01:24.881204+00:00",
  "source_id": "f47ac10b-...",
  "sql": "DROP TABLE orders",
  "key_hint": "tdbk_a...",
  "seq": 43,
  "prev_hash": "9c82...",
  "hash": "4d10..."
}
```

Two fields describe the refusal:

| Field | Description |
|---|---|
| `action` | What was attempted: `auth`, `authorize`, `query`, `register`, `ratelimit`, `mcp_auth`, `mcp_query`, or `mcp_call` |
| `reason` | Machine-readable cause — see the table below |

| `reason` | Raised when |
|---|---|
| `missing_credentials` | No `Authorization` header was sent |
| `invalid_api_key` | The key is not recognised |
| `invalid_or_expired_token` | A JWT failed signature or expiry validation |
| `insufficient_role_required_<role>_have_<role>` | RBAC refused the request (403) |
| `rate_limit_exceeded` | The key's per-minute budget was exhausted (429) |
| `sql_validation_failed` | The statement was not a permitted read-only `SELECT` |
| `prompt_injection_detected` | The prompt-injection filter rejected the input |
| `source_not_found` | The referenced source does not exist |
| `path_outside_allowed_dir` | A CSV path resolved outside `TDB_ALLOWED_DATA_DIR` |
| `file_unreadable` | The CSV backing a source is missing or unreadable |
| `unsupported_source_type` | The `source_type` is not a registered connector |
| `registry_conflict` | The source name is already taken |
| `tool_not_permitted_<tool>` | The key's `allowed_tools` scope excludes this MCP tool |

To review refusals only:

```bash
cat tdb_audit.jsonl | jq 'select(.event == "denied")'
```

To find keys probing for access:

```bash
cat tdb_audit.jsonl | jq -r 'select(.action == "auth") | .key_hint' | sort | uniq -c | sort -rn
```

!!! note "Community edition"
    tdb-community records the same `denied` events with the same `action` and
    `reason` fields. The hash chain (`seq`, `prev_hash`, `hash`) and
    `GET /v1/audit/verify` are enterprise features — community entries are
    unsigned NDJSON.

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
