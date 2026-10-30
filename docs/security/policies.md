# Column and Row Policies

**Mid-Market, from enterprise 0.22.0. Off unless `TDB_POLICIES_FILE` is set.**

A policy decides what each key sees of a source: which columns, which of them masked,
and which rows. Two keys can query the same table and get different answers, on every
path: `POST /v1/query`, every MCP tool, views, and the
[PostgreSQL wire gateway](../integrations/postgres-wire.md). The audit log records which
rule shaped each query.

TDB enforces a policy **in the database**: it rewrites the SQL so that every table the
query reads is replaced by a subquery holding only the allowed columns, masks and rows.
So `COUNT(*)`, `upper(ssn)`, joins and `WHERE ssn LIKE '9%'` all see policy data, not
the raw table. Nothing changes in your database.

---

## What it looks like

A `patients` table in PostgreSQL, and two keys. `clinician` sees everything.
`support-emea` sees only EMEA patients, the last four characters of each SSN, and no
diagnosis:

```yaml title="policies.yml"
version: 1
sources:
  clinic:                    # the registered source name
    default: deny            # a key no rule names gets nothing
    rules:
      - name: emea-support
        applies_to: {keys: [support-emea]}
        tables:
          patients:
            columns: [id, name, region, ssn]   # diagnosis is withheld
            mask: {ssn: partial}
            rows: "region = :region"           # the key's own region only
      - name: clinicians
        applies_to: {keys: [clinician]}
        tables:
          patients: {}                         # every column, every row
```

```bash
docker run ... -e TDB_POLICIES_FILE=/config/policies.yml -v /srv/tdb:/config:ro tdb-enterprise:<version>

# give support-emea its region (admin)
curl -X PUT http://localhost:8000/v1/auth/keys/<support-emea id>/attributes \
  -H "Authorization: Bearer <ADMIN_KEY>" -H "Content-Type: application/json" \
  -d '{"attributes": {"region": "EMEA"}}'
```

The same query, `SELECT * FROM patients ORDER BY id`, as each key:

=== "clinician"

    ```json
    {"id":1,"name":"Ann Lee","region":"EMEA","ssn":"123-45-6789","diagnosis":"asthma"}
    {"id":2,"name":"Bo Chen","region":"APAC","ssn":"987-65-4321","diagnosis":"diabetes"}
    {"id":3,"name":"Cy Okafor","region":"EMEA","ssn":"555-12-3456","diagnosis":"migraine"}
    {"id":4,"name":"Di Ruiz","region":"AMER","ssn":"222-33-4444","diagnosis":"hypertension"}
    ```

=== "support-emea (REST and MCP)"

    ```json
    {"id":1,"name":"Ann Lee","region":"EMEA","ssn":"***6789"}
    {"id":3,"name":"Cy Okafor","region":"EMEA","ssn":"***3456"}
    ```

=== "support-emea (psql, wire gateway)"

    ```text
     id |   name    | region |   ssn
    ----+-----------+--------+---------
      1 | Ann Lee   | EMEA   | ***6789
      3 | Cy Okafor | EMEA   | ***3456
    (2 rows)
    ```

For `support-emea`, `SELECT count(*) FROM patients` is `2`, and `SELECT diagnosis FROM
patients` fails with the database's own `column "diagnosis" does not exist`: a withheld
column is indistinguishable from one that is not there. A key no rule names, under
`default: deny`, is refused (`Refused by policy: no policy rule covers this key on this
source`). The audit entry for the `support-emea` query:

```json
{"event": "query", "policy_rule": "emea-support",
 "policy_masked": {"patients": ["ssn"]}, "policy_withheld": {"patients": ["diagnosis"]},
 "policy_row_filtered": ["patients"], "policy_sha256": "0b55e457d539…"}
```

The [governed access recipe](../recipes/governed-access.md) runs this end to end.

---

## The policy file

