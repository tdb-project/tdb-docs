# Release notes

What changed in each release, and **whether you have to do anything when you upgrade**.
Newest first. Both editions are released together when a change touches both; a dash
means that edition had no release for the change.

For security fixes in full — what was exposed, who was affected, how to check — see
[Security & updates](security/updates.md). The community edition's full changelog is
[CHANGELOG.md](https://github.com/tdb-project/tdb-community/blob/main/CHANGELOG.md).
To see what you are running: `GET /v1/version`.

!!! tip "Reading the version numbers"
    A **patch** (0.20.**3**) fixes behaviour without changing what a correct caller
    sees. A **minor** (0.**21**.0) adds something, or changes a response or status code
    a caller could notice — even when the change is a fix. Read the action column for
    every minor.

## October 2026

| Date | Enterprise | Community | What changed | Action on upgrade |
|---|---|---|---|---|
| 10-29 | 0.21.1 | — | Licence check hardening: the licence verification key can no longer be changed through configuration. | None. |
| 10-28 | **0.21.0** | **0.13.0** | **Lean Mode**: smaller MCP results for agents — rows as arrays, marked cell trimming, and in enterprise schema detail levels and a per-key setting. [Saving tokens](api/mcp.md#saving-tokens-lean-mode) | None. Opt-in; with nothing set, responses are unchanged. |
| 10-27 | **0.20.3** | 0.12.4 | **Security (enterprise):** API keys were written to the application log; now a six-character hint. Both: a mistake in the caller's SQL is a 400, not a 500. | **Enterprise: rotate API keys and purge retained application logs.** [Advisory](security/updates.md) |
| 10-27 | 0.20.2 | 0.12.3 | Line breaks in logged values are escaped, so caller input can't forge a log line. | None. |
| 10-26 | 0.20.1 | 0.12.2 | Every query value is plain JSON, the same on REST and MCP: binary as base64, NaN/Infinity as strings. REST used to fail on binary values. | Clients that parsed binary or NaN output get base64 / strings now. |
| 10-26 | **0.20.0** | — | **Parquet connector**: one Parquet file served as the table `data`. [Parquet](connectors/parquet.md) | None. |
| 10-26 | 0.19.1 | 0.12.1 | **Security:** `certifi` past GHSA-6q48-6c7w-9gw2. Enterprise was exposed on Splunk, S3 and Snowflake connections. | **Enterprise: upgrade.** Community was not exposed. |
| 10-16 | **0.19.0** | — | `tools/list` shows a scoped key only its `allowed_tools`; an unknown tool name in `PUT /v1/auth/keys/{id}/tools` is a 400. | Scripts that set `allowed_tools` with a misspelt name now fail instead of storing it. |
| 10-10 | **0.18.0** | **0.12.0** | MCP `2026-07-28` alongside the earlier revisions; audit entries name the AI client (`mcp_client`, `mcp_protocol`); enterprise OAuth `iss` and opt-in client metadata documents. | None. |
| 10-10 | **0.17.0** | **0.11.0** | MCP `2025-11-25` and `2025-06-18`: version negotiation, tool titles, read-only hints, structured output. Enterprise output filter fails closed. | A client sending an unsupported `MCP-Protocol-Version` header gets 400. |
| 10-10 | **0.16.0** | — | `TDB_MAX_ROWS` bounds every path that returns rows (MCP tools, view runs). | A `limit` above the ceiling is now refused instead of clamped. |
| 10-10 | 0.15.1 | — | MCP tools and the REST view run report `truncated`. | None. |
| 10-10 | **0.15.0** | **0.10.0** | MCP conformance: no 500 on malformed input, notifications get 202, `ping`, enterprise argument type checks. | A request from a foreign browser `Origin` is 403; list browser origins in `TDB_CORS_ORIGINS` (enterprise). |
| 10-10 | 0.14.2 | 0.9.2 | The row cap reaches the source when a query ends in a comment. | None. |
| 10-09 | 0.14.1 | 0.9.1 | pip removed from the images (six scanner findings, not exposed). Gateway refusals show TDB's error over the extended protocol. | None. |
| 10-09 | **0.14.0** | **0.9.0** | **PostgreSQL wire gateway** (enterprise, opt-in `TDB_PG_PORT`). Write keywords used as column aliases are accepted. [Wire gateway](integrations/postgres-wire.md) | None; the gateway is off until `TDB_PG_PORT` is set. |
| 10-09 | **0.13.0** | **0.8.0** | The `replace()` string function is accepted. | None. |
| 10-09 | **0.12.0** | 0.7.2 | **Security:** the read-only check could be shown one statement while the engine ran several (GHSA-qmj5-8fv9-h74r). Enterprise also: `allowed_tools` enforced on REST, privileged database roles refused, Postgres SSL and `require_auth` options. | **Upgrade.** Sources that connect as a privileged role are refused unless `TDB_ALLOW_PRIVILEGED_DB_ROLE=true`. [Advisory](security/updates.md) |

## Earlier releases

Earlier security releases, with their advisories, are listed in
[Security & updates](security/updates.md). The community edition's earlier releases are
in its [CHANGELOG](https://github.com/tdb-project/tdb-community/blob/main/CHANGELOG.md).
