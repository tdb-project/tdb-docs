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
  "error": "hash mismatch at seq=42",
  "at_seq": 42
}
```

```bash
curl -H "Authorization: Bearer <ADMIN_KEY>" http://localhost:8000/v1/audit/verify
```

Run this check as part of your daily compliance job, or trigger it after any log rotation.

---

## Exporting to a SIEM

Use `POST /v1/audit/export?destination=splunk|s3` to ship the log onward. Exports are **incremental** — each destination resumes where it left off, so a scheduled export does not re-send history. `GET /v1/audit/export/status` shows how far each has consumed.

- [Splunk HEC integration →](../integrations/splunk.md)
- [S3 audit archive →](../integrations/s3.md)

### Synchronous or backgrounded

By default the request **waits** for the export to finish and returns what it
shipped, which is what a cron job checking `errors` needs:

```json
{"exported": 2847, "skipped": 0, "errors": [], "disabled": false}
```

A first export of a large backlog can take a while. Pass `wait=false` to get
`202` straight away and let it run in the background:

```bash
curl -fsS -X POST "http://localhost:8000/v1/audit/export?destination=s3&wait=false" \
  -H "Authorization: Bearer $ADMIN_KEY"
```

```json
{"accepted": true, "destination": "s3", "detail": "Export started. Read the outcome from GET /v1/audit/export/status."}
```

Nothing is waiting on that run, so its outcome is recorded instead of returned.
`GET /v1/audit/export/status` reports the last run for every destination —
including synchronous ones, so the field means the same thing either way:

```json
{
  "destinations": [
    {
      "name": "s3",
      "configured": true,
      "last_exported_seq": 4210,
      "last_run": {
        "state": "succeeded",
        "started_at": "2026-08-12T04:10:02+00:00",
        "finished_at": "2026-08-12T04:10:39+00:00",
        "exported": 120,
        "skipped": 0,
        "errors": []
      }
    }
  ]
}
```

`state` is `running`, `succeeded` or `failed`; `last_run` is `null` for a
destination that has never exported.

!!! warning "One run per destination at a time"

    A second export to a destination that is already running returns **409**.
    Both runs would read the same cursor and ship the same entries, so
    overlapping them duplicates events in your SIEM.

    This matters most with `wait=false`: a cron firing every five minutes against
    an export that takes ten will collide. Either widen the interval past the
    slowest run, or treat 409 as "the previous run is still going" and skip.

    A run whose process is killed is treated as abandoned after **one hour**, so a
    crash cannot block a destination permanently.

---

## Configuration

| Variable | Default | Description |
|---|---|---|
| `TDB_LOG_FILE` | `tdb_audit.jsonl` | Path to the NDJSON audit log |
| `TDB_LOG_LEVEL` | `INFO` | Log verbosity (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |
| `TDB_AUDIT_ROTATE_MAX_BYTES` | `0` (off) | Seal the log into a segment once it exceeds this size |
| `TDB_AUDIT_ROTATE_MAX_DAYS` | `0` (off) | Seal the log once its oldest entry is older than this |

---

## Retention: sealed segments

!!! info "Enterprise, from 0.4.0"
    Before 0.4.0 the log grew without bound and rotation was entirely manual.

**TDB never deletes an audit entry.** Retention here decides when the live log is
*sealed*, not when anything is dropped.

When a threshold is crossed, TDB closes the live log with a final `rotated` entry,
renames it to a timestamped segment, and starts a fresh log:

```
tdb_audit.jsonl                         ← live chain
tdb_audit-20260801T031500Z-seq48210.jsonl   ← sealed segment
tdb_audit-20260715T014500Z-seq22119.jsonl   ← sealed segment
```

Each segment keeps its chain intact and verifies on its own. The new live log opens
with a `chain_anchor` entry naming the segment it follows and that segment's final
hash, so the segments form a verifiable sequence rather than disconnected files.

Rotation is evaluated when an entry is written and once at startup. Set either
variable, both, or neither; whichever threshold is crossed first seals the log.
You can also seal on demand with `POST /v1/audit/rotate` (admin).

!!! tip "Why you should enable this"
    **Not for speed — that argument expired in 0.8.0.** Every audit write needs the
    previous entry's hash. 0.4.0 made that independent of history for a running
    server, and 0.8.0 made the first write after a restart independent of it too, by
    reading the chain head backwards from the end of the file (measured at 500,000
    entries: 4.3 s before, under a millisecond after, and flat as the log grows).
    A large live log no longer slows startup or the first request.

    Enable rotation for the reasons that remain: **a multi-gigabyte single file is
    awkward to back up, copy and retain**, and sealed segments are what let you
    prune old history while keeping every remaining segment independently
    verifiable. Size the live log to whatever your backup story wants.

### Verifying across segments

`GET /v1/audit/verify` proves the **live** log has not been altered. It cannot prove
a whole segment was not removed, because every segment legitimately restarts its
chain from genesis.

`GET /v1/audit/history` walks the sealed segments, the live log, and the anchors
linking them:

```json
{ "valid": true, "segments": 4, "entries": 128340,
  "segment_files": ["tdb_audit-20260715T014500Z-seq22119.jsonl", "..."] }
```

Removing a segment from the **middle** of the sequence is reported as invalid.
Pruning the **oldest** segments is a supported retention action and stays valid —
that is how you actually delete old audit data.

## Backup

- Back up sealed segments to immutable object storage (S3, GCS). They never change
  again, which makes them well suited to write-once buckets.
- Run `GET /v1/audit/history` before pruning anything, to confirm the sequence is
  intact first.

!!! warning "Do not edit the log file"
    Any modification — including deleting lines or reordering entries — will break the hash chain and fail verification. Treat the log as write-once. Use `POST /v1/audit/rotate` rather than truncating the file yourself: rotation is recorded *in* the chain, hand-truncation is indistinguishable from tampering.

---

## Legacy entries

Entries written by TDB versions before hash-chaining was introduced do not contain `seq` or `hash` fields. The verifier skips these entries and starts chain validation from the first signed entry it encounters. Legacy and signed entries can coexist in the same file.
