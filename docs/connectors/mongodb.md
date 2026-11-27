# MongoDB Connector

**Enterprise, from 0.40.0.** Read-only SQL over MongoDB: one collection, or a whole
database with each collection as a table. It works through REST, MCP, views, paging and
streams, under the same row cap, timeout, audit, approvals and policies as every other
source.

!!! info "What it has run against"
    MongoDB **8**, self-hosted, in TDB's CI on every change, including TLS with a private
    certificate authority. **MongoDB Atlas** has been run once, on a free M0 cluster
    (MongoDB 8.0, `mongodb+srv` with TLS): the read-only user check, a query with exact
    decimals, timestamps and nested fields, the size cap, refusal of a user that can write,
    and a wrong password. Atlas is not part of CI.

---

## How it works, and what that means for you

MongoDB has no SQL. For each query, TDB reads the collections the query names with
`find()`, copies them into its own locked, in-memory query engine, and runs your SQL
there. Your SQL never reaches MongoDB: TDB sends it only `find()`, counts and collection
listing, so there is nothing to inject into.

This shapes three things:

- **Collection size has a limit.** A collection with more than
  `TDB_MONGO_MAX_DOCUMENTS` documents (default **100,000**), or a query that would copy
  more than `TDB_MONGO_MAX_BYTES` of data (default **256 MB**), is **refused** with
  `400 collection_too_large`. TDB never answers from part of a collection: a count or a
  sum over a partial copy would look complete and be wrong. To serve a larger collection,
  register the source with a [`filter`](#narrowing-a-large-collection) that narrows it.
- **Every query reads current data**, at the cost of the copy. Measured on MongoDB 8
  with documents of about 1 KB: 10,000 documents in about **0.4 s**, 100,000 in about
  **3 s**, plus the query itself.
- **Indexes are not used.** A `WHERE` runs on the copy, not in MongoDB.

For collections far beyond the cap, query MongoDB's own analytics tooling, or export to
a warehouse TDB connects to.

## 1. Create a user that only reads

TDB refuses a MongoDB user that can do more than read (see
[the privilege check](#the-privilege-check)). The built-in `read` role on the database is
what TDB needs:

```javascript
use shop
db.createUser({ user: "tdb_reader", pwd: "s3cret", roles: [{ role: "read", db: "shop" }] })
```

A custom role granting only `find` and `listCollections` on chosen collections is fine
too. `readAnyDatabase` passes the check: TDB reads only the source's own database, so it
gives callers nothing more.

## 2. Register it

```bash
curl -X POST http://localhost:8000/v1/sources \
  -H "Authorization: Bearer <ADMIN_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "shop",
    "source_type": "mongodb",
    "connection": {
      "host": "mongo.internal",
      "port": 27017,
      "database": "shop",
      "username": "tdb_reader",
      "password": "s3cret"
    }
  }'
```

| Key | Required | Notes |
|---|---|---|
| `host` | Yes | A host name or address, or a comma-separated seed list (`a:27017,b:27017`). **Not a URI**: options in a URI would change how TDB connects |
| `database` | Yes | The database the source reads |
| `port` | No | Default `27017`; ignored with `srv` |
| `collection` | No | Omit to register the whole database, each collection a table |
| `username`, `password` | No | The read-only user; omit only for a server without access control |
| `auth_source` | No | Where the user is defined; default the source's `database` |
| `srv` | No | `true` for `mongodb+srv://` (Atlas); `host` is then one DNS name. Turns TLS on |
| `tls` | No | Connect over TLS |
| `ca_cert` | No | Path inside the container to a CA certificate, for a private certificate authority |
| `replica_set` | No | The replica set name |
| `read_preference` | No | Default **`secondaryPreferred`**, so agent reads stay off the primary when a secondary is up. Also `primary`, `primaryPreferred`, `secondary`, `nearest` |
| `filter` | No | A MongoDB filter applied to every collection the source reads ([below](#narrowing-a-large-collection)) |

## Querying

Each collection is a table, named as in MongoDB:

```sql
SELECT customer, amount FROM orders WHERE status = 'paid' ORDER BY amount DESC
SELECT o.customer, count(*) FROM orders o JOIN refunds r ON o._id = r.order_id GROUP BY 1
SELECT address->>'$.city' AS city, count(*) FROM customers GROUP BY 1
```

Name a collection without a database prefix. System collections (`system.*`) and table
functions are refused (`400 sql_validation_failed`). MongoDB views are listed and read
like collections.

## Types

Each top-level field is a column. Its type is decided from every value it holds:

| In MongoDB | In TDB |
|---|---|
| Integers (32 and 64-bit) | Integer |
| Doubles, or integers mixed with doubles | Double |
| Decimal128 | Exact decimal, returned as a string (`"12.50"`), as on other connectors |
| Booleans | Boolean |
| Dates | Timestamp in UTC |
| ObjectId, UUID | Text |
| Binary | Base64 text |
| Embedded documents and arrays | JSON values; query into them with `->>` and `json_extract` |
| A field whose type differs between documents | Text, each value as written |
| A field a document lacks | `NULL` |

`GET /v1/sources/{ref}/schema` reads the types from a sample of up to 1,000 documents per
collection, so a field that appears only later in a collection may be missing there; a
query still reads every document.

## Narrowing a large collection

The `filter` key narrows every collection the source reads, before the copy, so a large
collection can be served in part, on purpose, under a name that says so:

```json
"connection": {
  "host": "mongo.internal", "database": "shop", "collection": "orders",
  "username": "tdb_reader", "password": "s3cret",
  "filter": { "created_at_year": { "$gte": 2026 }, "status": { "$in": ["paid", "shipped"] } }
}
```

Only the person who registers the source sets it; callers cannot. It may use `$eq`,
`$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`, `$and`, `$or`, `$nor`, `$not`,
`$exists`, `$type`, `$all`, `$size` and `$elemMatch`. Operators that run code or can run
for a long time (`$where`, `$function`, `$accumulator`, `$expr`, `$regex`) are refused at
registration.

## The privilege check

On every connection TDB reads the user's effective privileges (`connectionStatus`, with
inherited roles included) and refuses the source (`403 privileged_db_role`) if they
include anything beyond reading: `insert`, `update`, `remove`, `createIndex`, any
administrative action. A role that inherits a write from another role is caught too.

TDB itself never writes to MongoDB whatever the user may do; the check is so that a
credential TDB holds cannot be used to change your data if it ever leaks.
`TDB_ALLOW_PRIVILEGED_DB_ROLE=true` turns the check off.

On a server with access control off, there are no privileges to read and the check
passes. TDB still only reads, but enable access control.

## Timeouts

`TDB_QUERY_TIMEOUT` (default 30 s) covers the copy and the query on it, as one limit: TDB
keeps its own deadline while reading and closes the cursor when it passes, which stops the
read on the server, and from 0.40.1 it interrupts the query on the copy when the rest runs
out (`504 query_timeout`). MongoDB's `maxTimeMS` alone does not bound a read, because it
counts only server time per batch. Before 0.40.1 the query on the copy had no limit.

## Policies

Column and row policies apply as on other sources: a rule names collections as tables and
fields as columns, and masking, withholding and row filters are applied to the SQL before
it runs on the copy.

## Not supported

- Collections over the caps (refused; see above).
- Index use, and pushing a `WHERE` down to MongoDB.
- The PostgreSQL wire gateway (PostgreSQL sources only).
- Change streams, and writes of any kind.
