# CLI reference

TDB ships a command-line client alongside the server. It talks to a **running**
TDB over HTTP — it is not a second way into your data, and every rule the REST
API enforces (read-only SQL, the row ceiling, RBAC) applies identically here.

```bash
tdb serve       # start the REST + MCP server
tdb register    # register a CSV file as a source
tdb query       # run a read-only SQL query
```

## Environment

| Variable | Purpose | Default |
|---|---|---|
| `TDB_API_KEYS` | The key the CLI authenticates with. **Required** for `register` and `query` — the first key in the list is used. | — |
| `TDB_URL` | Server to talk to. | `http://localhost:8000` |
| `TDB_PORT` | Port for `tdb serve`. | `8000` |

Without `TDB_API_KEYS` the CLI fails before building a request and tells you
which variable to set.

---

## `tdb serve`

Starts the server. Hands control to uvicorn and does not return.

| Option | Default | Notes |
|---|---|---|
| `--host` | `127.0.0.1` | Use `0.0.0.0` to accept connections from the network. Prefer the Docker deployment for anything beyond local use. |
| `--port`, `-p` | `8000` | Or set `TDB_PORT`. |
| `--reload` | off | Development only. |

---

## `tdb register`

Registers a CSV file as a source. The server must already be running.

```bash
tdb register /data/sales.csv --name sales_q1 --tags "eu,finance"
```

| Option | Notes |
|---|---|
| `--name`, `-n` | **Required.** Unique source name. |
| `--description`, `-d` | Free text. |
| `--tags` | Comma-separated. Blank entries and surrounding spaces are dropped. |

The path must exist and is resolved to an absolute path before being sent. A
file without a `.csv` extension produces a warning but still registers — the
connector reads the contents, not the name. A missing file fails before any
request is made.

!!! note "Path confinement"
    When `TDB_ALLOWED_DATA_DIR` is set, the *server* rejects a path outside it.
    The CLI sends the path; it does not decide what is allowed. See
    [Environment Variables](environment-variables.md).

---

## `tdb query`

Runs one read-only SQL statement.

```bash
tdb query "SELECT country, COUNT(*) FROM data GROUP BY country"
tdb query "SELECT * FROM data" --source sales_q1 --output json
```

| Option | Default | Notes |
|---|---|---|
| `--source`, `-s` | — | Source **name or UUID**. See below. |
| `--limit`, `-l` | `100` | Maximum rows. The **server** enforces the ceiling. |
| `--output`, `-o` | `table` | `table`, `json` or `csv`. An unrecognised value is an error. |

**Table names.** CSV sources are queried as `data`. Database sources use their
real table name — the SQL is passed to the database essentially unchanged.

### Choosing the source

`--source` accepts either the registered name or the UUID; the server resolves
both, so you can use whichever you have.

When you omit it:

- **one source registered** — that source is used;
- **more than one** — the command stops and lists them, rather than guessing.

Naming a source also **skips the source listing request**, so `tdb query` works
with a key that has no permission to list sources.

!!! warning "Changed behaviour"
    Before this release, `tdb query` always used the **first registered source**
    and had no way to say otherwise. On a deployment with several sources every
    other one was unreachable from the CLI, and the result was indistinguishable
    from a correct one. If you relied on that implicit first-source behaviour
    with more than one source registered, pass `--source` explicitly.

### The row ceiling

`--limit` is sent as you typed it. If it exceeds the deployment's ceiling the
**request is refused**, not quietly shortened:

- **Community** — the ceiling is a fixed **1,000**.
- **Enterprise** — the ceiling is `TDB_MAX_ROWS` (default 1,000), which an
  operator can raise.

!!! warning "Changed behaviour"
    The CLI used to clamp `--limit` to 1,000 before sending, so a larger value
    returned 1,000 rows that looked like a complete result — and on an
    enterprise deployment that had raised `TDB_MAX_ROWS`, the extra rows were
    unreachable from the CLI with nothing explaining why. A request above the
    ceiling is now an error. This matches the REST API, which has always
    rejected rather than clamped so that a caller never mistakes a cut result
    for a whole one.

### Output formats

`table` is for reading. **`json` is the only format that distinguishes a SQL
NULL from an empty string** — `table` and `csv` both render NULL as an empty
cell, which is the correct display but is lossy.

```console
$ tdb query "SELECT id, country FROM data" --output json
[{"id": 1, "country": "UK"}, {"id": 2, "country": null}]
```

---

## Exit codes

`0` on success. `1` on any failure — a missing API key, an unreachable server, a
refused query, an unknown output format, or an ambiguous source. Errors go to
stderr, so `tdb query ... --output csv > out.csv` writes only rows to the file.
