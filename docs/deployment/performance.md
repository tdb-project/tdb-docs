# Performance

What we measured, how, and what changed because of it. Every number here was measured
on a six-core development host with the database on the same machine, so network
latency is close to zero. Treat them as the shape of TDB's behaviour, not a promise
for your hardware. Each measurement's version is given; later releases keep the fix.

---

## Memory is bounded by the row ceiling

A query returns at most the row ceiling, and TDB fetches at most that many rows (plus
one, to know whether more existed). It never reads the source's whole result into
memory and then cuts it.

| Measured | Before | After | From |
|---|---:|---:|---|
| PostgreSQL, 2,000,000-row result, 10 returned | 184 MB | **0.3 MB** | enterprise 0.2.1 |
| CSV, 200,000-row file | 30.9 MB | **0.01 MB** | community 0.4.4 |

Every connector reads a bounded batch: a server-side cursor on PostgreSQL, an unbuffered
cursor on MySQL, a limited fetch on SQL Server, Snowflake, CSV and Parquet. See
[capacity planning](it-admin-guide.md#8-performance-capacity).

## CSV and Parquet scale with concurrency

The file engine is one shared engine with a cursor per query, so concurrent queries share
threads instead of each claiming every core.

5.4 MB, 100,000-row CSV, `SELECT * FROM data LIMIT 10`:

| | Before (engine per query) | After (shared engine) |
|---|---:|---:|
| 1 query at a time | 228 ms | **131 ms** |
| 16 concurrent, median | 50,899 ms | **317 ms** |
| 16 concurrent, throughput | 0.35 req/s | **28.1 req/s** |

From enterprise 0.5.1 / community 0.4.6. The file is not loaded into memory: each query
reads it, so an edited file is seen straight away.

## The audit log costs the same at any size

Every request writes a signed, hash-chained audit entry. Writing one no longer depends on
how long the log already is.

| Entries already in the log | Before | After (from enterprise 0.4.0) |
|---:|---:|---:|
| 0 | 0.19 ms | 0.04 ms |
| 100,000 | 1,212 ms | 0.04 ms |
| 500,000 | 5,807 ms | 0.04 ms |

The first audited request after a restart reads the chain from the end of the file:
0.8 ms at 500,000 entries and 0.9 ms at 1,000,000 (enterprise 0.8.0; it took 4.3 s and
8.2 s before).

## Connection reuse

Opening a database connection is most of a short query's cost.

| Database | Measurement | Result |
|---|---|---|
| PostgreSQL | connect alone vs a query on an open connection (median of 30) | 57.6 ms vs **4.5 ms**: the handshake is 79% of a short query's database time |
| SQL Server | connect + close with ODBC pooling on (the default) vs off | **3.2 ms** vs 24.5 ms |

PostgreSQL connection pooling is opt-in with `TDB_POOL_SIZE` (enterprise 0.5.0), because
it keeps idle sessions open on a database you own; pooled sessions are reset to read-only
on every return. SQL Server's driver pools already. See
[Environment variables](../reference/environment-variables.md).

## Tokens an agent reads

Measured with Claude Code and Claude Opus 5.5 (community 0.13.0 / enterprise 0.21.0), in
tokens the model reads:

| Result | Default | Lean Mode |
|---|---:|---:|
| 100 rows | 10,412 | **6,868** |
| 30-table, 292-column schema | 5,235 | **3,025** (`types`), 1,400 (`names`), 113 (`tables`) |

On a 21-question accuracy check, Lean results gave the same answers. Across a whole agent
session that mostly writes aggregate queries, the saving was about 6%, because most
results are a few rows. See [Saving tokens](../api/mcp.md#saving-tokens-lean-mode).

## Timeouts

`TDB_QUERY_TIMEOUT` (default 30 s, enterprise) is enforced by the source database, which
cancels the query. Measured on MySQL, the cancellation landed 25–33 ms after the limit. A
cancelled query returns 504 and is audited as `query_timeout`.

---

Want a number we haven't published? Ask: [hello@tdb.jiracorp.co.in](mailto:hello@tdb.jiracorp.co.in).
