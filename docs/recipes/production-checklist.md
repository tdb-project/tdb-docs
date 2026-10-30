# Production checklist

Before TDB serves real data to anyone but you. Each item links to where it is explained.

## Network

- [ ] TDB is reachable only through a TLS proxy, and its own port is not published —
      [HTTPS with Caddy](https-caddy.md)
- [ ] `TDB_SERVER_URL` is the public `https://` address (enterprise OAuth needs it)
- [ ] `TDB_CORS_ORIGINS` lists specific origins, not `*`, if browsers call TDB —
      [Environment variables](../reference/environment-variables.md)

## Keys and identities

- [ ] `TDB_API_KEYS` holds a strong secret, not `dev-insecure-key-change-me`
- [ ] Agents and integrations use DB-managed keys with the `read` role and an expiry
      (enterprise) — [API keys](../auth/api-keys.md)
- [ ] Each agent's key is limited to the tools it needs with `allowed_tools` —
      [RBAC](../security/rbac.md#restricting-tool-access)
- [ ] `TDB_JWT_SECRET` (if JWTs are used) is at least 32 random bytes

## Sources

- [ ] Every database source uses a least-privileged, `SELECT`-only role, and
      `TDB_ALLOW_PRIVILEGED_DB_ROLE` is unset —
      [PostgreSQL](../connectors/postgresql.md#use-a-least-privileged-role)
- [ ] Databases reached over an untrusted network use `"sslmode": "verify-full"`
- [ ] CSV, Parquet and DuckDB files sit under `TDB_ALLOWED_DATA_DIR`

## Data at rest

- [ ] The registry (`TDB_REGISTRY_DB`) and audit log (`TDB_LOG_FILE`) are on a persistent,
      backed-up volume
- [ ] `TDB_ENCRYPTION_KEY` is set from a secret store, so the key that protects source
      passwords does not sit beside them (enterprise) —
      [Credentials at rest](../security/credentials.md)

## Audit

- [ ] The audit log is exported off the host to Splunk or S3 on a schedule (enterprise) —
      [Audit log](../security/audit.md)
- [ ] Someone checks `GET /v1/audit/verify` periodically, and rotation is configured if the
      log will grow large
- [ ] You know what each refusal means: [Errors & audit reasons](../reference/errors.md)

## Operations

- [ ] `/health` is wired to your orchestrator, and `/metrics` to Prometheus (enterprise) —
      [Prometheus metrics](../observability/metrics.md)
- [ ] `TDB_QUERY_TIMEOUT` and `TDB_MAX_ROWS` suit your sources (enterprise)
- [ ] Someone reads the [release notes](../release-notes.md) and
      [security updates](../security/updates.md) before each upgrade