YAML, set with `TDB_POLICIES_FILE` and read once at startup. **An invalid file stops
TDB from starting**, with the reason: serving without the rules someone wrote is the
unsafe failure. An unknown field is an error, so a typo cannot silently turn a rule off.
Check a loaded file with [`GET /v1/policies`](#checking-a-policy-file).

| Field | Meaning |
|---|---|
| `version` | `1` |
| `sources.<name>` | A registered source, by name. Sources the file does not name are unaffected |
| `default` | **Required.** `allow` or `deny`: what a caller **no rule names** gets on this source. `allow` = today's behaviour, unrestricted |
| `rules[].name` | Shown in the audit log. Unique per source |
| `rules[].applies_to` | `roles: [read, readwrite, admin]` and/or `keys: [key names]`. A key-name match beats a role match. A role or key in two rules of one source is refused at load |
| `rules[].tables.<table>` | The tables this rule's callers may query. **Any other table, view or copy on the source is refused for them** |
| `columns` | The visible columns. A column added to the database later is **hidden** until you list it. Omit (with `hidden`) for all columns |
| `hidden` | Instead of `columns`: the withheld columns. A column added later is **visible** |
| `mask` | `{column: redact \| partial \| hash}` — see below |
| `rows` | A row filter, written in the source's SQL dialect. `:name` is a caller attribute |

**Key names, not IDs.** Rotation issues a new key ID and keeps the name (and the key's
attributes), so a rule follows a key through rotation.

### Masks

| Mask | Returns | Example |
|---|---|---|
| `redact` | `[redacted]` | `[redacted]` |
| `partial` | `***` and the last 4 characters | `***6789` |
| `hash` | Hex SHA-256 of the value | `01a54629efb9…` (for `123-45-6789`) |

!!! warning "`hash` is not anonymisation"
    The hash is unkeyed. A value with few possibilities (an SSN, a date of birth, a
    postcode) can be recovered by hashing every possible value and comparing. Use
    `hash` to join or count distinct values without showing them, not to hide them.

### Row filters and caller attributes

`rows: "region = :region"` is evaluated per caller. Each `:name` is bound, as an escaped
literal, to the caller's attribute of that name:

- **API keys:** `PUT /v1/auth/keys/{id}/attributes` (admin) with
  `{"attributes": {"region": "EMEA"}}`. Up to 20 attributes; names start with a letter
  or `_` (letters, digits, `_`, at most 64 characters); values are strings of up to 200
  characters. `{}` clears them. Shown on
  `GET /v1/auth/keys`.
- **JWTs:** an `attributes` claim, an object of string values:
  `"attributes": {"region": "EMEA"}`.

**A caller whose rule needs an attribute it does not have is refused**
(`policy_missing_attribute`), never served unfiltered.

### Who a rule applies to

TDB finds the caller's rule on that source by key name first, then by role:

| Caller | Matched by |
|---|---|
| DB-managed API key | its name, then its role |
| Static key (`TDB_API_KEYS`) | role `admin` |
| JWT | its `role` claim (default `read`) |

No rule → the source's `default`. So `default: deny` with rules only for `read` refuses
`admin` keys too: give admins a rule (`tables: {t: {}}` is unrestricted access to `t`) or
use `default: allow`.

---

## How queries are enforced

For a caller with a rule, TDB parses the SQL, replaces each table reference with the
policy subquery, and runs the result. Withheld columns are not in the subquery, so
`SELECT *` leaves them out on its own.

**It fails closed.** Anything TDB cannot fully account for is refused with an audited
reason, never passed through:

- a table the rule does not name, including a view over a protected table;
- a function that is not on TDB's allow-list for policied callers (functions that run
  SQL text or read files, for example);
- a table expression it cannot place, another schema or database, more than one
  statement, `SELECT INTO`, locking clauses;
- SQL it cannot parse.

Callers **without** a rule on the source are unaffected: no parsing, no allow-list.

Verified against real PostgreSQL, MySQL and SQL Server, and DuckDB (CSV and Parquet
sources). **Snowflake has not yet been verified against a real Snowflake account.**

### What a policied caller sees

| Surface | Behaviour |
|---|---|
| `POST /v1/query`, MCP tools, views | Rewritten as above. Views run under the caller's rule |
| `GET /v1/sources/{ref}/schema`, `schema_source` | Withheld columns and unlisted tables are left out; a masked column carries `"masked": "<mask>"` |
| SQL errors | Only the reason (`column "x" does not exist`). The engine's context lines would quote TDB's rewritten SQL |
| Registering or deleting sources | Refused for a non-admin key a policy restricts (`policy_registry_change`): a copy under a new name would carry no rule. Admin keys are exempt |

