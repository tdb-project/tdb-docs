# Tenants

**Enterprise, from enterprise 0.33.0. Off unless `TDB_TENANTS_DIR` is set.**

One TDB deployment can serve several teams or customers, each as a **tenant** that sees only
its own sources, keys, views, policies, approvals and audit log. Two tenants can both have a
source called `sales` and never meet: neither can name, query or audit the other's.

Each tenant's state lives in its own directory, with its own registry and its own
hash-chained audit log. A request's tenant comes only from its credential, never from a
header or a URL it could choose.

---

## How it is laid out

```text
TDB_TENANTS_DIR/
├── acme/
│   ├── tenant.yaml          # id: '1a2b3c4d'   (suspended: true, to suspend)
│   ├── registry.db          # its sources, keys, annotations, approvals
│   ├── audit.jsonl          # its own hash chain, with sealed segments beside it
│   ├── data/                # where its CSV, Parquet and DuckDB files go
│   ├── policies.yml         # optional: its column and row policies
│   └── views/               # optional: its named views
├── beta/ …
└── .deployment/
    └── audit.jsonl          # the deployment's own chain (below)
```

**`default`** is the deployment as it was before tenants. Its state stays where the `TDB_*`
variables put it (`TDB_REGISTRY_DB`, `TDB_LOG_FILE`, `TDB_ALLOWED_DATA_DIR`,
`TDB_POLICIES_FILE`, `TDB_VIEWS_DIR`). Turning tenants on moves nothing.

What belongs to the **deployment**, not to a tenant: the people who sign in through your
identity provider, SCIM, the OAuth sign-in state, the encryption key for stored credentials,
the licence, the PostgreSQL wire gateway's port, and the webhook and SIEM destinations.

---

## Who acts in which tenant

| Credential | Acts in |
|---|---|
| A database key minted in a tenant: `tdbt_<tenant id>_…` | That tenant |
| A database key from before tenants: `tdbk_…` | `default` |
| A person signed in by single sign-on | The tenant their groups name (below) |
| A static `TDB_API_KEYS` key, or the `TDB_ADMIN_USER` login | **No tenant**: the operator (below) |

A key edited to name another tenant is refused with 401: the tenant it names must hold it.

!!! warning "Static keys read no tenant, `default` included"
    With tenants on, `TDB_API_KEYS` keys are the operator's. Scripts that query with one stop
    working. **Before you set `TDB_TENANTS_DIR`, mint a database key for `default`** and move
    them to it: `POST /v1/auth/keys` today, or `POST /v1/tenants/default/keys` after. TDB logs
    `tenant_default_has_no_key` at startup while `default` has none.

---

## Turning tenants on

1. Have an **Enterprise** licence. With any lower tier, TDB refuses to start while
   `TDB_TENANTS_DIR` is set (`Tenants are an Enterprise feature`).
2. Mint a key for `default` (see the warning above).
3. Give `default` a data directory **outside** `TDB_TENANTS_DIR`. TDB refuses to start if
   `TDB_ALLOWED_DATA_DIR` is inside it.
4. If you use single sign-on, add `tenant:` to every group in `TDB_SSO_FILE` (below).
5. Create an empty directory, set `TDB_TENANTS_DIR` to it, and restart.

```bash
docker run … -e TDB_TENANTS_DIR=/tenants -v /srv/tdb/tenants:/tenants …
```

---

## Creating a tenant

Run the CLI inside the container, then restart TDB:

```bash
docker exec tdb python -m tdb.cli tenants init acme
# Created tenant acme (id 1a2b3c4d).
# Its admin key, shown once; store it now:
# tdbt_1a2b3c4d_…
# Restart TDB to serve the new tenant.
docker restart tdb
```

`init` creates the directory with a fresh id and one **admin** key for the tenant, shown once.
That admin then works as admins do today, inside the tenant: registers sources, mints keys,
writes its policy file and views.

A tenant name is 1–32 characters of `a-z`, `0-9` and `-`, starting with a letter or digit.
Never create a tenant by copying another's directory: TDB refuses to start when two tenants
share an id or a source id.

### Suspending a tenant

```bash
docker exec tdb python -m tdb.cli tenants suspend acme   # then restart
docker exec tdb python -m tdb.cli tenants resume acme    # then restart
```

