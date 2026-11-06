# Query Approvals

**Mid-Market, from enterprise 0.32.0. Part of the [policy file](policies.md); off unless a
rule asks for it.**

Some data should not be read without a person saying yes first. An approval rule makes a
query that reads named tables or columns **wait**: TDB records a request, an approver
decides, and only then does the query run, for the person who asked. The audit log records
the request, who decided, and every run under the approval.

Nothing waits on a connection while a person decides. The caller gets an answer at once,
with a request id, and runs the query again (or runs the request by its id) once it is
approved.

---

## What it looks like

```yaml title="policies.yml"
version: 1
sources:
  clinic:
    default: deny
    approvers: {groups: [privacy-office]}   # who may decide, on this source
    rules:
      - name: support
        applies_to: {roles: [read]}
        tables:
          patients: {columns: [id, name, ssn, city]}
          visits: {}
        approval:
          columns: [patients.ssn]           # a query reading this waits
          # tables: [visits]                # or any column of a table
```

A support agent's query for names and cities runs as before. A query that reads
`patients.ssn` (in the select list, a `WHERE`, an `ORDER BY`, a `*`, a CTE or a subquery)
gets this answer instead:

```text
Approval required: this query reads patients.ssn, which needs a person's approval before
it runs. Request apr_4f1c09d2b7e3a581 is pending. Once it is approved, send the same query
again or run the request by its id.
```

On REST that is a `403` with the text as `detail`; on MCP a tool result with `isError`; on
the [wire gateway](../integrations/postgres-wire.md) an error with SQLSTATE `42501`.

---

## Deciding

An approver lists, reads and decides requests with the CLI or REST, using **their own**
key or signed-in token:

```bash
export TDB_API_KEYS=<your key or SSO token>
tdb approvals list
tdb approvals show apr_4f1c09d2b7e3a581      # the full statement it would run
tdb approvals approve apr_4f1c09d2b7e3a581 --reason "ticket 1842"
tdb approvals deny apr_4f1c09d2b7e3a581 --reason "use the masked view"
```

| Route | What it does |
|---|---|
| `GET /v1/approvals?status=pending\|all` | Requests you made, or may decide |
| `GET /v1/approvals/{id}` | One request, with its statement |
| `POST /v1/approvals/{id}/approve` | `{"reason": "…"}`, as one of the source's approvers |
| `POST /v1/approvals/{id}/deny` | `{"reason": "…"}`; the reason is shown to the requester |
| `POST /v1/approvals/{id}/run` | `{"limit": N}`; the approved statement, by its requester |

A request is visible only to the person who made it and to that source's approvers; anyone
else gets `404`.

**Rules that always hold:**

- The person who asked can never decide their own request, even if they are an approver.
- A static key from `TDB_API_KEYS` names no one: it can neither make a request nor decide
  one. Use a [database key](../auth/api-keys.md) or [SSO](../auth/sso.md).
- An approval is for **one statement, one person, one source**. Sending it again with other
  spacing or keyword case matches; a changed column, filter, `LIMIT`, alias or comment is a
  new request, because the approver judged what was written.
- It can be run for `TDB_APPROVAL_TTL` (1 hour) after the decision, by the requester only,
  against live data, under the policy in force then. TDB never runs a query ahead or keeps
  results.
- A pending request expires after `TDB_APPROVAL_PENDING_TTL` (24 hours). Changing the policy
  file voids every open request, since its approvers judged other rules.
- A requester may have at most `TDB_APPROVAL_MAX_PENDING` (20) requests waiting.
- A key is identified by its name **and** id, so rotating a key starts afresh.

---

## Agents

An agent that meets `approval_required` should tell its user and stop. In TDB's tests with
Claude (Opus, Sonnet and Haiku), every agent did so and named the request id. In the same
conversation, every agent sent the identical query again once told it was approved.

A **new** conversation does not reliably reproduce the exact SQL. For that case, callers a
rule can hold see one more MCP tool, `run_approved`:

```json
{"name": "run_approved", "arguments": {"approval_id": "apr_4f1c09d2b7e3a581"}}
```

It runs the stored statement, so the agent cannot drift from what was approved. Callers
no approval rule applies to never see it, so it costs them nothing; for a caller that does
see it, it adds 109 tokens to each request (measured with Claude Code). A key whose
`allowed_tools` names any query tool may use it.

---

## Announcing requests

Set a webhook to tell approvers that a request is waiting. Decisions are always made in TDB.

| Variable | Default | |
|---|---|---|
| `TDB_APPROVAL_WEBHOOK_URL` | unset (off) | https, or http to `localhost` only |
| `TDB_APPROVAL_WEBHOOK_SECRET` | | Required with the URL; at least 32 characters |
| `TDB_APPROVAL_WEBHOOK_SQL` | `redacted` | `full` sends literals too |

A URL that is set without a usable secret stops TDB at startup.

TDB sends one JSON `POST` per new request (a retry by the caller is not a new request),
with three tries and no redirects:

```json
{
  "event": "approval_requested",
  "approval_id": "apr_4f1c09d2b7e3a581",
  "source": "clinic",
  "requester": "user:7c2e…",
  "rule": "support",
  "reads": ["patients.ssn"],
  "sql": "SELECT name, ssn FROM patients WHERE name = ?",
  "sql_redacted": true,
  "created_at": "2026-11-06T10:14:03Z",
  "expires_at": "2026-11-07T10:14:03Z",
  "text": "TDB approval request apr_4f1c…: user:7c2e… wants to read patients.ssn on clinic. …"
}
```

Verify it with the shared secret before trusting it:

```python
import hashlib, hmac

def verify(body: bytes, timestamp: str, signature: str, secret: str) -> bool:
    mac = hmac.new(secret.encode(), timestamp.encode() + b"." + body, hashlib.sha256)
    return hmac.compare_digest("sha256=" + mac.hexdigest(), signature)
```

The headers are `X-TDB-Timestamp` and `X-TDB-Signature`. Reject old timestamps to refuse
replays. The `text` field is meant for chat incoming webhooks; TDB has not been tested
against Slack or Teams themselves.

---

## Audit

| Entry | When |
|---|---|
| `approval_requested` | A request is made: `approval_id`, `requester`, `approval_reads`, `policy_rule` |
| `approval_decided` | An approver decides: `decision`, `approver`, `approval_reason`, `expires_at` |
| `denied`, reason `approval_pending` / `approval_denied` | A query waiting or refused; carries `approval_id` |
| `denied`, reason `approval_unresolved` | TDB could not tell which columns the query reads; it is refused, never run |
| `denied`, reason `approval_requires_identity` / `approval_limit` | A static key, or too many waiting |
| `query` with `approval_id` | Every run under an approval |

See [Errors & audit reasons](../reference/errors.md) for every reason.