### Through the PostgreSQL wire gateway

Statements are rewritten in both protocols, including the query inside `DECLARE`.
Drivers and BI tools first read the catalog (`pg_catalog`, `information_schema`) to list
tables and columns, so a query that reads **only** catalog tables passes as written.
Catalogs that hold data rather than metadata (`pg_stats` and similar) and functions that
run SQL text are refused. Verified with psql 16, Metabase (pgjdbc) and
SQLAlchemy + psycopg 3.

!!! info "Known limit: names are visible through the gateway"
    Through the gateway a policied caller can see the **names** of withheld columns
    and of tables and views outside its rule, plus approximate table row counts, in
    the catalog. Their **values** stay protected: querying them fails. REST and MCP
    schema responses do not show them. Tell us if column names are sensitive for you.

---

## Audit

While a policy file is loaded, **every** query and denial entry carries
`policy_sha256`, the SHA-256 of the file. A query run under a rule adds:

| Field | Value |
|---|---|
| `policy_rule` | The rule's name |
| `policy_masked` | `{table: [masked columns]}` |
| `policy_withheld` | `{table: [withheld columns]}` |
| `policy_row_filtered` | `[tables a row filter applied to]` |

They are present, possibly empty, on every query under a rule. A denial under a rule
carries `policy_rule`. The denial reasons are listed under
[Errors & audit reasons](../reference/errors.md#column-and-row-policies).

---

## Checking a policy file

`GET /v1/policies` (admin) returns the loaded file's path, SHA-256, and each source's
`default` and rules. With `?validate=true` it also reads each source and reports:

- `problems`: a table or column the rules name that the source does not have, or a
  source name that is not registered;
- `warnings`: a view that reads a protected table while the source's `default` is
  `allow`, so callers with no rule read the view in full (PostgreSQL, MySQL, SQL Server).

```bash
curl -s "http://localhost:8000/v1/policies?validate=true" -H "Authorization: Bearer <ADMIN_KEY>"
```

---

## Suggested policies

`tdb policy suggest <source>` (or `GET /v1/sources/{ref}/policy-suggestions`, admin)
flags columns that look like personal data and prints a draft policy file:

```bash
TDB_URL=http://localhost:8000 TDB_API_KEYS=<ADMIN_KEY> tdb policy suggest clinic > policies.yml
```

```yaml
#   patients.ssn        government_id -> partial ('ssn')
#   patients.diagnosis  health -> hidden ('diagnosis')
version: 1
sources:
  clinic:
    default: allow
    rules:
    - name: suggested
      applies_to:
        roles: [read]
      tables:
        patients:
          columns: [id, name, region, ssn]
          mask: {ssn: partial}
```

- **Columns are judged by name and type only. No value is read**, so personal data in
  a free-text column (`notes`) is not found.
- Categories: secrets, government IDs, identity documents, financial accounts, medical
  record numbers, health data, special-category data, dates of birth, contact details,
  network addresses, postal addresses, personal names, locations, compensation.
- The draft lists every column of every table for the `read` role, so it loads as
  written. `hash` is never suggested.
- Suggestions only: nothing is enforced until you set `TDB_POLICIES_FILE`. Available on
  every commercial licence.

---

## Licence

Policies are a **Mid-Market** feature. Licences carry a `tier`; a licence issued before
0.22.0 has none and counts as Mid-Market. **An SMB licence with `TDB_POLICIES_FILE` set
stops TDB from starting**, with a message that names both ways out: unset the variable,
or ask us for a Mid-Market licence. TDB never serves a source without the rules written
for it.

---

## Limits

- A row filter is SQL in the source's dialect; TDB binds attributes but does not check
  the filter's logic. Test it with the key it is for.
- Denials raised after the rewrite (a query timeout, a privileged database role) carry
  `policy_sha256` but not `policy_rule`.
- Policies apply to reading data. Who may register sources and manage keys is
  [RBAC](rbac.md).