A suspended tenant's keys and people are refused with 403. Its files stay where they are:
TDB never deletes a tenant, its keys or its audit log.

---

## The operator

With tenants on, the static `TDB_API_KEYS` keys and the `TDB_ADMIN_USER` login manage
tenants and read none. Everything the operator does that touches a tenant is on record.

| Request | What it does |
|---|---|
| `GET /v1/tenants` | Lists the tenants: name, id, suspended |
| `POST /v1/tenants/{name}/keys` | Mints a key inside a tenant (`role`, default `read`), recorded in that tenant's chain and in the deployment's |
| `GET /v1/audit/verify?tenant=acme` | Verifies acme's chain; `?tenant=_deployment` the deployment's own |
| `GET /v1/audit/history?tenant=…`, `POST /v1/audit/rotate?tenant=…` | The same for history and rotation |
| `POST /v1/audit/export?tenant=…`, `GET /v1/audit/export/status?tenant=…` | Exports one tenant's chain to Splunk or S3 |

The operator must name the chain; without `?tenant=` these return 400. A tenant's admin
calls them without `?tenant=` and acts on its own chain; naming another is a 403.

`tdb tenants list` lists tenants through the server, with the operator's key in
`TDB_API_KEYS`.

---

## Single sign-on

People belong to the deployment; their **groups** place them in a tenant. With tenants on,
every group in `TDB_SSO_FILE` names one:

```yaml title="sso.yml"
groups:
  acme-analysts: {role: read, tenant: acme}
  acme-admins:   {role: admin, tenant: acme}
  beta-analysts: {role: read, tenant: beta}
```

TDB reads a person's groups on every request, so a change at sign-in or by SCIM moves them on
their next one, with the same token. Someone whose groups name **two** tenants is refused
(`sso_tenant_ambiguous`), and so is someone whose groups name none (`sso_no_tenant`). TDB
refuses to start if a group names no tenant, or one that is not in `TDB_TENANTS_DIR`.

See [Single sign-on and SCIM](../auth/sso.md) for the rest of the group file.

---

## The audit logs

Each tenant has its own chain, verified on its own, and every entry names it (`"tenant":
"acme"`). A tenant's admin verifies, rotates and exports its own chain exactly as before.

The **deployment's chain**, `TDB_TENANTS_DIR/.deployment/audit.jsonl`, records what belongs to
no tenant:

| Entry | When |
|---|---|
| `denied`, `action: tenant` | A request refused before it had a tenant: `invalid_api_key` (a key naming no tenant that holds it), `tenant_suspended`, `sso_tenant_ambiguous`, `sso_no_tenant`, `sso_user_inactive` |
| `denied`, `operator_names_no_tenant` | The operator called a tenant route |
| `operator_key_minted`, `operator_audit_rotated`, `operator_audit_export` | The operator acted on a tenant |
| `tenants_loaded` | Each start, with the tenants it serves, so creating or suspending one is on record |

A key forged to name a real tenant is refused by that tenant, so the attempt is in that
tenant's chain.

**S3 export:** with tenants on, each chain's objects go under `<prefix>/<tenant>/`
(`<prefix>/_deployment/` for the deployment's), because every chain numbers its entries from 1.

---

## Files

A tenant's file sources must be inside its own `data/` directory. A file under
`TDB_TENANTS_DIR` can be read only by the tenant whose directory holds it, whatever path or
symlink points at it, `default` included.

**Database sources are not separated by TDB.** Each tenant registers its own sources with its
own database credentials. Give each tenant a database user that can read only that tenant's
data: TDB keeps tenants from naming each other's sources, but a tenant that is given another
tenant's database password can register it.

---

## The PostgreSQL wire gateway

One port serves every tenant. A session's tenant comes from its password, the same as REST:
a tenant key, or a signed-in person's `/sso/token`. The operator cannot log in. TDB checks
before every statement that the credential still acts in the session's tenant, so a person
moved to another tenant loses the session.

---

## Limits

- Creating, suspending and resuming a tenant take effect on restart.
- One identity provider for the deployment; each group places people in one tenant.
- `/metrics` is the deployment's: it is not split by tenant.
- Deleting a tenant is done by hand, by moving its directory away while TDB is stopped. TDB
  never deletes an audit log.
