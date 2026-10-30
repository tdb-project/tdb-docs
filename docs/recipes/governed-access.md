# Two keys, one table: column and row policies

*Enterprise, Mid-Market. About 20 minutes. Checked end to end against enterprise 0.22.0
and PostgreSQL 16.*

By the end, two keys query the same PostgreSQL table. One sees every column and every
row. The other gets SSNs masked, diagnoses withheld and only its own region's rows, on
REST, MCP and the PostgreSQL wire gateway alike, and the audit log names the rule that
applied. Reference: [Column and row policies](../security/policies.md).

## 1. A table and a read-only role

In `psql`, as the database owner:

```sql
CREATE TABLE patients (id int PRIMARY KEY, name text, region text, ssn text, diagnosis text);
INSERT INTO patients VALUES
  (1, 'Ann Lee',   'EMEA', '123-45-6789', 'asthma'),
  (2, 'Bo Chen',   'APAC', '987-65-4321', 'diabetes'),
  (3, 'Cy Okafor', 'EMEA', '555-12-3456', 'migraine'),
  (4, 'Di Ruiz',   'AMER', '222-33-4444', 'hypertension');

CREATE ROLE tdb_reader LOGIN PASSWORD 'choose-a-strong-password';
GRANT USAGE ON SCHEMA public TO tdb_reader;
GRANT SELECT ON patients TO tdb_reader;
```

## 2. Write the policy

The source will be registered as `clinic`. Save this as `/srv/tdb/policies.yml`:

```yaml
version: 1
sources:
  clinic:
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

Rules name keys that do not exist yet; that is fine. TDB checks the file's shape at
startup and refuses to start if it is invalid.

## 3. Start TDB with the policy and the gateway

The gateway needs a certificate its clients trust (here `/srv/tdb/certs/`, valid for
`tdb.internal`):

```bash
docker run -d --name tdb -p 8000:8000 -p 5432:5432 \
  -e TDB_API_KEYS=$ADMIN_KEY \
  -e TDB_POLICIES_FILE=/config/policies.yml \
  -e TDB_PG_PORT=5432 \
  -e TDB_PG_TLS_CERT=/config/certs/tdb.crt -e TDB_PG_TLS_KEY=/config/certs/tdb.key \
  -v /srv/tdb:/config:ro \
  tdb-enterprise:<version>
docker logs tdb 2>&1 | grep -E 'policies_loaded|license_ok'
```

```text
INFO:     policies_loaded sources=1 rules=2 sha256=0b55e457d539 from=/config/policies.yml
INFO:     license_ok customer=… edition=… tier=midmarket expires=… days_left=…
```

`tier=smb` with a policy file set would stop here with a message instead: policies are
Mid-Market.

## 4. Register the source and create the keys

```bash
export TDB=http://localhost:8000
api() { curl -s -H "Authorization: Bearer $1" -H "Content-Type: application/json" "${@:2}"; }

api $ADMIN_KEY $TDB/v1/sources -d '{"name":"clinic","source_type":"postgres",
  "connection":{"host":"db.internal","port":5432,"dbname":"clinic",
                "user":"tdb_reader","password":"choose-a-strong-password"}}'

api $ADMIN_KEY $TDB/v1/auth/keys -d '{"name":"support-emea","role":"read"}'   # note key.id and raw_key
api $ADMIN_KEY $TDB/v1/auth/keys -d '{"name":"clinician","role":"read"}'

export SUPPORT=...      # support-emea's raw_key
export CLINICIAN=...    # clinician's raw_key
api $ADMIN_KEY -X PUT $TDB/v1/auth/keys/<support-emea key.id>/attributes \
  -d '{"attributes":{"region":"EMEA"}}'
```

## 5. Query as each key

```bash
Q='{"source_id":"clinic","sql":"SELECT * FROM patients ORDER BY id"}'
api $CLINICIAN $TDB/v1/query -d "$Q" | jq -c '.rows[]'
api $SUPPORT   $TDB/v1/query -d "$Q" | jq -c '.rows[]'
```

```json
{"id":1,"name":"Ann Lee","region":"EMEA","ssn":"123-45-6789","diagnosis":"asthma"}
{"id":2,"name":"Bo Chen","region":"APAC","ssn":"987-65-4321","diagnosis":"diabetes"}
{"id":3,"name":"Cy Okafor","region":"EMEA","ssn":"555-12-3456","diagnosis":"migraine"}
{"id":4,"name":"Di Ruiz","region":"AMER","ssn":"222-33-4444","diagnosis":"hypertension"}

{"id":1,"name":"Ann Lee","region":"EMEA","ssn":"***6789"}
{"id":3,"name":"Cy Okafor","region":"EMEA","ssn":"***3456"}
```

Try to get around it as `support-emea`:

```bash
api $SUPPORT $TDB/v1/query -d '{"source_id":"clinic","sql":"SELECT count(*) AS n FROM patients"}' | jq -c .rows
# [{"n":2}]
api $SUPPORT $TDB/v1/query -d '{"source_id":"clinic","sql":"SELECT diagnosis FROM patients"}'
# {"detail":"Query failed: column \"diagnosis\" does not exist"}
```

## 6. The same key over MCP and the wire gateway

```bash
api $SUPPORT $TDB/v1/mcp -H 'MCP-Protocol-Version: 2025-11-25' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"query_source",
       "arguments":{"source_name":"clinic","sql":"SELECT * FROM patients ORDER BY id"}}}' \
  | jq -c '.result.structuredContent.rows[]'
# {"id":1,"name":"Ann Lee","region":"EMEA","ssn":"***6789"}
# {"id":3,"name":"Cy Okafor","region":"EMEA","ssn":"***3456"}

PGPASSWORD=$SUPPORT psql "host=tdb.internal port=5432 dbname=clinic user=support-emea \
  sslmode=verify-full sslrootcert=tdb-ca.pem" -c 'SELECT * FROM patients ORDER BY id'
```

```text
 id |   name    | region |   ssn
----+-----------+--------+---------
  1 | Ann Lee   | EMEA   | ***6789
  3 | Cy Okafor | EMEA   | ***3456
(2 rows)
```

## 7. Read the audit log

```bash
docker exec tdb sh -c 'grep "\"event\": *\"query\"" /app/tdb_audit.jsonl | tail -1' \
  | jq -c '{policy_rule, policy_masked, policy_withheld, policy_row_filtered}'
```

```json
{"policy_rule":"emea-support","policy_masked":{"patients":["ssn"]},"policy_withheld":{"patients":["diagnosis"]},"policy_row_filtered":["patients"]}
```

A key no rule names is refused, and the refusal is audited too:

```bash
INTERN=$(api $ADMIN_KEY $TDB/v1/auth/keys -d '{"name":"intern","role":"read"}' | jq -r .raw_key)
api $INTERN $TDB/v1/query -d "$Q"
# {"detail":"Refused by policy: no policy rule covers this key on this source"}
# audit: {"event":"denied","action":"query","reason":"policy_no_rule",...}
```

## Next

- Run `tdb policy suggest clinic` for a draft policy covering every column that looks
  like personal data. See [Suggested policies](../security/policies.md#suggested-policies).
- `GET /v1/policies?validate=true` checks the file against the live schema.
